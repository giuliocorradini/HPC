# CUDA Streams

Represents a FIFO queue of GPU operations (such as kernle launches, memcopies and events) that get executed in a
specific order. The default stream (stream 0) is transparent.

Streams allows for concurrent execution of kernels and data transfers, the hardware can dynamically schedule operations.

To exploit this overlapping mechanism, you can use the `Async` variant of CUDA API and go on with host code execution.

Kernel execution in streams can be executed by passing four arguments (stream parameters) in angular brackets.

```c++
...
```

Data buffering is achieved through the exploitation of streams. If you're capable of divide your execution in different
ways, and send data transfers to a different stream, you can prefetch data on the device.

## Double buffering

You may also split memcopies in smaller units, and send them in different stream to start computation before the whole
data is transfered. In theory this increases the speedup.

Double buffering is useful when computing local algorithms, i.e. an algorithm that has no dependency on the whole image
and can be computed with a subset of the data. Different patches can in theory go in parallel.
