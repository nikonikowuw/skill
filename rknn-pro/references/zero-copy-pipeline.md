# Zero-Copy Pipeline Guide

## Goal

Keep video or camera frames in shared DMA-capable memory from producer to consumer, and avoid unnecessary CPU copies or remaps.

## Kernel-Level Ground Truth

The Linux V4L2 DMA-BUF importer API documents that:

- A device that supports streaming I/O can accept DMA-BUF-backed buffers via `V4L2_MEMORY_DMABUF`.
- DMA-BUF file descriptors are queued via `VIDIOC_QBUF`.
- Single-plane and multi-plane APIs both support DMA-BUF-backed queueing.

The Linux dma-buf documentation describes the model as:

- Shared buffers exposed as file descriptors
- Cross-device and cross-subsystem sharing
- Synchronization through `dma-fence` and `dma-resv`

That means a zero-copy design still needs correct synchronization. No memcpy does not mean no waits.

## Rockchip-Specific Pipeline Shape

On Rockchip Linux systems, the common high-performance path is:

`V4L2 capture or MPP decode -> DMA-BUF fd -> RGA -> DMA-BUF fd -> RKNN Runtime -> postprocess -> display or encoder`

The critical design question at each hop is:

- Does the next stage consume the same fd directly
- Or does the code map to CPU memory, repack, and submit a new buffer

## MPP Guidance

Rockchip MPP documentation describes three decoder memory modes. For zero-copy-oriented work:

- Pure internal mode is easy to start with, but difficult for zero-copy display style paths.
- Half internal mode gives more control, but still does not make zero-copy easy.
- Pure external mode is described as the most efficient way for zero-copy display paths.

When the code uses MPP decode and later wants display or accelerator sharing, inspect whether the decoder is stuck in internal or half-internal allocation mode.

### What Pure External Mode Implies In Practice

MPP's readme states that pure external mode requires the user to create an empty `MppBufferGroup` and import memory from an external allocator by file handle. The same document also gives a sizing rule of thumb for decode buffers:

- Pixel data: `hor_stride * ver_stride * 3 / 2`
- Extra info: `hor_stride * ver_stride / 2`
- Safe total: `hor_stride * ver_stride * 2`

It also notes that H.264 or H.265 often needs `20+` buffers, while other codecs often need `10`.

Design consequence:

- If a project wants decode-to-display or decode-to-preprocess zero-copy, it should own buffer-pool design explicitly instead of relying on decoder-private allocation.
- If buffer count is too low, the pipeline may look like a performance problem when it is actually backpressure.

## RGA Guidance

Rockchip's RGA FAQ documents several constraints that matter in zero-copy paths:

- `dma_fd` is generally the recommended memory type for balancing efficiency and usability.
- Virtual-address submission is slower and more CPU-sensitive.
- Different image formats have alignment requirements, especially stride and YUV geometry.
- Driver and `librga` version mismatches can cause compatibility mode or failures.
- `importbuffer_fd()` is intentionally expensive and should not be done every frame if the buffer set is reusable.

Design consequence:

- Prefer passing DMA-BUF fds into RGA.
- Keep a per-stage record of `format`, `width`, `height`, `w_stride`, `h_stride`, and plane layout.
- If RGA starts failing with parameter errors, check alignment and driver or library compatibility before rewriting the pipeline.
- If RGA is called on a rolling pool of buffers, import once and reuse `buffer_handle` objects rather than import and release every frame.

### RGA Alignment Rules That Commonly Bite

The RGA FAQ makes several specific points:

- Alignment requirements differ by format.
- RGA fetches image lines in 4-byte or 32-bit aligned units.
- `RGB565` needs 2-byte alignment.
- `RGB888` needs 4-byte alignment.
- YUV formats have special constraints: width stride commonly needs 4 alignment, and YUV dimensions or offsets often need 2 alignment.

It also shows a concrete `imcheck()` failure case where `NV12` width `1281` fails because YUV width is not aligned to `2`.

Use that as a first-pass filter when RGA rejects an otherwise plausible pipeline.

### DMA-BUF Is Better, Not Free

The RGA FAQ compares memory types and recommends `dma_fd` as the practical balance between efficiency and usability. It also notes that:

- Virtual-address paths add CPU cost for page-table work.
- Cacheable buffers can still trigger expensive cache synchronization even when using `dma_fd`.
- Common allocator behavior can change the cost profile.

Therefore:

- `dma_fd` is the preferred default, but high CPU with `dma_fd` can still be real.
- If CPU remains high, compare allocator behavior and cacheability before concluding the zero-copy design failed.

## Common Copy Traps

- V4L2 dequeue to CPU pointer, then memcpy into a new RKNN input buffer
- MPP decode to internal buffers, then software conversion before display or inference
- RGA called with virtual addresses because DMA-BUF wrapping was skipped
- RKNN output always read back to CPU even when only lightweight metadata is needed
- Hidden colorspace conversion in a stage that does not accept the upstream format

## What To Prove Before Claiming Zero-Copy

- Buffer producer type
- Original allocation owner
- Exported fd count and lifetime
- Which subsystem imports the fd next
- Whether any stage maps the buffer into CPU space for a full-frame walk
- Whether cache synchronization is required because a stage uses CPU-visible cached memory
- Whether fences or dequeue waits dominate runtime even though copies are gone

## Debug Procedure

1. Draw the exact buffer lineage from source to sink.
2. Log every fd import, wrap, map, unmap, and release.
3. Time each stage separately.
4. Compare a DMA-BUF path against a virtual-address path if unsure where the CPU is burning time.
5. If RGA is present, enable its logs or debug nodes before assuming the bottleneck is inference.
6. If decode is involved, inspect whether the external-buffer path is really active or only planned in comments.

## Sources

- Linux kernel V4L2 DMA-BUF importer API: https://docs.kernel.org/userspace-api/media/v4l/dmabuf.html
- Linux kernel dma-buf overview: https://docs.kernel.org/driver-api/dma-buf.html
- Rockchip MPP readme: https://github.com/rockchip-linux/mpp/blob/develop/readme.txt
- Rockchip librga FAQ: https://github.com/airockchip/librga/blob/master/docs/Rockchip_FAQ_RGA_EN.md
