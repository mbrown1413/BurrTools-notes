# Solver Benchmarks

I have benchmarks set up in a private repository. This is the standard
benchmark command I'll use here:

    $ time python verify.py --actions=benchmark --benchmark-reps 10 --filter-baseline-time 0-300

## No change

* [Benchmark graphs](benchmarks/no_change/benchmark_graphs.html)
* [Raw data](benchmarks/no_change/benchmark.csv)

As a baseline to show how much variance is typical, this compares two identical
versions of BurrTools.

## Max Holes

* Changes: Comment out the block at [assembler_1.cpp:1505](burr-tools/src/lib/assembler_1.cpp#L1505)
* [Benchmark graphs](benchmarks/max_holes/benchmark_graphs.html)
* [Raw data](benchmarks/max_holes/benchmark.csv)

Max holes doesn't seem to help when it's automatically determined, and in at
least one case it seems to hurt (`BillCutlerLattice.xmpuzzle`). I haven't found
any puzzles which actually use a manually set max holes.

## Range Column

* Changes: Add `hasRange = false;` to the start of [assembler_1_c::prepare](burr-tools/src/lib/assembler_1.cpp#L341)
* [Benchmark graphs](benchmarks/range_column/benchmark_graphs.html)
* [Raw data](benchmarks/range_column/benchmark.csv)

The range column doesn't appear to make any noticable difference in
performance. However, we should be able to simplify the code a fair amount if
we don't have a range column, since it's the only time node weights are used.

TODO: Make patch for removing weights entirely and run benchmarks on that.

## Matrix Reduction

## Recursive vs Iterative Implementations

## Compiler Options

`-O2` vs `-O3`