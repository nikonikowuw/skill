---
name: rknn-pro
description: >
  Expert on Rockchip RKNN inference and media pipelines on RK3568 and RK3576 Linux systems —
  PyTorch/TF → ONNX → RKNN model conversion, zero-copy DMA-BUF pipeline design, RGA preprocessing,
  MPP video decode/encode, RKNN Runtime API usage, device-environment baselining, and memory alignment.
  Use this skill whenever the user mentions Rockchip, RKNN, RK3568, RK3576, MPP, RGA, librga, rknn-toolkit2,
  rknn Runtime, NPU, DMA-BUF, or Rockchip media pipelines. It replaces the old rockchip-performance skill
  with broader scope.
---

# rknn-pro

Build or tune Rockchip inference and media pipelines on RK3568 and RK3576 Linux systems. Covers: model
conversion (PyTorch/TF → ONNX → RKNN via rknn-toolkit2), NPU runtime (RKNN Runtime), media processing
(RGA, MPP), zero-copy DMA-BUF pipeline design, and device-environment baselining.

Prefer C/C++ for product runtime; use Python for model conversion, environment inspection, and benchmarks.

## Session Start: Device Identity Verification

**Every session** must verify the current board's serial number before using any cached context.

### Step 1 — Read existing context (if any)

If `.agents/rknn-context.md` exists, read it. Extract the stored `board_serial`.

### Step 2 — Get current board serial number

Ask the user to run this **one command** on the target board:

```bash
cat /proc/cpuinfo | grep "Serial" || cat /sys/class/soc/serial_number 2>/dev/null || cat /proc/device-tree/serial-number 2>/dev/null
```

The user pastes back the serial number (e.g., `Serial: 0123456789abcdef`).

### Step 3 — Compare

| Result | Action |
|---|---|
| **Match** (same board) | Cached context is valid. Proceed with `.agents/rknn-context.md`. |
| **Mismatch** (different board) | Do not use cached context. Run full initialization below for the new board. |
| No `.agents/rknn-context.md` exists | Run full initialization below. |

> ⚠️ Even if the SoC model is the same (both are RK3568), different serial numbers mean
> different physical hardware with potentially different BSP, kernel, or driver configurations.
> Always verify.

## Workflow

1. **Initialize project context** — collect board evidence + generate `.agents/rknn-context.md`.
2. **Identify the user's stage:**
   - **Model conversion** → route to [model-conversion.md](references/model-conversion.md) (PT/TF→ONNX→RKNN).
   - **Inference pipeline** → route to [zero-copy-pipeline.md](references/zero-copy-pipeline.md).
   - **API query** → route to [rknn-api-reference.md](references/rknn-api-reference.md), [rga-api-reference.md](references/rga-api-reference.md), or [mpp-api-reference.md](references/mpp-api-reference.md).
3. **For pipeline work**: draw buffer path → prefer DMA-BUF handoff → prove copies → tune one bottleneck.

### Zero-Copy Audit

When the user asks to audit an existing implementation for zero-copy:

1. Read [references/zero-copy-check.md](references/zero-copy-check.md) for the full audit procedure.
2. Identify the source files that handle buffer allocation, MPP decode, RGA processing, RKNN I/O.
3. Dispatch a **subagent** with the prompt template from `zero-copy-check.md` to analyze the code.
4. The subagent produces a structured report: pipeline map → per-stage audit → copy count table → score.
5. Present findings and fix recommendations to the user.

This is especially important for pipelines that claim zero-copy but have hidden `mmap` + `memcpy`
in the hot path, or use `rknn_inputs_set` with host buffers when `rknn_create_mem_from_fd` would work.

## Initialization: `.agents/rknn-context.md`

### When to initialize

- No `.agents/rknn-context.md` exists (first time working on this project).
- Serial number mismatch (different physical board).
- BSP/kernel/drivers have been updated.

### How to generate

**Step 1 — Collect board evidence**

If the user has board access:
```bash
bash scripts/detect-rockchip-env.sh      # SoC, kernel, BSP, NPU
bash scripts/collect-rockchip-debug.sh   # .so, symbols, headers
cat /proc/cpuinfo | grep Serial          # board serial number (context key)
```
If no direct access: give the user the command checklist from [first-response-template.md](references/first-response-template.md).

> ⚠️ **Board serial number is mandatory.** It uniquely identifies a physical Rockchip board.
> Even the same SoC with a different BSP version is a different context.

**Step 2 — Build baseline**
```bash
python3 scripts/render-project-baseline.py pasted-evidence.txt -o .agents/rknn-context.md
```

**Step 3 — Append API context**

After the baseline, append:
- Key API signatures from [rknn-api-reference.md](references/rknn-api-reference.md).
- Memory alignment rules from [memory-alignment.md](references/memory-alignment.md).
- Active RGA/MPP/RKNN version info from evidence.

**Step 4 — Set context ID**

Format: `{board_serial}-{soc}-{host_or_container}-{purpose}`.
Example: `0123456789abcdef-RK3568-host-video-infer`

## Start Here (quick helpers)

- `scripts/detect-rockchip-env.sh`
- `scripts/collect-rockchip-debug.sh`
- `scripts/render-project-baseline.py`

## References

### Inherited from rockchip-performance

| File | When to read |
|---|---|
| [soc-matrix.md](references/soc-matrix.md) | RK3568 vs RK3576 differences, default assumptions |
| [device-scoped-context.md](references/device-scoped-context.md) | Multiple boards, containers, or BSP images |
| [version-audit.md](references/version-audit.md) | BSP library version compatibility |
| [project-onboarding-workflow.md](references/project-onboarding-workflow.md) | Unfamiliar project onboarding |
| [device-evidence-workflow.md](references/device-evidence-workflow.md) | Collecting board info from user |
| [first-response-template.md](references/first-response-template.md) | First reply when no device baseline exists |
| [device-command-checklist.md](references/device-command-checklist.md) | Exact commands for user to run on board |
| [baseline-review-checklist.md](references/baseline-review-checklist.md) | Reviewing baseline draft output |
| [baseline-file-convention.md](references/baseline-file-convention.md) | Storing baseline in project |
| [zero-copy-pipeline.md](references/zero-copy-pipeline.md) | DMA-BUF pipeline, MPP/RGA zero-copy patterns |
| [zero-copy-check.md](references/zero-copy-check.md) | **Audit procedure** — dispatch subagent to check if implementation achieves max zero-copy |
| [perf-debugging.md](references/perf-debugging.md) | Throughput, CPU load, hidden copies, sync waits |

### New references for rknn-pro

| File | When to read |
|---|---|
| [model-conversion.md](references/model-conversion.md) | PT/TF→ONNX→RKNN via rknn-toolkit2, quantization, precompile |
| [rknn-api-reference.md](references/rknn-api-reference.md) | RKNN Runtime API signatures, parameters, memory modes |
| [rga-api-reference.md](references/rga-api-reference.md) | RGA im2d API, DMA-BUF import, format/alignment constraints |
| [mpp-api-reference.md](references/mpp-api-reference.md) | MPP decode/encode, external buffer mode, buffer pool |
| [memory-alignment.md](references/memory-alignment.md) | **Critical**: RGA stride alignment, MPP buffer size formulas |

## Key API Signatures (Inline Quick Reference)

### RKNN Runtime — Core

```c
// Initialize runtime
int rknn_init(rknn_context *ctx, void *model, size_t size, uint32_t flag, rknn_init_extend *extend);
// flag: 0 (default), RKNN_FLAG_PRIOR_MEDIUM, RKNN_FLAG_PRIOR_HIGH, RKNN_FLAG_PRIOR_LOW
//       RKNN_FLAG_ASYNC_MASK, RKNN_FLAG_COLLECT_PERF_MASK

// Query model I/O info
int rknn_query(rknn_context ctx, rknn_query_cmd cmd, void *info, size_t info_size);
// cmd: RKNN_QUERY_IN_OUT_NUM, RKNN_QUERY_INPUT_ATTR, RKNN_QUERY_OUTPUT_ATTR, etc.

// Set inputs
int rknn_inputs_set(rknn_context ctx, uint32_t n_inputs, rknn_input inputs[]);

// Run inference (sync)
int rknn_run(rknn_context ctx, rknn_run_extend *extend);

// Get outputs
int rknn_outputs_get(rknn_context ctx, uint32_t n_outputs, rknn_output outputs[], rknn_output_extend *extend);

// Release outputs
int rknn_outputs_release(rknn_context ctx, uint32_t n_outputs, rknn_output outputs[]);

// Destroy context
int rknn_destroy(rknn_context ctx);
```

### RKNN Runtime — Zero-Copy Memory

```c
// Create internal memory (device-side, accessible by NPU)
rknn_tensor_mem *rknn_create_mem(rknn_context ctx, size_t size);

// Create memory from DMA-BUF fd (zero-copy import)
rknn_tensor_mem *rknn_create_mem_from_fd(rknn_context ctx, int fd, void *priv_data, size_t size, int prot);

// Create memory from physical address
rknn_tensor_mem *rknn_create_mem_from_phys(rknn_context ctx, uint64_t phys_addr, size_t size);

// Set tensor with memory handle (zero-copy input)
int rknn_set_io_mem(rknn_context ctx, rknn_tensor_mem *mem, rknn_tensor_attr *attr);

// Destroy memory
int rknn_destroy_mem(rknn_context ctx, rknn_tensor_mem *mem);
```

### RKNN Runtime — NPU Core Mask (multi-core)

```c
// Set which NPU cores to use (RK3576 has 2 NPU cores)
int rknn_set_core_mask(rknn_context ctx, rknn_core_mask core_mask);
// RKNN_NPU_CORE_AUTO, RKNN_NPU_CORE_0, RKNN_NPU_CORE_1, RKNN_NPU_CORE_0_1
```

### RGA — Buffer Import and Processing (im2d API)

```c
// Import DMA-BUF fd for RGA processing
rga_buffer_handle_t importbuffer_fd(int fd, int size);

// Import virtual address buffer
rga_buffer_handle_t importbuffer_virtualaddr(void *virt_addr, int size);

// Release imported buffer
int releasebuffer_handle(rga_buffer_handle_t handle);

// Create RGA buffer handle from fd/virt
rga_buffer_t wrapbuffer_handle(rga_buffer_handle_t handle, int width, int height, int format, int w_stride, int h_stride);

// Resize
IM_STATUS imresize(const rga_buffer_t src, rga_buffer_t dst, double fx, double fy, int interpolation, int sync);

// Crop + resize
IM_STATUS imcrop(const rga_buffer_t src, rga_buffer_t dst, im_rect rect, int interpolation, int sync);

// Color space conversion
IM_STATUS imcvtcolor(rga_buffer_t src, rga_buffer_t dst, int sfmt, int dfmt, int mode);

// Flip / rotate
IM_STATUS imflip(const rga_buffer_t src, rga_buffer_t dst, int mode);
IM_STATUS imrotate(const rga_buffer_t src, rga_buffer_t dst, int rotation, int sync);

// Validate parameters before calling (returns IM_STATUS)
IM_STATUS imcheck(const rga_buffer_t src, rga_buffer_t dst, const im_rect *src_rect, const im_rect *dst_rect, int mode);

// Sync
IM_STATUS imsync(void);  // Wait for all RGA tasks complete
```

### MPP — Decode (External Buffer Mode)

```c
// Create MPP decoder
MPP_RET mpp_create(MppCtx *ctx, MppParam *param);
MPP_RET mpp_init(MppCtx ctx, MppCtxType type, MppCodingType coding);

// Set external buffer group
MPP_RET mpp_set_ext_grp(MppCtx ctx, MppBufferGroup group);

// Decode frame
MPP_RET mpp_decode_put_frame(MppCtx ctx, MppPacket packet);
MPP_RET mpp_decode_get_frame(MppCtx ctx, MppFrame *frame);

// Create external buffer group
MPP_RET mpp_buffer_group_get_external(MppBufferGroup *group, MppBufferMode mode, MppBufferType type);

// Import fd into buffer group
MPP_RET mpp_buffer_import(MppBufferGroup group, int fd, size_t size);

// Reset / destroy
MPP_RET mpp_reset(MppCtx ctx);
MPP_RET mpp_destroy(MppCtx ctx);
```

## Device-Scoped Context

Runtime context = `board_serial + SoC + kernel/BSP + driver versions + .so set + headers + container/rootfs + RKNN artifact`.

- Do not mix `librga.so`, `librknnrt.so`, `libmpp.so` across different boards/BSPs.
- Maintain separate context blocks (e.g., `0123456789abcdef-RK3568-host-video`).
- State which context is active before proposing code.

## Design Checklist

- [ ] Buffer origin: V4L2, MPP, DRM, custom allocator?
- [ ] Pixel format, W×H, stride, plane layout at each hop?
- [ ] Does next stage consume the same DMA-BUF fd directly?
- [ ] Is implicit conversion, cache sync, or software copy occurring?
- [ ] Is RGA used only where format/resize/crop/CSC is actually needed?
- [ ] Do RKNN inputs/outputs use runtime-managed or imported memory?
- [ ] Does CPU postprocess dominate after NPU inference?

## Operating Rules

- Keep data in DMA-BUF form across subsystems when APIs support it.
- Treat virtual-address processing as a fallback, not a default.
- Don't claim zero-copy until every ownership transfer, sync point, and format change is explained.
- Prefer explicit stage timing over whole-pipeline timing.
- Check version compatibility before deep changes — mismatched BSP/librga/rknnrt/MPP is a frequent root cause.

## Non-Goals

- Not generic CUDA, TensorRT, or non-Rockchip inference stacks.
- Not Android Java guidance.
- Not assuming all SoCs have identical RGA/MPP/NPU capabilities — require evidence.

## Deliverables

For substantial tasks produce:
1. Device-scoped project baseline
2. Buffer-flow summary
3. Bottleneck hypothesis
4. Code or config changes
5. Measurement method
6. Unverified board-specific risks
