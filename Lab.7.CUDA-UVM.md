# Unified Virtual Memory

UVM on integrated GPUs cannot provied concurrent access on memory, if you try to read it from the host while a kernel is
still running you will get a Segmentation Fault.

Access from the CPU and the GPU need to be synchronized.

```cuda
cudaDeviceSynchronize();
```

must be put after the kernel call to avoid reading memory that the GPU is accessing. What happens otherwise?

```
make: *** [run] Segmentation fault (core dumped)
```

When compute bound, never use UVM in discrete GPU. Pinned memory is basically not cacheable for the host, and that
introduces a huge slowdown for the host.
