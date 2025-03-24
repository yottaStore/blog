# The second iteration: Dynamo and Golang

To address Node.js's limitations, I switched to Golang for its performance and concurrency features.  My goal was
still to build a system closer to the original Dynamo paper's principles. For this iteration, I experimented with
using Amazon S3 as the underlying storage layer: this provided faster iteration speed, but S3's cost became a concern.

To reduce S3 costs, I explored using locally attached NVMe drives as a cache. This, however, plunged me into the
complexities of direct storage management. To interact directly with the NVMe hardware and bypass the operating
system's file system overhead, I had to use low-level Linux system calls like `ftruncate` (for resizing files),
`writev` (for writing multiple data buffers at once), and the `O_DIRECT` flag
(for bypassing the kernel's page cache).  Within this cache, I implemented partial indexes
using Log-Structured Merge Trees (LSM trees), a data structure well-suited for handling frequent writes.

This led to a radical rethinking of the architecture: What if we eliminated S3 entirely and relied solely on NVMe
drives?  The cost and complexity of managing the S3/NVMe interaction were becoming significant. My idea was to treat
the NVMe drives as a form of persistent, byte-addressable memory, similar to RAM but with different performance
characteristics.  After all, an NVMe drive is essentially a large, 0-indexed array of 4KB blocks, allowing
for random access.  This model would allow me to leverage the extensive research on in-memory, wait-free algorithms,
but applied to persistent storage. In this model I could reuse the large academic literature about in memory wait
free algorithms, imagining a collection...

To realize this vision, I needed a high-performance interface for managing concurrent operations on both the NVMe
drives and the network. The [NVMe ZNS specification](https://nvmexpress.org/specifications/) offered a promising
approach, providing a rich API to maximize NVMe performance. For asynchronous I/O operations, I turned to
the [io_uring](https://github.com/axboe/liburing) Linux kernel interface.

However, integrating `io_uring`, the NVMe ZNS API, and Golang proved incredibly challenging.  Golang's abstractions,
while generally beneficial, made it difficult to work with the low-level, memory-aligned operations required by
`io_uring` and NVMe.  This forced me to write significant amounts of unidiomatic Go code, sacrificing some of the
language's elegance and ergonomics.

And so, once again, I found myself at an impasse...

# Better indexes