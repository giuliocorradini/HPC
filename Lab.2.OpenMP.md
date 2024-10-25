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

## Sections

The main limitation is static dispatching, that makes sections very limited. All the tasks must be outlined and unrolled
manually.

To "section" the body of a for-loop you use `single`. We add `nowait`, because they otherwise have an implicit barrier.
Useful for list traversals.

```c++
#pragma omp parallel
for (auto el = list.begin(); el != list.end(); el++) {
    #pragma omp single nowait
    process(el);
}
```

Using sections you can traverse a tree.

```c++
#pragma omp parallel sections
{
    #pragma omp section
    visit(tree->left);

    #pragma omp section
    visit(tree->right);
}
```

That's very poor performance-wise. You spend a lot of time creating threads instead of having a fixed set of threads and
feeding them jobs. Moreover you may not create parallel regions inside other parallel regions.

In the Intel Compiler only the outmost parallel section is executed.

## Task parallelism

A better solution for those problems and the main addition to OpenMP. Really dynamic. Main addition to OpenMP 3.0
We have a queue and we enforce a producer/consumer scheme.

Tasks may be deferred, sections are instead executed in a lexically order. Tasks are putted in the queue and executed
later by some worker.

The implementation of a task requires a data structure with code to execute, data environemnt and internal contol variables.

```c
#pragma omp task [ clauses ]
```

Tasks can be everywhere, even nested. The task directive packs the job that should be done and put it in a queue.

### List traversal with tasks

```c++
for (auto el = list.begin(); el != list.end(); el++) {
    #pragma omp task
    process(el);
}
```

What's the scope of element? If no clause is applied to the variable, the scope is the default one. Global values are
shared, otherwise first private and shared attribute are lexically inherited.

Beware of scoping, it's the first source of problems in OpenMP applications.

### Barriering

To join all the created tasks, you put the `#pragma omp taskwait` directive.

Threads enclosing the parallel regions, the tasks are distributed across those threads.

## Dependencies

At the moment the execution of multiple tasks you can specify dependencies. You would usually put a barrier to implicit
wait for the dependencies to be met.

```c
#pragma omp task shared(x, ...) depend(out: x)
```

Dependencies can be out, in, inout.

With this feature we can create dependency graphs of tasks. You need not to put a barrier, but let the runtime understand
dependencies and linearize the graph in the compiler.

Runtime implementation is a bit complex and is not very well supported.
