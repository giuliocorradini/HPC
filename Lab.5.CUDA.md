# CUDA

Proprietary language by Nvidia. Not open source but very mature and stable. That determined the success of NVIDIA GPUs
because this programming model is very simple. Technically speaking other vendors have similar peak performance to NVIDIA's
but they have a very stable environment, very easy to use to exploit the behemot hardware.

You have two APIs: a runtime level which is simpler and higher level compared to the lower level **driver** API.
The driver level API, although more complex and verbose, allows more flexible programs (e.g. runtime compilation) and
is much closer to OpenCL. The runtime API is a composition of several calls to the driver API.

## First program: the kernel

```cuda
__global__ void helloworld_GPU(void) {
    printf("Hello world\n");
}

int main(void) {
    helloworld <<<1, 1>>>();

    return 0;
}
```

To launch a kernel, there is a special operator with three angular brackets. You can specify the `gridDim` which is the
number of instances of the kernel and `blockDim` wihich is the number of threads within each instance.

`<<<1, 1>>>` makes the kernel run on a single block with a single thread (only a single GPU ALU will be controlled).

`nvcc` is the NVIDIA compiler for CUDA, based on LLVM. Separates device code from host code which is compiled by the system
compiler. So you both specify `NVCC` and `CXX`.

## Global thread ID

The association between GID and blockIdx/threadIdx is important when designing a CUDA program. There is now way to execute
two blocks at the same time.

Beware of corner cases where the problem dimension is not an integer multiple of the block size: you have to take care
of that! Use this formula to compute the number of blocks to create:

$$
\lceil ({N \over \text{float}(M)}) \rceil
$$

To bound to N threads:

```c
kernel_signature(...) {
    int gid = threadIdx.x + blockDim.x * blockIdx.x;

    if (gid < N) {
        //run code...
    }
}
```

Use block size bigger than 32, because it's the warp size so the scheduler will divide the block execution into warps.

Try to generate as much blocks as possible: the more blocks you create, the more hardware can perform context switch to
hide memory latency.

## SAXPY

I can envision to build a grid where every single thread performs a scalar SAXPY. I proceed to compute the global thread
ID from the blockId, blockDim and threadId.

```c++
int i = blockId.x * blockDim.x + threadId.x;
```

I can add a check for corner cases where the grid size is not an integer multiple of N block size.

```c++
if (i < N)
    y[i] = a * x[i] + y[i];
```

Most of the time is spent in copying memory back and forth to the GPU. As stated by the profiler report:

```
==22039== Profiling result:
            Type  Time(%)      Time     Calls       Avg       Min       Max  Name
 GPU activities:   56.85%  738.82ms         3  246.27ms  237.42ms  251.19ms  [CUDA memcpy HtoD]
                   36.21%  470.50ms         2  235.25ms  234.89ms  235.61ms  [CUDA memcpy DtoH]
                    6.94%  90.235ms         1  90.235ms  90.235ms  90.235ms  saxpy(float*, float, float*, int)
      API calls:   50.74%  1.30310s         5  260.62ms  236.40ms  325.84ms  cudaMemcpy
                   46.64%  1.19759s         2  598.79ms  552.58ms  645.01ms  cudaMalloc
                    2.61%  66.952ms         2  33.476ms  6.9022ms  60.049ms  cudaFree
                    0.01%  158.65us         1  158.65us  158.65us  158.65us  cudaLaunchKernel
                    0.01%  128.75us        97  1.3270us     729ns  29.532us  cuDeviceGetAttribute
                    0.00%  11.250us         1  11.250us  11.250us  11.250us  cuDeviceTotalMem
                    0.00%  6.7710us         3  2.2570us  1.3540us  2.7090us  cuDeviceGetCount
                    0.00%  3.2290us         2  1.6140us  1.5630us  1.6660us  cuDeviceGet
                    0.00%  2.0310us         1  2.0310us  2.0310us  2.0310us  cuDeviceGetName
                    0.00%  1.9270us         1  1.9270us  1.9270us  1.9270us  cudaPeekAtLastError
                    0.00%     990ns         1     990ns     990ns     990ns  cuDeviceGetUuid
```

## MMUL

Let's try improving matrix multiplication performance on the GPU, use a two-dimensional grid.

In High Performance Computing you don't use matrices through double pointers, but you want to marshall the structure
and linearize it. At the end you will have an array.

If we use a grid size of 1x1, the block dimension is limited to 32 anyway. Generate multiple blocks, and let every thread
of the block work on multiple portions.
