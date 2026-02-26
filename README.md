# Laminar Compile-Time Benchmark: v17 vs v18

Measures the compile-time regression introduced by Laminar 18 compared to Laminar 17, using **identical user code** compiled against different library versions.

## Background

After upgrading from Laminar 17 to 18, projects report significantly slower compilation — the typer phase dominates at ~80% of total compile time. The user-facing API is nearly identical, but the internal changes in Laminar/Airstream 18 increase compiler workload:

- `combineWith` overloads expanded from 8 to 21 arities
- Tuple arities expanded from 8 to 22 (tuplez "light" → "full")
- F-bounded self-types added to all `Key` types
- `StyleProp` now extends 4 traits (was fewer)
- SVG attributes doubled

This benchmark quantifies the regression by compiling the same source files against both library versions.

## Results

Initial clean-compile timings (Scala 3.3.7, Scala.js 1.20.2, macOS Apple Silicon):

| Project | Files | LOC | v17 | v18 | Ratio |
|---------|-------|-----|-----|-----|-------|
| bench | 51 | 8,593 | ~10s | ~38s | **3.8x** |
| micro | 4 | 384 | ~1s | ~4s | **4.0x** |

## Prerequisites

- **JDK 11+** (tested with JDK 21)
- **sbt** (installed via [sdkman](https://sdkman.io/), [brew](https://brew.sh/), or [sbt download page](https://www.scala-sbt.org/download))
- **Node.js** (required by Scala.js)

## Quick Start

```bash
# Clone
git clone https://github.com/nguyenyou/laminar-compile-bench.git
cd laminar-compile-bench

# Verify both versions compile
sbt bench-v17/compile bench-v18/compile

# Run the full benchmark (5 iterations + profiling)
./run-bench.sh

# Analyze phase timings
./analyze.sh
```

## Project Structure

```
├── build.sbt                          # Multi-project sbt build
├── project/
│   ├── build.properties               # sbt 1.10.7
│   └── plugins.sbt                    # sbt-scalajs 1.20.2
│
├── bench-v17/src/main/scala/bench/    # 51 files — Laminar 17.2.0
├── bench-v18/src/main/scala/bench/    # 51 files — Laminar 18.0.0-M2 (identical source)
│   ├── Model.scala                    # Sealed trait with 15 case classes
│   ├── HeavyCombine_01..10.scala      # combineWith chains, arities 2-5
│   ├── ReactiveDSL_01..10.scala       # Dense element trees, reactive bindings
│   ├── SplitPattern_01..10.scala      # signal.split on 15-subtype hierarchy
│   ├── StyleHeavy_01..10.scala        # Heavy StyleProp usage (.px, .em, reactive)
│   └── MixedComponent_01..10.scala    # All patterns combined (end-to-end)
│
├── micro-v17/src/main/scala/micro/    # 4 files — Laminar 17.2.0
├── micro-v18/src/main/scala/micro/    # 4 files — Laminar 18.0.0-M2 (identical source)
│   ├── MicroCombineWith.scala         # Only combineWith chains
│   ├── MicroAttributes.scala          # Only attr/prop setters
│   ├── MicroStyleProps.scala          # Only StyleProp usage
│   └── MicroModifiers.scala           # Only modifier/implicit resolution
│
├── run-bench.sh                       # Benchmark runner (5 iterations + Vprofile)
└── analyze.sh                         # Phase timing diff parser
```

## Running the Benchmark

### Full benchmark (recommended)

```bash
./run-bench.sh
```

This will:
1. Resolve all dependencies
2. Warm up the JVM (one compile, discarded)
3. Run 5 clean compiles for each of the 4 sub-projects
4. Run a final compile with `-Vprofile` to capture phase timings
5. Print a comparison table with mean, median, stddev, min, max

You can adjust the number of iterations:

```bash
./run-bench.sh 10   # 10 iterations instead of 5
```

### Quick comparison

For a quick one-off comparison without the full runner:

```bash
# Clean compile v17
sbt "bench-v17/clean" "bench-v17/compile"

# Clean compile v18
sbt "bench-v18/clean" "bench-v18/compile"
```

### Phase profiling

To get per-phase timing breakdown:

```bash
# Compile with -Vprofile
sbt 'set bench-v17/scalacOptions += "-Vprofile"' "bench-v17/clean" "bench-v17/compile"
sbt 'set bench-v18/scalacOptions += "-Vprofile"' "bench-v18/clean" "bench-v18/compile"
```

### Analyzing results

After running the full benchmark:

```bash
./analyze.sh
```

This parses the `-Vprofile` output and produces a phase-by-phase comparison table showing where the time difference comes from (typically the `typer` phase).

## Benchmark Design

The source files simulate patterns from a real 192-file / 45K-LOC Laminar application. Pattern density per category:

| Category | Pattern | Why it stresses the compiler |
|----------|---------|------------------------------|
| HeavyCombine | 5+ `combineWith` chains, arities 2-5 | Overload resolution + `Composition` implicit search |
| ReactiveDSL | Dense element trees with `<--`, `-->`, `cls` | F-bounded Key type resolution |
| SplitPattern | `signal.split(_.id)` on 15-subtype sealed trait | Pattern matching + signal splitting |
| StyleHeavy | Heavy `StyleProp` with `.px`, `.em` | StyleProp 4-trait mixin resolution |
| MixedComponent | Realistic mix of all patterns | End-to-end cost |

## Versions

- **Scala**: 3.3.7
- **Scala.js**: 1.20.2
- **Laminar v17**: 17.2.0
- **Laminar v18**: 18.0.0-M2
- **sbt**: 1.10.7
