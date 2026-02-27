# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Purpose

Benchmark proving the root cause of Laminar 18's 3.3x compile-time regression vs Laminar 17. The sole cause is increased combinator arity (22 overloads in v18 vs 7-8 in v17). Reducing arity to 9 restores v17-level compile times.

## Build & Run

```bash
# Build
sbt bench-v17/compile bench-v18/compile          # core comparison
sbt bench-v18-A/compile bench-v18-B/compile       # experiment variants

# Run benchmarks
./run-experiment.sh [iterations]   # default 3, runs all 4 variants
./run-bench.sh [iterations]        # default 5, includes micro benchmarks

# Analyze compiler phase timings from profile output
./analyze.sh

# Clean
sbt clean                          # all projects
sbt bench-v17/clean                # single project
```

## Architecture

**Build**: sbt 1.10.7, Scala 3.3.7, Scala.js 1.20.2. Six sub-projects in `build.sbt` sharing common settings.

**Four benchmark variants** (51 synthetic files each, identical user code):
- `bench-v17/` — Laminar 17.2.0 baseline
- `bench-v18/` — Laminar 18.0.0-M2 stock (tuplez-full, arity 22) — 3.3x slower
- `bench-v18-A/` — Laminar 18 + tuplez-full-light, arity 9 — **restores v17 speed**
- `bench-v18-B/` — Laminar 18 + tuplez-full-light, arity 22 — still 3.3x slow

**Two micro variants** (4 files each, isolated patterns):
- `micro-v17/`, `micro-v18/`

**Source patterns** (in each bench variant's `src/main/scala/bench/`):
- `Model.scala` — shared sealed trait `PdfObject` with 15 case classes
- `HeavyCombine_01..10` — `combineWith` chains (arities 2-5)
- `ReactiveDSL_01..10` — element trees with reactive bindings
- `SplitPattern_01..10` — `signal.split(_.id)` on sealed trait
- `StyleHeavy_01..10` — reactive StyleProp usage
- `MixedComponent_01..10` — realistic mix of all patterns

**Scripts**: `run-bench.sh` and `run-experiment.sh` do clean-compile timing with JVM warmup, statistics, and `-Vprofile` profiling. `analyze.sh` parses phase timing diffs.

## Search Exclusions

Do not search in: `out/`, `.bsp/`, `.metals/`, `.idea/`, `target/`, `profiles/`, `project/target/`
