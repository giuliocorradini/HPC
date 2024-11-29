# CUDA memory

Cache loading is managed by hardware. Shared memory and constant cache was born for graphics computations.
Textures can be cached.

Column access of matrices is a problem of strided access: not only does it evict lines of cache every time.
Try to avoid strided access as much as possible, in the global memory.

In shared memory it's not so much of a problem since you have fine grained lines to access.

Constant data can be stored in a separate area, managed by hardware, and is declared with the `__constant__` keyword.
To load data from the host to the resident memory on GPU you will use `cudaMemcpyToSymbol` where the symbol is the
constant memory you're trying to load.

Moving the CCR we are moving the computation from memory-bounded to be compute-bounded.

## GEMM

Implement GEMM with CUDA shared memory.

Even with coalesced access, there is no gain. In theory the access pattern is changed, but there is no difference
performance-wise.

The reason it's so fast is because we're diminishing access to the main memory.

## Race condition

> An effect arising when multiple threads try to update a shared variable.

To handle the situation in OpenMP you can use `critical` or `atomic`. The directive make sure the operations will be
handled atomically.

Let's make a parallel image histogram. We have 255 buckets (corresponding to the possible levels of a grayscale image
pixel).

```c
for (int i=0; i<n; i++)
    for (int j=0; j<n; j++) {
        int lev = img[i][j];
        hist[lev]++;
    }
```

We have a shared variable update. CUDA supports atomics, and provides support for basic arithmetic, min, max, increment,
decrement, bitwise operations.

[CUDA C++ Programming Guide](https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html#atomic-functions)

Atomics are slower than normal access, but are convenient to use. Sometimes is more worthy to replicate data, since you
have massive parallelism. It's a tradeoff. This is a first option for when porting an algorithm.

To improve performance you can privatize memory and have each thread to have its own histogram. You can reserve
some shared memory and have rectangular blocks.
