# Lab 4 Accelerator

Architecture have shifted to an heterogeneous system, which specialised hardware.

OpenMP evolved to target new accelerators, such as GPGPUs. They work in other addressing spaces, and you cannot have
a purely shared memory model, but you also have to map which kind of buffers should be used by GPUs.

### Select the appropriate compiler

We need to select clang instead of gcc. Since this target model for OpenMP is pushed by NVIDIA which historically used
LLVM, we need it. Also LLVM has better support for OpenMP.

## Mapping

You can specify memory to copy between host and target addressing spaces, and also direction. `to` tells OpenMP to copy
from host to device, while `from` qualifies memory to be copied from device back to the host.

Example of MM:

```c
double A[n][n], B[n][n], C[n][n];

#pragma omp target map(to: A, B) map(from: C)
{
    int n = 64;

    #pragma omp parallel for
    for (int i=0; i<n; i++)
        for (int j=0; j<n; j++)
            for (int k=0; k<n; k++)
                C[i, j] = A[i, k] * B[k, j]

}
```

Direction is provided to avoid unnecessary copies. `Parallel for` spawn threads and divide work among them.

GPUs are not typically allowed to have global synchronizations. OpenMP resolves with `teams`: you can synchronized only
inside teams and perform local synchronization in the local threads inside teams.
`parallel` is a flat parallelism that requires global parallelization.

## SAXPY on accelerator

`target` by itself is not sufficient and will trigger a segmentation fault. You have to map memory. Memory ranges are
specified to optimize memcopies and avoid useless copies.

Target offloads the execution of code to the master thread of the device. We aren't parallelizing work yet.
Unfortunately a single thread of a GPU is much less powerful than a CPU core. GPU streaming processors are less capable
and run at a lower clock frequency.

We're not still accelerating because flat parallelism in GPUs is not so efficient. GPUs basically groups the threads in
cluster of threads called streaming multiprocessors, and threads running in the same SMs can exploit very efficient hardware
synchronization otherwise it's a mess.

`teams` spawn threads that run in parallel but **are not synchronized**

```c
#pragma omp target teams num_teams(n/NTHREADS_GPU) thread_limit(NTHREADS_GPU) map(to: a, n, x[0:n])  map(tofrom: z[0:n])
#pragma omp distribute parallel for num_threads(NTHREADS_GPU) dist_schedule(static, NTHREADS_GPU)
for (i = 0; i < n; i++)
{
    z[i] = a * x[i] + z[i];
}
```

Proper way to parallelize core. Can achieve 100x, but we're still going slow. We are copying data.
With `nvprof` we can see that more than 70% of time is spent in CUDA memcpy host-to-device, while only 6% of time is spent
in the function `__omp_offloading...` which is the name given by the compiler to our SAXPY.

GPU usage is not for free. In cases where data is used once, the cost is bound by the copies more than the computation.

## Matmul

Trivial changes: b2 declaration is moved in b2, which is used by reorder2 and kernel. It's necessary as it's equivalent
to having a global shared variable across all the teams and it's going to destroy the execution times.

To have a private variable, it's best to declare it inside the loop.

`collapse` tries to merge the iteration space of N iteration groups, so distribute parallel for now applies not just to
i-iteration space but also to j and k iteration space.

OpenMP implementation of the code is not really optimized with what you could do in CUDA. We could achieve 400GFLOPS.

You can manually unroll the GEMM loops, or use cuBLAS which is CUDA implementation for basic linear algebra programs.

But cuBLAS still performs poorly, why? Because it's optimized for very large matrices. Think about of who's going to use
supercomputers: scientists that are performing simulations. For them learning how to use CUDA is impossible because you
need a background in programming and systems architecture. A way to access GPUs with the paradigm they already know, is
still reasonable.

Jacobi shows it's better to perform two loop inside the GPU.