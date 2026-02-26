# Laminar Compile-Time Benchmark: v17 vs v18

Measures and **proves the root cause** of the compile-time regression introduced by Laminar 18 compared to Laminar 17, using identical user code compiled against different library versions.

## TL;DR

**Root cause: generated combinator arity count (22 overloads), not the tuplez library variant.**

Reducing `generateTupleCombinatorsTo` from 22 to 9 in Airstream eliminates the 3.3x compile-time regression entirely. Switching from `tuplez-full` to `tuplez-full-light` alone has zero effect.

## Context

**Issue**: After upgrading from Laminar 17 to 18, large projects report significantly slower compilation — roughly 3x slower than before.

**What was tried in the real codebase (all failed):**

| Intervention | Sites changed | Result |
|---|---|---|
| `combineWith` -> `combineWithFn` | 86 call sites | No improvement |
| Type annotations on Signal vals | All Signal vals | -1.2% (noise) |
| Pre-compute repeated `combineWith` | Heavy files | No improvement |
| Union type -> String | Model types | No improvement |

**Why per-call-site changes don't work:** The Scala 3 compiler caches implicit resolution within compilation units. The cost is in *loading and evaluating overload candidates from the dependency JAR's TASTy*, which happens once per compilation unit regardless of how many call sites exist. ~85% of typer time is untraced "self-time" — the overload candidate evaluation itself.

## Benchmark Results

### Phase 1: Quantify the regression (51 synthetic files, identical source code)

| Project | Mean | SD | vs v17 |
|---------|------|----|--------|
| bench-v17 (Laminar 17.2.0) | 11.9s | 0.1s | baseline |
| bench-v18 (Laminar 18.0.0-M2) | 39.3s | 0.1s | **3.3x** |

### Phase 2: Prove the root cause (modified Airstream experiments)

| Variant | Airstream config | Mean | vs v17 |
|---------|-----------------|------|--------|
| **v17** | Laminar 17 baseline | **11.9s** | **1.00x** |
| **v18** | tuplez-full, arity 22 (stock v18) | **39.3s** | **3.30x** |
| **v18-A** | tuplez-full-light, arity 9 | **12.3s** | **1.03x** |
| **v18-B** | tuplez-full-light, arity 22 | **39.4s** | **3.30x** |

**Interpretation:**

- **v18-A** (arity 9) = essentially identical to v17. Regression **eliminated**.
- **v18-B** (arity 22, light tuplez) = just as slow as stock v18. Tuplez variant is **irrelevant**.
- **Conclusion: the arity count (22 generated overloads) is the sole cause.**

### TASTy output sizes (identical across variants)

| Variant | TASTy total | Files |
|---------|-------------|-------|
| v17 | 798 KB | 70 |
| v18 | 812 KB | 70 |
| v18-A | 815 KB | 70 |
| v18-B | 815 KB | 70 |

TASTy sizes are nearly identical (~2% difference). The regression is purely in compile-time overload resolution, not in output size.

### Airstream JAR sizes

| Version | JAR size |
|---------|----------|
| Airstream 17.2.0 | 2,355 KB |
| Airstream 18.0.0-M1 | 2,757 KB |
| Airstream 18.0.0-bench-A (light, arity 9) | 2,647 KB |
| Airstream 18.0.0-bench-B (light, arity 22) | 3,046 KB |

The arity-22 JAR is ~400KB larger than arity-9, containing the additional generated combinator methods that the Scala 3 compiler must evaluate during overload resolution.

## Root Cause: Technical Explanation

In Laminar 17, `combineWith` supported arities 2-8 (7 overloads). In Laminar 18, Airstream expanded this to arities 2-22 (21 overloads) via code generation with `generateTupleCombinatorsTo = 22`.

When the Scala 3 compiler encounters a `combineWith` call, it must:

1. **Load** all 21 overload candidates from the Airstream JAR's TASTy
2. **Evaluate** each candidate's type parameters against the call site
3. **Rank** candidates by specificity to select the best match

This happens for every method that has generated overloads: `combineWith`, `combineWithFn`, `Signal.combine`, `EventStream.combine`, and their `Fn` variants — multiplied across 6+ generated extension classes (`CombinableSignal`, `CombinableStream`, `CombineSignalObjectOps`, etc.).

The cost is **per compilation unit** (not per call site), because the compiler caches overload resolution results within a unit. But with 51 source files, the aggregate cost is 51 x ~0.5s = ~27s of additional compile time.

**Key insight**: This is not a bug in Laminar or Airstream — it's an inherent scaling property of Scala 3's overload resolution. The compiler must evaluate all candidates for each overloaded method, and with 21 candidates per method across multiple generated classes, the work adds up.

## Recommendations

For Airstream/Laminar maintainers:

1. **Reduce the generated arity cap** from 22 to 9 (or even lower). Arities 2-9 cover 99%+ of real-world usage. This single change eliminates the entire regression.
2. **The tuplez variant (full vs full-light) does not matter** for compile time. Choose based on runtime needs.
3. **Consider a compile-time benchmark in CI** to catch future regressions early.

For users experiencing slow compilation:

- The regression is in the **library dependency**, not in your code. Per-call-site changes (type annotations, `combineWithFn`, etc.) will not help.
- Wait for an Airstream release with reduced arity cap, or use the bench-A configuration locally.

## Reproducing the Experiments

### Prerequisites

- JDK 11+ (tested with JDK 25)
- sbt 1.10.7+
- Node.js (required by Scala.js)

### Quick comparison (no modifications needed)

```bash
git clone https://github.com/nguyenyou/laminar-compile-bench.git
cd laminar-compile-bench

# Verify all variants compile
sbt bench-v17/compile bench-v18/compile bench-v18-A/compile bench-v18-B/compile

# Run the full benchmark (3 iterations per variant)
./run-experiment.sh 3
```

### Reproducing from scratch

To reproduce the modified Airstream experiments:

```bash
# Clone both repos
git clone https://github.com/raquo/Airstream.git
git clone https://github.com/raquo/Laminar.git

# --- Experiment A: tuplez-full-light + arity 9 ---
cd Airstream
# Edit build.sbt: "tuplez-full" -> "tuplez-full-light", arity 22 -> 9
sbt 'set ThisBuild/version := "18.0.0-bench-A"' +publishLocal

cd ../Laminar
# Edit project/Versions.scala: Airstream = "18.0.0-bench-A"
sbt 'set ThisBuild/version := "18.0.0-bench-A"' publishLocal

# --- Experiment B: tuplez-full-light + arity 22 ---
cd ../Airstream
# Edit build.sbt: "tuplez-full" -> "tuplez-full-light", keep arity 22
sbt clean 'set ThisBuild/version := "18.0.0-bench-B"' +publishLocal

cd ../Laminar
# Edit project/Versions.scala: Airstream = "18.0.0-bench-B"
sbt clean 'set ThisBuild/version := "18.0.0-bench-B"' publishLocal

# --- Run benchmarks ---
cd compile-bench
./run-experiment.sh 3
```

## Project Structure

```
├── build.sbt                          # Multi-project sbt build
├── project/
│   ├── build.properties               # sbt 1.10.7
│   └── plugins.sbt                    # sbt-scalajs 1.20.2
│
├── bench-v17/src/main/scala/bench/    # 51 files — Laminar 17.2.0
├── bench-v18/src/main/scala/bench/    # 51 files — Laminar 18.0.0-M2 (stock)
├── bench-v18-A/src/main/scala/bench/  # 51 files — Laminar 18.0.0-bench-A (light, arity 9)
├── bench-v18-B/src/main/scala/bench/  # 51 files — Laminar 18.0.0-bench-B (light, arity 22)
│   ├── Model.scala                    # Sealed trait with 15 case classes
│   ├── HeavyCombine_01..10.scala      # combineWith chains, arities 2-5
│   ├── ReactiveDSL_01..10.scala       # Dense element trees, reactive bindings
│   ├── SplitPattern_01..10.scala      # signal.split on 15-subtype hierarchy
│   ├── StyleHeavy_01..10.scala        # Heavy StyleProp usage (.px, .em, reactive)
│   └── MixedComponent_01..10.scala    # All patterns combined (end-to-end)
│
├── micro-v17/src/main/scala/micro/    # 4 files — Laminar 17.2.0
├── micro-v18/src/main/scala/micro/    # 4 files — Laminar 18.0.0-M2
│
├── run-bench.sh                       # Original v17 vs v18 benchmark runner
├── run-experiment.sh                  # Experiment A/B benchmark runner
└── analyze.sh                         # Phase timing diff parser
```

## Benchmark Design

Source files simulate patterns from a real-world Laminar application:

| Category | Files | Pattern | Why it stresses the compiler |
|----------|-------|---------|------------------------------|
| HeavyCombine | 10 | 5+ `combineWith` chains, arities 2-5 | Overload resolution across 21 candidates per call |
| ReactiveDSL | 10 | Dense element trees with `<--`, `-->`, `cls` | Modifier implicit resolution |
| SplitPattern | 10 | `signal.split(_.id)` on 15-subtype sealed trait | Pattern matching + signal splitting |
| StyleHeavy | 10 | Heavy `StyleProp` with `.px`, `.em` | StyleProp trait mixin resolution |
| MixedComponent | 10 | Realistic mix of all patterns | End-to-end cost |
| Model | 1 | Sealed trait with 15 case classes | Shared data model |

## Versions

- **Scala**: 3.3.7
- **Scala.js**: 1.20.2
- **sbt**: 1.10.7
- **Laminar v17**: 17.2.0
- **Laminar v18**: 18.0.0-M2
- **Laminar v18-A**: 18.0.0-bench-A (Airstream: tuplez-full-light, arity 9)
- **Laminar v18-B**: 18.0.0-bench-B (Airstream: tuplez-full-light, arity 22)
