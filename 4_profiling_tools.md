
# Finding bottlenecks, the simple story

You cannot optimize what you cannot measure.

Profiling tools tell you where your program spends its time and why it is slow.

## The time command

The simplest measurement tool is time.

```bash
time ./program
```

This gives you three numbers:
* **real:** Wall clock time from start to finish
* **user:** CPU time spent in your code
* **sys:** CPU time spent in the kernel

If real time is much larger than user time, your program is waiting. Maybe for I/O, maybe for network, maybe for other processes.

If user time is large, your program is CPU-bound. That is where optimization usually focuses.

## gprof: function-level profiling

gprof tells you which functions use the most time.

```bash
# Compile with profiling enabled
gcc -pg -O2 program.c -o program

# Run the program (creates gmon.out)
./program

# Generate the profile report
gprof program gmon.out > profile.txt
```

The output shows:
* How much time each function used
* How many times each function was called
* The call graph showing who called whom

This tells you which functions are hot spots.

## gcov: line-level profiling

gcov shows which lines of code executed and how often.

```bash
# Compile with coverage
gcc -fprofile-arcs -ftest-coverage program.c -o program

# Run the program
./program

# Generate coverage report
gcov program.c
```

The .gcov file shows execution counts for each line.

This is useful when a function is slow but you need to know which part of the function is the problem.

## perf: hardware counters

perf is a powerful Linux tool that measures hardware events.

```bash
# Basic statistics
perf stat ./program

# Detailed hardware counters
perf stat -e cycles,instructions,cache-misses,branch-misses ./program
```

This shows:
* **cycles:** How many CPU cycles the program took
* **instructions:** How many instructions executed
* **IPC (instructions per cycle):** Efficiency metric. Higher is better. Modern CPUs can achieve 2-4 IPC.
* **cache-misses:** How often data was not in cache
* **branch-misses:** How often branch prediction failed

These numbers tell you *why* code is slow.

**Low IPC?** The CPU is stalling. Maybe waiting for memory, maybe hitting data dependencies.

**High cache-misses?** Your memory access pattern is cache-unfriendly.

**High branch-misses?** Your branches are unpredictable.

## perf record and perf report

For detailed profiling, perf can record samples.

```bash
# Record profile
perf record -g ./program

# View the report
perf report
```

This shows a function-level breakdown like gprof, but with hardware event information.

The -g flag includes call graphs, so you can see the calling context.

## Performance counters: what to look for

Modern CPUs have dozens of hardware counters. Here are the most useful ones.

**Cycle counters:**
* Total cycles
* Cycles stalled on memory
* Cycles stalled on execution

**Memory counters:**
* L1/L2/L3 cache misses
* TLB misses
* Memory bandwidth used

**Branch counters:**
* Branches taken
* Branch mispredictions
* Misprediction rate

**Instruction counters:**
* Total instructions
* Instructions per cycle (IPC)
* Vectorized instructions executed

High cache miss rates mean you should improve locality.

High branch misprediction rates mean you should make branches more predictable or remove them.

Low IPC means the CPU is not being kept busy.

## Top-down microarchitecture analysis

Intel and AMD CPUs support top-down analysis.

This categorizes stalls into:
* **Frontend bound:** CPU waiting for instruction fetch/decode
* **Backend bound:** CPU waiting for data or execution units
* **Bad speculation:** Wasted work from branch mispredictions
* **Retiring:** Actual useful work

```bash
perf stat --topdown ./program
```

This tells you where the bottleneck is at the CPU microarchitecture level.

If backend bound is high, focus on memory access or data dependencies.

If bad speculation is high, fix branches.

## Flame graphs

Flame graphs visualize where time is spent in a call stack.

```bash
# Record with call stacks
perf record -g --call-graph dwarf ./program

# Generate flame graph (requires flamegraph scripts)
perf script | stackcollapse-perf.pl | flamegraph.pl > flame.svg
```

The graph shows the call stack as horizontal bars. Wider bars mean more time.

This makes it easy to spot hot paths through your code.

## Choosing the right tool

**Use time when:** You want a quick measurement of total runtime.

**Use gprof when:** You need to know which functions are slow.

**Use gcov when:** You need to know which lines inside a function are slow.

**Use perf when:** You need to understand hardware-level behavior like cache misses or branch prediction.

**Use flame graphs when:** You want to visualize complex call stack behavior.

## The profiling workflow

1. Start with time to see if there is a problem
2. Use gprof or perf to find hot functions
3. Use perf counters to understand why they are slow
4. Use gcov or perf report to find hot lines
5. Optimize the bottleneck
6. Measure again to confirm improvement

## One sentence to remember

Profiling tools like gprof, perf, and gcov tell you which code is slow and why by measuring function time, hardware events, and execution counts.
