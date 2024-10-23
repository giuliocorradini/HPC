# OpenMP

Pargmas are suggestions for the compiler to do something. With `-fopenmp` the compiler is asked to preprocess the code
according to the pragmas omp.

Pragmas are not APIs per se, but they use suggestions to the compiler to perform code manipulation to achieve parallelization.
Every operation cannot be performed at build time, there's a runtime library that performs the other functions.

```sh
make EXT_CFLAGS='-DNTHREADS=10'
```

which is defined in "utils.h"

Thread order is random because of a resource concurrency problems about the stdout. You cannot predict how multiple
threads trying to acquire a logkc will come out. Order is not guaranteed.

At the end of a parallel block, there is an implicit synchronization barrier. We're adopting a fork-join scheme for
threading.

Given N cores, you can have at most N threads running in parallel and there is no specific need to run more than N
threads. Parallel systems aim to use all the cores all the time.

### Chunk size

If you think about the space of iterations, to execute it in parallel on N cores you divide the space in N even spaces
each assigned to a different thread.
To get the chunk size you simply divide the number of iterations by the number of threads.

A chunk is the minimum granularity in terms of iterations that you can assign.

## For-loop

With static scheduling

```c
#pragma omp parallel for num_threads(NTHREADS) schedule(static)
```

If the duration of the iteration is static and costant among different iterations, the best scheduling is the static.

Dynamic scheduling best suits situations where the workload changes across iterations: let's say the workload actually
increases with each iteration. If you used static scheduling you would end up in a situation where the first loop takes
all the shorter loads, whilst the last loop do all the heavy loops.

Dynamic scheduling is based on a working queue containing the chunks. Each thread queries the queue for work to do.
