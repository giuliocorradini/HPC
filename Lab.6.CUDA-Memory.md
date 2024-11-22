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