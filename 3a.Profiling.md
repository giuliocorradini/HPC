# Profiling (lab)

Before improving the program you have to first understand where the performance pits are.

- Identify the hotspots: places of your code where you spend the most time.

- Partition: decompose in multiple jobs that hopefully run in parallel, and account for synchronization.

- Assess or evaluate that you successfully improved a metric of your system.

## Metrics

The capability of the hardware can be expressed in different ways:

- throughput, usually measured in FLOPS (floating point operations per second);

- memory bandwidth, how much memory can the system move from buffer to buffer;

- energy efficiency: throughput/power consumption measured in FLOPS/W. Important in embedded contexts 

The features of a hardware piece are also important: how many threads/cores? Vector units? Specialised hardware and
accerelators?

## Roofline model

The model for expressing the capability of a system. If your application is purely operational, moving to the right
of the graph you make the processor run in a flat region where the performance is limited by the diverging performance
of computing power and memory performance.

If the application is more memory intesive, you're working in the memory bound area: the performance grows as operational
intensity grows. The region is bound by the memory bandwidth of the system.

This model helps optimizing for computation intensity and avoids wasting time by being limited by memory.

## Profiling

Dynamic program analysis that measures the memory usage of a program, time complexity, usage of instructions, frequence
and duration of function calls.

Typically the system uses other applications, and estimated complexity can be better than real-time performance.

Some applications depend on the input data statistics (for example iterative algorithms), instructions can have different
execution time and complex memory hierarchies are not easy to predict (shared multi level cache).

To gather data you can use performance counters, hardware counters accessible from the software used for profiling.
You may also write custom code instrumentation, but there's an Heisenberg effect: instrumentation of code for profiling
can affect the code and give you different results.

Hardware interrupts: each time quant the execution is interrupted.

Finally you may use simulators, that are very slow, but you have a full introspection of the system.

### Output

Statistical summary of the events. Some tools give a time trace, or live monitoring hovering the window. FPS monitoring
in games is based on profiling.

## Perf

`perf` is a Linux profiling tools with performance counters. In the CPU there are counters that counts every aspect of
the system. This data goes under the name of perf_events.

```sh
module load perf/1.0
```

Given a command, you can measure the performance using:

```sh
perf stat [<command>]
```

Each CPU has its own performance monitoring unit, and each processor has a different list of counters that can 
monitored.

`perf stat dd if=/dev/zero of=/dev/null count=1000000` performs a million reads of dev zero and a million writes of dev
null.

**0,61  insn per cycle** comes from memory latency. We were expecting somewhat close to 1 operation per cycle.

If we stat the cache misses and dcache load, we can see they're equivalent: they map to the same counter.

Sample mulitple execution and average the perfomance to avoid potential cold cache misses that may false the results.

### gemm-s

gemmv2 performs a million more loads than gemmv1, gemmv1 is thrashed by 100x cache misses.

```
user@nano:~/hpc-lab-public-sources/profile/gemmv2$ perf stat -d ./gemmv2.exe 
gemm_opt on host :     2.512 sec     854.7 MFLOPS

 Performance counter stats for './gemmv2.exe':

       2514,610007      task-clock (msec)         #    0,999 CPUs utilized          
                 7      context-switches          #    0,003 K/sec                  
                 2      cpu-migrations            #    0,001 K/sec                  
             2.114      page-faults               #    0,841 K/sec                  
     3.719.041.623      cycles                    #    1,479 GHz                    
     7.716.192.075      instructions              #    2,07  insn per cycle         
   <not supported>      branches                                                    
        17.366.091      branch-misses                                               
     3.293.653.614      L1-dcache-loads           # 1309,807 M/sec                  
         7.444.294      L1-dcache-load-misses     #    0,23% of all L1-dcache hits  
   <not supported>      LLC-loads                                                   
   <not supported>      LLC-load-misses                                             

       2,518148876 seconds time elapsed

user@nano:~/hpc-lab-public-sources/profile/gemmv2$ perf stat -d ../gemmv1/gemmv1.exe 
gemm on host :     9.188 sec     233.7 MFLOPS

 Performance counter stats for '../gemmv1/gemmv1.exe':

       9189,957375      task-clock (msec)         #    1,000 CPUs utilized          
                13      context-switches          #    0,001 K/sec                  
                 2      cpu-migrations            #    0,000 K/sec                  
             2.110      page-faults               #    0,230 K/sec                  
    13.591.799.990      cycles                    #    1,479 GHz                    
     7.558.779.856      instructions              #    0,56  insn per cycle         
   <not supported>      branches                                                    
         1.181.922      branch-misses                                               
     2.169.295.803      L1-dcache-loads           #  236,051 M/sec                  
       558.277.967      L1-dcache-load-misses     #   25,74% of all L1-dcache hits  
   <not supported>      LLC-loads                                                   
   <not supported>      LLC-load-misses                                             

       9,194057731 seconds time elapsed

```

The gemm operation is a matrix multiplication, taking each row and column and performing the multiplication.
Gemm_opt uses another concept: there are two new functions, a reorder and a kernel. The new algorithm is avoiding
reading whole columns: it's performing tiling to avoid reading a column that wouldn't fit in cache.

> Naive implementation of an algorithm can bring degradation of performance because of the underlying architecture.

Perf is a coarse grained tool, you don't have notion about the part of the program that takes the most time. We ultimately
needed to open the editor and manully inspect the code to understand why.

## Gprof

You tipically want to enable profiling inside gcc. Enable profiling with `-pg` when compiling the code, and gcc instruments
the code to collect data when running.

The program generates a trace in the file `gmon.out`. `gprof` can then be run to inspect the result of the output.

The biggest con of gprof is that the call graph (the most useful thing) is **unreadable**. This should be used only to
get the main idea. Gprof tends to be hard to use in very complex applications.

There's a catch: it doesn't use hw performance counters like perf does. It does use them but sampling. GCC instruments
your code to monitor the status of perf counter every 100 seconds. Typically the information for execution time is not
perfect.

## Valgrind

It's a framework for debugging, profiling and even sanity check of memory accesses. Memcheck, cachegrind for cache
profiling and callgrind which performs function call profiling.

Memcheck detects memory management problems aimed primarily at C/C++ programmers. It detects memory leaks, and all the
ind of problems related to the use of malloc.

The application runs in a sandbox, while running Valgrind is able to profile every aspect of the application: even detect
memory leaks! That's the way it's able to detect this kind of bugs.

The drawback comes from the sandbox: this is a sort of emulation, and may come with slow downs.
Don't trust the profiling measurement, since it's slower take Valgrind measurement as a rough measure.
