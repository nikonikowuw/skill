# Rockchip Memory Alignment Reference

Rockchip hardware (RGA, MPP, RKNN NPU) enforces strict alignment requirements on buffer addresses,
widths, heights, strides, and total sizes. Misaligned buffers cause silent data corruption or
runtime errors.

## Quick Reference Table

| Operation | Width Stride Align | Height Stride Align | Buffer Addr Align | Size Formula |
|---|---|---|---|---|
| **RGA** RGB565 | 2 | 1 | 4 | `ws * hs * 2` |
| **RGA** RGB888 | 4 | 1 | 4 | `ws * hs * 3` |
| **RGA** RGBA/BGRA8888 | 4 | 1 | 4 | `ws * hs * 4` |
| **RGA** NV12/NV21 | 4 | 2 | 4 | `ws * hs * 3 / 2` |
| **MPP** decode (YUV420SP) | 16 | 16 | varies | `ws * hs * 2` (safe) |
| **MPP** encode (YUV420SP) | 16 | 16 | varies | `ws * hs * 3 / 2` |
| **RKNN** `rknn_inputs_set` (host→NPU copy) | model-defined stride | model-defined stride | 4 (FP32) / 2 (FP16) / 1 (UINT8) | from `rknn_tensor_attr.size` |
| **RKNN** `rknn_create_mem` (NPU-side) | model-defined stride | model-defined stride | page (4KB, dma-heap) | as requested |
| **RKNN** `rknn_create_mem_from_fd` (DMA-BUF import) | set w_stride to match source | set h_stride to match source | page (4KB, from source allocator) | from source buffer |
| **RKNN** w_stride / h_stride in `rknn_set_io_mem` | must match actual buffer stride (query via `rknn_query`) | must match actual buffer stride | N/A | from `rknn_tensor_attr` |
| **DMA-BUF** dma_heap | 4 (varies) | 4 (varies) | page (4KB) | as requested |

> **ws** = width_stride, **hs** = height_stride.
> Alignment may vary by BSP version and SoC (RK3568 vs RK3576).

## Alignment Calculation Functions

### C / C++

```c
#define ALIGN_UP(x, align)  (((x) + (align) - 1) & ~((align) - 1))

// RGA NV12 buffer size
size_t calc_rga_nv12_size(int32_t width, int32_t height) {
    int32_t ws = ALIGN_UP(width, 4);     // RGA NV12 width stride = 4-byte aligned
    int32_t hs = ALIGN_UP(height, 2);    // RGA NV12 height stride = 2-byte aligned
    return (size_t)ws * hs * 3 / 2;
}

// MPP decode buffer size (safe total)
size_t calc_mpp_decode_size(int32_t width, int32_t height) {
    int32_t ws = ALIGN_UP(width, 16);    // MPP stride = 16-byte aligned
    int32_t hs = ALIGN_UP(height, 16);   // MPP stride = 16-byte aligned
    return (size_t)ws * hs * 2;          // MPP safe total = ws * hs * 2
}

// RGA RGB888 buffer size
size_t calc_rga_rgb888_size(int32_t width, int32_t height) {
    int32_t ws = ALIGN_UP(width, 4);     // RGB888 stride = 4-byte aligned
    return (size_t)ws * height * 3;
}
```

### Python

```python
def align_up(x, align):
    return (x + align - 1) // align * align

def rga_nv12_size(width, height):
    ws = align_up(width, 4)
    hs = align_up(height, 2)
    return ws * hs * 3 // 2

def mpp_decode_size(width, height):
    ws = align_up(width, 16)
    hs = align_up(height, 16)
    return ws * hs * 2
```

## RKNN Memory Alignment Detail

RKNN memory alignment depends on the **data path**:

### Path 1: `rknn_inputs_set` (host memory → NPU, with internal copy)

```c
rknn_input input;
input.buf = host_buffer;           // host-side buffer
input.size = image_size;           // total bytes
input.pass_through = 0;            // runtime handles quantize
input.type = RKNN_TENSOR_UINT8;
input.fmt = RKNN_TENSOR_NHWC;
rknn_inputs_set(ctx, 1, &input);
```

- No strict buffer address alignment — the runtime reads from host memory via DMA.
- For `UINT8`: 1-byte alignment is sufficient.
- For `FP32`: 4-byte alignment recommended.
- For NEON-optimized CPU preprocessing feeding this path: **16-byte alignment** improves performance.

### Path 2: `rknn_create_mem` (NPU-managed memory, zero-copy)

```c
rknn_tensor_mem *mem = rknn_create_mem(ctx, size);
```

- Allocated from dma-heap / ion, physically contiguous.
- Address is **page-aligned (4KB)** — more than enough for NPU access.
- Use with `rknn_set_io_mem` to bind to a tensor.

### Path 3: `rknn_create_mem_from_fd` (DMA-BUF import, zero-copy)

```c
rknn_tensor_mem *mem = rknn_create_mem_from_fd(ctx, dma_fd, NULL, size, PROT_READ);
```

- Imports an existing DMA-BUF from RGA, MPP, or dma_heap.
- Address alignment is inherited from the source (typically 4KB page-aligned).
- **Critical**: `w_stride` / `h_stride` in `rknn_tensor_attr` must match the actual buffer layout.
  Query the model's expected stride via `rknn_query`, then override with the real stride.

### Path 4: `rknn_set_io_mem` with w_stride / h_stride

```c
rknn_tensor_attr attr;
attr.index = 0;
rknn_query(ctx, RKNN_QUERY_INPUT_ATTR, &attr, sizeof(attr));

// Override stride to match actual buffer (e.g., from RGA)
attr.w_stride = 640;     // actual buffer width stride (not necessarily 640)
attr.h_stride = 640;
rknn_set_io_mem(ctx, mem, &attr);
```

- `w_stride` is in **elements** (not bytes) for NHWC/NCHW formats.
- If w_stride ≠ width, the NPU reads `w_stride` elements per row, skipping the padding.
- Incorrect w_stride/h_stride cause **garbage output** (NPU reads wrong memory locations).
- Query the model's default stride with `rknn_query` and adjust.

## Why Alignment Matters

Rockchip hardware processes pixels in fixed-size blocks:
- RGA fetches lines in 4-byte or 32-bit aligned units.
- MPP video codecs operate on 16×16 macroblocks.
- YUV420 chroma operates on 2×2 blocks.

Misaligned dimensions cause:
1. **RGA `imcheck` failure** or silent data corruption.
2. **MPP decode artifacts** at right/bottom edges.
3. **RKNN tensor stride mismatch** leading to garbage NPU output.

## RGA Format-Specific Alignment

### NV12 (YUV420SP)

```c
// CORRECT
int w_stride = ALIGN_UP(width, 4);    // width 1281 -> stride 1284 (not 1281!)
int h_stride = ALIGN_UP(height, 2);
int size = w_stride * h_stride * 3 / 2;

// WRONG — will fail imcheck or corrupt
int size = width * height * 3 / 2;
```

**Common failure:** `NV12` width `1281` fails because 1281 is not aligned to 2 (let alone 4).
Always align before passing to RGA.

### RGB888

```c
int w_stride = ALIGN_UP(width, 4);     // 4-byte alignment
```

### RGB565

```c
int w_stride = ALIGN_UP(width, 2);     // 2-byte alignment
```

## MPP Buffer Sizing

MPP's decode buffer sizing:

```text
Pixel data: hor_stride * ver_stride * 3 / 2
Extra info: hor_stride * ver_stride / 2
Safe total: hor_stride * ver_stride * 2
```

The "safe total" includes padding for hardware alignment and metadata.
When in doubt, use `hor_stride * ver_stride * 2`.

H.264/H.265 decode typically needs a pool of **20+** buffers.
MPP pure external mode: user provides buffers via `mpp_buffer_import`.

## RKNN Tensor Stride

When using zero-copy (`rknn_create_mem_from_fd` or `rknn_set_io_mem`), the tensor attributes
`w_stride` and `h_stride` must match the actual buffer layout:

```c
rknn_tensor_attr attr;
rknn_query(ctx, RKNN_QUERY_INPUT_ATTR, &attr, sizeof(attr));

// Update strides to match the actual buffer (e.g., from RGA output)
attr.w_stride = ALIGN_UP(model_width, 4);    // must match buffer stride
attr.h_stride = ALIGN_UP(model_height, 2);   // must match buffer stride

rknn_set_io_mem(ctx, input_mem, &attr);
```

If strides don't match, NPU will read/write at wrong offsets and output will be garbage.

## Common Mistakes

1. **Passing unaligned width/height to RGA** → `imcheck` fails or silent corruption.
2. **Using `width * height * 3 / 2` instead of aligned stride formula** → buffer too small.
3. **MPP decode using internal mode when zero-copy needed** → can't get DMA-BUF fd.
4. **RKNN passthrough input without setting correct `w_stride`/`h_stride`** → NPU misreads data.
5. **Reusing `importbuffer_fd` every frame** → performance regression (import is expensive; do once).
6. **Assuming RK3568 and RK3576 have identical alignment requirements** → verify on target BSP.

## Verification Snippet

```c
#include <assert.h>

void check_rga_alignment(const char* label, int width, int height, int format) {
    int ws_align = 4;  // default for RGB888/RGBA/NV12
    int hs_align = 1;  // default for RGB

    if (format == RK_FORMAT_RGB_565) {
        ws_align = 2;
    } else if (format == RK_FORMAT_YCbCr_420_SP || format == RK_FORMAT_YCbCr_422_SP) {
        hs_align = 2;
    }

    assert(width % ws_align == 0 && "RGA width alignment failed");
    assert(height % hs_align == 0 && "RGA height alignment failed");
    printf("[%s] width=%d (align=%d) height=%d (align=%d) OK\n",
           label, width, ws_align, height, hs_align);
}
```
