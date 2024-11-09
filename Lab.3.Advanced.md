# Lab 3

Pi-finding code.

The problem with the for-loop is in area, you have an accumulator. In OpenMP you can achieve reduction with a custom
directive.

You may also share the variable but you will eventually occur in race conditions, and you have to synchronize.

Critical: only a single thread can enter a path of the code. Piece of code with multiple instructions.

Atomic: another directive applying to single operations only.

Reduction creates partial results, one for every thread, and at the end it adds a piece of code that performs
reduction, i.e. the operation of accumulating results in a single variable.

Critical section, coarse grained and software based. Atomic tries to map onto processors instructions.

In reduction, the accumulator variable is explictly separated. Formally speaking it's wrong to not put the `shared`
clause because you are not accumulating to a shared variable.

Critical sections are required to update a single variable across threads. For loops you can use smarter pattern like
reductions.

## Fibonacci

We end up with a huge creation of tasks. Overhead of tasks can prevent any speedup.

In any task, the job is simply to create other tasks... not very efficient isn't it?

We stop creating task at some point, so there are tree cutoff techniques. The if clause can be applied to the task and
tells the runtime to execute the task only if a condition is met.

```c++
#pragma omp task shared(x)
\ if (n > CUTOFF)
{
    x = fib(n-1);
}
```

I will still have a slowdown, because the if still performs some runtime functionality. Better to manually perform the
cutoff.

The cutoff number is architectural depend, it's how you control how many tasks you create.
