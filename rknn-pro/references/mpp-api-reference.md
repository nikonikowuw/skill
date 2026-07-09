# MPP API Reference

Rockchip MPP (Media Process Platform) provides hardware-accelerated video decode and encode.
This reference covers the C API with emphasis on zero-copy and external buffer mode.

## Overview

MPP provides hardware-accelerated video decode and encode on Rockchip SoCs. This reference
covers the C API with emphasis on zero-copy and external buffer mode.

## Decode Capabilities (by SoC)

> ⚠️ Capabilities vary by SoC and BSP version. The table below shows typical headline specs.
> **Always verify on the target board** — BSP drops may restrict features.

### Video Decode

| Codec | RK3568 | RK3576 | RK3588 |
|---|---|---|---|
| **H.264** | 1080p@60 | 4K@120 | 8K@60 |
| **H.265 / HEVC** | 1080p@60 | 4K@120 | 8K@60 |
| **VP9** | 1080p@60 | 4K@120 | 8K@60 |
| **AV1** | ❌ Not supported | 4K@120 | 8K@60 |
| **AVS2** | ❌ | 4K@120 | 8K@60 |
| **MJPEG** | 1080p@60 | 4K@30 | 4K@30 |
| **VC-1** | 1080p@30 | ❌ | ❌ |
| **MPEG-2/4** | 1080p@30 | ❌ | ❌ |

### Video Encode

| Codec | RK3568 | RK3576 | RK3588 |
|---|---|---|---|
| **H.264** | 1080p@30 | 4K@30 | 8K@30 |
| **H.265 / HEVC** | 1080p@30 | 4K@30 | 8K@30 |
| **MJPEG** | 1080p@30 | 4K@30 | 4K@30 |

### Key limitations

- **H.264**: Baseline / Main / High Profile up to Level 4.0 (RK3568), Level 5.1 (RK3576/RK3588).
- **H.265**: Main Profile up to Level 4.0 (RK3568), Level 5.1 (RK3576/RK3588).
- **AV1**: RK3568 has **no** AV1 hardware decoder. Use software decoding (limited to low resolution) or skip.
- **Max bitstream buffer**: varies — ensure buffer pool is sized correctly for peak bitrate.
- **Decoded output format**: Always YUV420SP (NV12) for video codecs. MJPEG can output YUV420SP or RGB.
- **Multiple instances**: Some SoCs support multiple concurrent decode sessions (check BSP docs).

### What MPP cannot do

| Operation | Alternative |
|---|---|
| Audio decode | ALSA / ffmpeg-software |
| Display output | DRM / Wayland / fbdev |
| Post-processing (denoise, deinterlace) | CPU NEON or RGA (limited) |
| Encoded stream muxing/demuxing | ffmpeg / GStreamer |
| Network streaming (RTSP, HLS) | ffmpeg / live555 / GStreamer |

## Decoder Memory Modes

MPP supports three decoder memory modes:

| Mode | Description | Zero-copy capable |
|---|---|---|
| **Pure internal** | MPP allocates and manages all buffers | No |
| **Half internal** | MPP allocates buffers, user can reference them | Limited |
| **Pure external** | User provides buffers via `MppBufferGroup` | **Yes** |

For zero-copy pipelines (decode → RGA → RKNN), always use **pure external mode**.

---

## Lifecycle

### `mpp_create` / `mpp_init`

```c
MPP_RET mpp_create(MppCtx *ctx, MppParam *param);
MPP_RET mpp_init(MppCtx ctx, MppCtxType type, MppCodingType coding);
```

Create and initialize MPP context.

| `type` | Description |
|---|---|
| `MPP_CTX_DEC` | Decoder |
| `MPP_CTX_ENC` | Encoder |

| `coding` | Codec |
|---|---|
| `MPP_VIDEO_CodingAVC` | H.264 |
| `MPP_VIDEO_CodingHEVC` | H.265 |
| `MPP_VIDEO_CodingVP9` | VP9 |
| `MPP_VIDEO_CodingAV1` | AV1 (device-dependent) |
| `MPP_VIDEO_CodingMJPEG` | MJPEG |

### `mpp_destroy`

```c
MPP_RET mpp_destroy(MppCtx ctx);
```

Destroy MPP context and free resources.

---

## External Buffer Group

### `mpp_buffer_group_get_external`

```c
MPP_RET mpp_buffer_group_get_external(MppBufferGroup *group, MppBufferMode mode, MppBufferType type);
```

Create an external buffer group. MPP will use user-provided buffers.

| `mode` | Description |
|---|---|
| `MPP_BUFFER_INTERNAL` | MPP-managed (for fallback) |
| `MPP_BUFFER_EXTERNAL` | User-managed (for zero-copy) |

### `mpp_buffer_import`

```c
MPP_RET mpp_buffer_import(MppBufferGroup group, int fd, size_t size);
```

Import a DMA-BUF fd into the external buffer group. This is the key zero-copy API.

- `fd`: DMA-BUF file descriptor
- `size`: buffer size

### `mpp_buffer_group_put`

```c
MPP_RET mpp_buffer_group_put(MppBufferGroup group);
```

Release the buffer group and all imported buffers.

### `mpp_set_ext_grp`

```c
MPP_RET mpp_set_ext_grp(MppCtx ctx, MppBufferGroup group);
```

Set the external buffer group for MPP context. Must be called after `mpp_create()` but before
the first decode frame.

---

## Decode

### `mpp_decode_put_frame`

```c
MPP_RET mpp_decode_put_frame(MppCtx ctx, MppPacket packet);
```

Send a compressed frame for decoding. Non-blocking (queues the packet).

### `mpp_decode_get_frame`

```c
MPP_RET mpp_decode_get_frame(MppCtx ctx, MppFrame *frame);
```

Get a decoded frame. Blocking — waits until a frame is ready.

### MPP Packet (input)

```c
MPP_RET mpp_packet_init(MppPacket *packet, void *data, size_t size);
void mpp_packet_deinit(MppPacket *packet);
void mpp_packet_set_pts(MppPacket packet, RK_S64 pts);
RK_S64 mpp_packet_get_pts(MppPacket packet);
```

### MPP Frame (output)

```c
// Get decoded frame info
RK_U32 mpp_frame_get_width(MppFrame frame);
RK_U32 mpp_frame_get_height(MppFrame frame);
RK_U32 mpp_frame_get_hor_stride(MppFrame frame);
RK_U32 mpp_frame_get_ver_stride(MppFrame frame);
MppFrameFormat mpp_frame_get_fmt(MppFrame frame);
MppBuffer mpp_frame_get_buffer(MppFrame frame);  // DMA-BUF buffer
RK_S64 mpp_frame_get_pts(MppFrame frame);
RK_U32 mpp_frame_get_errinfo(MppFrame frame);   // non-zero = decode error

// Get DMA-BUF fd from MppBuffer
int mpp_buffer_get_fd(MppBuffer buffer);
void *mpp_buffer_get_ptr(MppBuffer buffer);
size_t mpp_buffer_get_size(MppBuffer buffer);
```

---

## H.265 / HEVC Decode — Special Considerations

H.265 (HEVC) decoding on Rockchip MPP has specific requirements beyond basic MPP setup.
Based on real-world experience from Rockchip MPP GitHub issues and developer reports.

### Buffer pool sizing — critical

H.265 requires **more decode buffers** than H.264 due to the larger DPB (Decoded Picture Buffer)
for B-frames and hierarchical prediction:

| Codec | Minimum buffers | Recommended | Notes |
|---|---|---|---|
| H.264 | 16 | 20 | Reference: MPP developer guide |
| **H.265** | **20** | **24-30** | Larger DPB, especially for hierarchical B-frames |
| MJPEG | 4 | 8 | No reference frames |

```c
int h265_buffer_count = 24;  // H.265 needs more than H.264!
for (int i = 0; i < h265_buffer_count; i++) {
    int dma_fd = alloc_dma_buffer(buf_size);
    mpp_buffer_import(group, dma_fd, buf_size);
}
```

**⚠️ Deadlock trap:** If buffer pool is too small, `mpp_decode_put_frame` fails with
`MPP_ERR_BUFFER_FULL` while `mpp_decode_get_frame` returns empty — this creates an
**infinite deadlock loop** because you can't put new packets and can't get any frames.
The decoder just spins. Always size the pool generously (24-30 for H.265).

**⚠️ 8K OOM trap:** At 8K resolution, a single frame can be ~100 MB. 20+ buffers = ~2 GB total.
Monitor memory usage and enable Frame Buffer Compression (FBC) for 8K workloads.

### Resolution change — major pain point

H.265 streams commonly change resolution mid-stream (adaptive streaming, SVC, multi-profile).
MPP has known issues here:

**Known bug (RK3588, MPP issue #648):** H.265 decoder goes into an infinite
`"reset done"` loop after resolution change, producing **blurry output**.
H.264 decoder is NOT affected. No clean fix as of MPP develop branch — workaround:

```c
// Workaround for resolution change on H.265 (also helps RK3568/RK3576)
static uint32_t prev_width = 0, prev_height = 0;

while (has_data) {
    mpp_decode_put_frame(ctx, packet);
    if (mpp_decode_get_frame(ctx, &frame) == MPP_OK && frame) {
        uint32_t w = mpp_frame_get_width(frame);
        uint32_t h = mpp_frame_get_height(frame);
        uint32_t err = mpp_frame_get_errinfo(frame);

        // Check for resolution change or error
        if ((w != prev_width || h != prev_height) && prev_width > 0) {
            printf("H.265 resolution change: %dx%d -> %dx%d\n", prev_width, prev_height, w, h);
            // Full reset: destroy and recreate context + buffer group
            mpp_destroy(ctx);
            mpp_create(&ctx, NULL);
            mpp_init(ctx, MPP_CTX_DEC, MPP_VIDEO_CodingHEVC);
            mpp_buffer_group_get_external(&group, MPP_BUFFER_EXTERNAL, MPP_BUFFER_TYPE_DMA_BUF);
            mpp_set_ext_grp(ctx, group);
            for (int i = 0; i < h265_buffer_count; i++) {
                int dma_fd = alloc_dma_buffer(calc_mpp_decode_size(w, h));
                mpp_buffer_import(group, dma_fd, calc_mpp_decode_size(w, h));
            }
        } else if (err) {
            printf("H.265 decode error: %d\n", err);
            // Skip this frame, don't reset
        }
        prev_width = w;
        prev_height = h;

        if (!err && w > 0 && h > 0) {
            process_frame(mpp_frame_get_buffer(frame), w, h);
        }
        mpp_frame_deinit(&frame);
    }
    mpp_packet_deinit(&packet);
}
```

> On RK3588 with RTSP streams that dynamically change resolution, the H.265 decoder
> may enter an unrecoverable blurry state (issue #648). The safest workaround is to
> **destroy and recreate** the entire MPP context on resolution change, not just the buffer pool.
> H.264 does not have this problem.

### Low-latency decoding

Standard H.265 decode adds latency because MPP buffers frames in the DPB before output.

```c
// Try these MPP parameters for lower latency (YMMV by SoC and stream type):

RK_U32 immediate_out = 1;
mpp_control(ctx, MPP_DEC_SET_IMMEDIATE_OUT, &immediate_out);  // H.264 only — NO effect on H.265

RK_U32 disable_split = 1;
mpp_control(ctx, MPP_DEC_SET_PARSER_SPLIT_MODE, &disable_split);  // WORKS for H.265
```

| Setting | Effect on H.265 | Risk |
|---|---|---|
| `MPP_DEC_SET_IMMEDIATE_OUT` | **No effect** — documented for H.264 only. H.265 is unaffected (issue #503). | None |
| `MPP_DEC_SET_PARSER_SPLIT_MODE = 0` | **Significant latency reduction** (~11ms → ~4ms in one test, issue #503) | **May cause decode artifacts** on intra-refresh streams (issue #709) |
| Pure external buffer mode | **Reduces display latency** by avoiding extra copy | Requires correct buffer pool sizing and management |

> **Recommendation:** Start with `MPP_DEC_SET_PARSER_SPLIT_MODE = 0` for latency-sensitive H.265.
> If you see artifacts (flickering, corruption), enable it back and find other latency optimizations
> (e.g., reduce buffer count, optimize display pipeline).

### Short-file cycling — hidden memory leak

Repeatedly initializing and destroying MPP for short H.265 clips (e.g., 5-second segments)
can cause `MPP_ERR_BUFFER_FULL` after many cycles (MPP issue #?). The decoder accumulates
internal state that doesn't fully release.

```c
// BAD: Creates/destroys MPP per short clip
for each 5-second clip {
    mpp_create(); mpp_init(HEVC);
    decode_all_frames();
    mpp_destroy();   // internal state may leak
}
// After ~50 clips: MPP_ERR_BUFFER_FULL, decoder locked

// BETTER: Reuse MPP context across clips
mpp_create(); mpp_init(HEVC);
for each clip {
    mpp_reset(ctx);   // reset decoder state, keep context alive
    decode_all_frames();
}
mpp_destroy();
```

**Fix:** Use `mpp_reset()` instead of `mpp_destroy()` + `mpp_create()` between clips.
Or upgrade to MPP >= 1.0.11 which has improved buffer management for this pattern.

### Profile / Level constraints per SoC

| Constraint | RK3568 | RK3576 | RK3588 |
|---|---|---|---|
| Max level | **Main Profile, Level 4.0** | Main Profile, Level 5.1 | Main Profile, Level 6.0 |
| Max resolution | 1920×1088 | 4096×2304 | 8192×8192 |
| Bit depth | **8-bit only** | **8-bit only** | 8-bit + 10-bit (Main 10) |
| Chroma | 4:2:0 only | 4:2:0 only | 4:2:0 only |
| Tiles | ✅ | ✅ | ✅ |
| WPP | ✅ | ✅ | ✅ |
| SAO | ✅ Hardware | ✅ Hardware | ✅ Hardware |
| Deblocking | ✅ Hardware | ✅ Hardware | ✅ Hardware |

> ⚠️ **Main 10 (10-bit) on RK3568 and RK3576 is NOT supported.**
> If your H.265 stream is 10-bit:
> - **RK3568/RK3576**: ffmpeg software decode (slow) or pre-transcode to 8-bit server-side
> - **RK3588**: Main 10 is supported, but confirm in BSP docs
> - Workaround: Use `ffmpeg -pix_fmt yuv420p` to down-convert before feeding to MPP

### Stride alignment

```c
// H.265 decode output buffer sizing — same 16-byte stride as H.264
size_t hor_stride = ALIGN_UP(width, 16);
size_t ver_stride = ALIGN_UP(height, 16);
size_t buf_size = hor_stride * ver_stride * 2;  // safe total
```

H.265 and H.264 share the same 16-byte stride requirement. The key difference is the
**buffer pool count**, not stride alignment.

### Zero-copy encode failure after decode (RK3576/RK3588)

When chaining H.265 decode → encode (transcoding with zero-copy), the rk_vcodec driver
may fail to import the DMA-BUF fd from the decoder output (issue #80):

```
mpp_task_attach_fd failed
alloc task failed. ret: -5
```

**Root cause:** The decoder's output buffer fd is incompatible with the encoder's DMA-BUF
requirements on some BSP versions.

**Workaround:** Insert a CPU memcpy or RGA copy between decode and encode (breaks zero-copy,
but works reliably). Or use RGA to copy the decoded frame to a new DMA-BUF.

### Complete H.265 decode checklist

- [ ] Buffer pool sized to **24+** (not the default 16 from generic examples)
- [ ] Pure external buffer mode enabled (`MPP_BUFFER_EXTERNAL`)
- [ ] Resolution change handler destroys and recreates MPP context (not just buffer pool)
- [ ] 10-bit detection: if SoC is RK3568/RK3576, reject or down-convert 10-bit streams
- [ ] `MPP_DEC_SET_IMMEDIATE_OUT` NOT used (has no effect on H.265)
- [ ] `MPP_DEC_SET_PARSER_SPLIT_MODE = 0` tested for latency; artifacts checked
- [ ] `mpp_reset()` used instead of destroy/create when cycling between clips
- [ ] Memory usage monitored for 4K/8K streams (large buffer pool → OOM risk)
- [ ] Transcoding path tested for DMA-BUF fd compatibility (decode→encode)

## Encode

```c
// Create encoder context
mpp_create(&ctx, NULL);
mpp_init(ctx, MPP_CTX_ENC, MPP_VIDEO_CodingAVC);  // or HEVC

// Configure encoder
MPP_RET mpp_enc_cfg_set_s32(param, "type:rc_mode",  MPP_ENC_RC_CBR);      // CBR/VBR
MPP_RET mpp_enc_cfg_set_s32(param, "type:bps_target", target_bps);
MPP_RET mpp_enc_cfg_set_s32(param, "type:bps_max", max_bps);
MPP_RET mpp_enc_cfg_set_s32(param, "type:bps_min", min_bps);
MPP_RET mpp_enc_cfg_set_s32(param, "common:width", width);
MPP_RET mpp_enc_cfg_set_s32(param, "common:height", height);
MPP_RET mpp_enc_cfg_set_s32(param, "common:hor_stride", width_stride);
MPP_RET mpp_enc_cfg_set_s32(param, "common:ver_stride", height_stride);
MPP_RET mpp_enc_cfg_set_s32(param, "common:format", MPP_FMT_YUV420SP);
MPP_RET mpp_enc_cfg_set_s32(param, "common:fps_in_flex", fps_in_num);
MPP_RET mpp_enc_cfg_set_s32(param, "common:fps_out_flex", fps_out_num);
MPP_RET mpp_enc_cfg_set_s32(param, "common:qinit", qp_init);  // initial QP

// Send frame to encoder
mpp_frame_set_width(frame, width);
mpp_frame_set_height(frame, height);
mpp_frame_set_hor_stride(frame, width_stride);
mpp_frame_set_ver_stride(frame, height_stride);
mpp_frame_set_fmt(frame, MPP_FMT_YUV420SP);
mpp_frame_set_buffer(frame, mpp_buffer);  // import DMA-BUF
mpp_encode_put_frame(ctx, frame);

// Get encoded packet
mpp_encode_get_packet(ctx, &packet);
// ... write packet->data to file/stream
mpp_packet_deinit(packet);
```

---

## Buffer Size Calculation

For YUV420SP (NV12) decode buffers:

```c
// MPP decode buffer sizing rule of thumb
size_t hor_stride = ALIGN_UP(width, 16);    // horizontal stride
size_t ver_stride = ALIGN_UP(height, 16);   // vertical stride
size_t pixel_size = hor_stride * ver_stride * 3 / 2;  // YUV420SP
size_t extra_size = hor_stride * ver_stride / 2;      // extra info
size_t total_size = hor_stride * ver_stride * 2;      // safe total
```

H.264/H.265 decode typically needs **20+** buffers in the pool. Lower codecs (MJPEG) often need ~10.

---

## Typical Decode Loop (External Buffer)

```c
// === Setup ===
MppCtx ctx;
MppBufferGroup group;

mpp_create(&ctx, NULL);
mpp_init(ctx, MPP_CTX_DEC, MPP_VIDEO_CodingHEVC);

// Create external buffer group
mpp_buffer_group_get_external(&group, MPP_BUFFER_EXTERNAL, MPP_BUFFER_TYPE_DMA_BUF);
mpp_set_external_grp(ctx, group);

// Pre-import DMA-BUF buffers for the pool
for (int i = 0; i < 20; i++) {
    int dma_fd = alloc_dma_buffer(buf_size);  // from dma_heap, DRM, etc.
    mpp_buffer_import(group, dma_fd, buf_size);
}

// === Decode loop ===
MppPacket packet;
MppFrame frame;
while (has_data) {
    // Read compressed frame
    uint8_t *data = read_h265_frame(&size);
    mpp_packet_init(&packet, data, size);

    // Send to decoder
    mpp_decode_put_frame(ctx, packet);

    // Get decoded frame
    if (mpp_decode_get_frame(ctx, &frame) == MPP_OK && frame) {
        // Get DMA-BUF fd for zero-copy handoff to RGA/RKNN
        MppBuffer buf = mpp_frame_get_buffer(frame);
        int dma_fd = mpp_buffer_get_fd(buf);

        // Process frame (e.g., send to RGA)
        process_frame(dma_fd, mpp_frame_get_width(frame), mpp_frame_get_height(frame));

        mpp_frame_deinit(&frame);
    }
    mpp_packet_deinit(&packet);
}

// === Cleanup ===
mpp_buffer_group_put(group);
mpp_destroy(ctx);
```
