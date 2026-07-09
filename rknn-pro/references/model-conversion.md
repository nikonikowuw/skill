# Model Conversion Guide (Framework → ONNX → RKNN)

Full model conversion pipeline for Rockchip NPU: from PyTorch or TensorFlow to ONNX, then ONNX to RKNN
via rknn-toolkit2. Also covers direct framework-to-RKNN paths.

---

## 1. PyTorch → ONNX

### Basic export

```python
import torch

model = torch.load("model.pt")
model.eval()

dummy_input = torch.randn(1, 3, 224, 224)

torch.onnx.export(
    model,
    dummy_input,
    "model.onnx",
    opset_version=11,       # RKNN supports opset 11-15; 13 is a safe default
    input_names=["input"],
    output_names=["output"],
    dynamic_axes={
        "input": {0: "batch_size", 2: "height", 3: "width"},
        "output": {0: "batch_size"}
    }  # omit for static shapes
)
```

### Key parameters

| Parameter | Recommendation |
|---|---|
| `opset_version` | 11–15; 13 is safe for RKNN |
| `dynamic_axes` | Use for variable batch/resolution; verify runtime support in rknn-toolkit2 |
| `input_names` / `output_names` | Match names used during RKNN conversion |

### Common issues

- **Dynamic control flow**: Use `torch.jit.script()` first if model has data-dependent branches.
- **Unsupported ONNX ops**: Check rknn-toolkit2 supported operator list.
- **Verification**:
  ```python
  import onnx
  onnx.checker.check_model("model.onnx")
  ```

---

## 2. TensorFlow → ONNX

### Using tf2onnx

```bash
pip install tf2onnx onnx

# From SavedModel
python -m tf2onnx.convert \
    --saved-model ./saved_model \
    --output model.onnx \
    --opset 13

# From frozen .pb
python -m tf2onnx.convert \
    --input model.pb --inputs input:0 --outputs output:0 \
    --output model.onnx --opset 13
```

### NHWC → NCHW

TensorFlow defaults to NHWC. Rockchip RKNN commonly expects NHWC (TensorFlow-compatible).
If the model was trained in NCHW, use `--inputs-as-nchw` in tf2onnx or transpose in preprocessing.

---

## 3. ONNX → RKNN (rknn-toolkit2)

### Basic Python conversion script

```python
from rknn.api import RKNN

rknn = RKNN(verbose=True)

# Load ONNX model
ret = rknn.load_onnx(model="model.onnx")
assert ret == 0, "load_onnx failed"

# Build (quantize and convert)
ret = rknn.build(do_quantization=True, dataset="./dataset.txt")
assert ret == 0, "build failed"

# Export RKNN
ret = rknn.export_rknn("./model.rknn")
assert ret == 0, "export failed"

rknn.release()
```

### Pre-Processing Configuration

```python
rknn.config(
    mean_values=[[123.675, 116.28, 103.53]],   # per-channel mean
    std_values=[[58.395, 58.395, 58.395]],      # per-channel std
    target_platform="rk3568",                    # or rk3576
    quantized_dtype="asymmetric_quantized-8",    # quantization type
    quantized_algorithm="normal",                # or "mmse" for better accuracy
    batch_size=1,
)
```

### ⚠️ Normalization = Input must be [0, 255] uint8

**Critical rule:** When you set `mean_values` and `std_values` in `rknn.config()`,
the normalization is **baked into the RKNN model graph**. The model's first layer
becomes: `(input - mean) / std`.

This means:

| `mean_values` / `std_values` set? | Runtime input must be | Runtime `rknn_input` type |
|---|---|---|
| **Yes** (normalization in model) | **Raw [0, 255] uint8** | `RKNN_TENSOR_UINT8` |
| **No** (no normalization in model) | **Normalized float32** (as trained) | `RKNN_TENSOR_FLOAT32` or do normalization yourself |

**Common mistake — double normalization:**

```c
// ❌ WRONG: CPU normalizes + model also normalizes = garbage output
float normalized[640 * 480 * 3];
for (int i = 0; i < size; i++)
    normalized[i] = (image[i] / 255.0f - mean) / std;  // normalization on CPU

rknn_input input;
input.buf = normalized;           // already normalized
input.type = RKNN_TENSOR_FLOAT32; // but model will normalize again!

// ✅ CORRECT: feed raw uint8, let model handle normalization
rknn_input input;
input.buf = image;                // raw [0,255] uint8 pixels
input.type = RKNN_TENSOR_UINT8;   // model applies mean/std internally
input.fmt = RKNN_TENSOR_NHWC;
input.pass_through = 0;           // runtime quantizes to INT8
```

**Why this matters for zero-copy:**
- If model expects [0,255] uint8, you can feed the **raw camera/decoder output** directly
  (`rknn_inputs_set` with `RKNN_TENSOR_UINT8`) — no CPU-side normalization loop.
- If model expects normalized float32, you must run the normalization on CPU first
  (extra pass over the entire image), which kills the benefit of zero-copy.

**Check what your model expects:**
```python
# During conversion, inspect the model's expected input type
import onnx
m = onnx.load("model.onnx")
input_tensor = m.graph.input[0]
type_info = input_tensor.type.tensor_type
elem_type = type_info.elem_type  # 1=float32, 2=uint8, ...
```
If the ONNX model expects **float32** input, the RKNN model after `mean/std` config
will expect **uint8 [0,255]** as input (because the normalization is baked in).

### ⚠️ Channel order: RGB vs BGR

**Different models expect different channel orders.** Feeding RGB data to a BGR-trained
model (or vice versa) produces silently wrong results — colors are swapped.

| Training framework | Typical training format | Common practice |
|---|---|---|
| PyTorch / torchvision | **RGB** (C×H×W: 3,224,224) | Images loaded as RGB |
| TensorFlow / Keras | **RGB** (H×W×C: 224,224,3) | Images loaded as RGB |
| OpenCV (C++) | **BGR** (H×W×C: rows, cols, 3) | cv::imread returns BGR |
| Caffe | **BGR** | BVLC models expect BGR |
| Darknet (YOLO) | **BGR** | YOLO trained on BGR |
| ONNX Model Zoo | **RGB** | Most ONNX models are RGB |

**How to check the channel order for an ONNX model:**

```python
import onnx

m = onnx.load("model.onnx")

# Look at the mean_values in preprocessing info (if available)
# RGB models typically use means like [123.675, 116.28, 103.53] (R mean, G mean, B mean)
# BGR models typically use means like [103.53, 116.28, 123.675] (B mean, G mean, R mean)

# Or inspect the model's documentation / training script
# Or run a test: feed an image with known colors (red square) and check output
```

**In RKNN config — `mean_values` order tells you the expected format:**

| `mean_values` | Expected input order |
|---|---|
| `[123.675, 116.28, 103.53]` | **RGB** (R mean first) |
| `[103.53, 116.28, 123.675]` | **BGR** (B mean first) |
| `[0, 0, 0]` (no normalization) | Check model docs — ambiguous! |

**In the pipeline — matching channel order:**

```
Source format → RGA conversion → Model expected format

NV12 (MPP decode) → RGA CSC → RGB888 → model expects RGB  ✅
NV12 (MPP decode) → RGA CSC → BGR888 → model expects BGR  ✅
NV12 (MPP decode) → RGA CSC → RGB888 → model expects BGR  ❌ colors swapped!
NV12 (MPP decode) → RGA CSC → BGR888 → model expects RGB  ❌ colors swapped!
```

**How to specify the channel order in RKNN config:**

If your model expects a different order than what the training framework outputs,
you can sometimes handle it during conversion:

```python
rknn.config(
    mean_values=[[123.675, 116.28, 103.53]],   # R, G, B means
    std_values=[[58.395, 58.395, 58.395]],
    target_platform="rk3568",
    quantized_dtype="asymmetric_quantized-8",
    # Note: RKNN does NOT have a direct "channel_order" parameter
    # The mean_values/std_values order implicitly defines the expected order
)
```

**If you cannot change the model, change the preprocessing:**

```c
// RGA CSC: NV12 → RGB888  (feed to RGB model)
// RGA CSC: NV12 → BGR888  (feed to BGR model)

// Or swap channels on CPU before RKNN input
if (model_expects_bgr && source_is_rgb) {
    for (int i = 0; i < width * height; i++) {
        uint8_t r = rgb[i * 3 + 0];
        uint8_t g = rgb[i * 3 + 1];
        uint8_t b = rgb[i * 3 + 2];
        // Swap R↔B
        bgr[i * 3 + 0] = b;
        bgr[i * 3 + 1] = g;
        bgr[i * 3 + 2] = r;
    }
}
// ⚠️ This is a full-frame CPU pass — breaks zero-copy.
// Better: chain RGA to output the correct order directly.
```

**Zero-copy implication:** RGA can output both RGB888 and BGR888 via CSC.
Use the correct RGA format to match the model — no CPU channel swap needed:

```c
// RGA output format choice depends on model:
rga_buffer_t dst = wrapbuffer_handle(dst_handle, w, h,
    (model_expects_rgb ? RK_FORMAT_RGB_888 : RK_FORMAT_BGR_888),  // ⚠️ choose here!
    w_stride, h_stride);
```

### Quantization Guide

| `do_quantization` | Accuracy | Performance | Use case |
|---|---|---|---|
| `False` | FP16 (highest) | Lower | Debug, accuracy-sensitive |
| `True` (INT8) | Slightly lower | 2-4x faster | Production deployment |

Dataset file (`dataset.txt`): one image path per line for calibration.

```text
./calib_images/img001.jpg
./calib_images/img002.jpg
./calib_images/img003.jpg
```

### Target Platforms

| Value | SoC |
|---|---|
| `rk3568` | RK3566 / RK3568 |
| `rk3576` | RK3576 |
| `rk3588` | RK3588 / RK3588S |

### Pre-Compile for Faster Loading

```python
rknn.build(do_quantization=True, dataset="./dataset.txt", pre_compile=True)
```

Pre-compiled models load faster at runtime but are **not portable** across different NPU driver versions.

---

## 4. Direct Framework → RKNN

rknn-toolkit2 also supports direct conversion without the ONNX intermediate:

```python
# PyTorch
rknn.load_pytorch(model="model.pt", input_size_list=[[1, 3, 224, 224]])

# TensorFlow
rknn.load_tensorflow(tf_pb="model.pb", inputs=["input"], outputs=["output"], input_size_list=[[1, 224, 224, 3]])

# TFLite
rknn.load_tflite(model="model.tflite")

# Caffe
rknn.load_caffe(model="model.prototxt", blobs="model.caffemodel")

# Darknet
rknn.load_darknet(model="model.cfg", weight="model.weights")
```

Direct conversion is simpler but the ONNX intermediate is recommended for:
- Debugging (check model structure at each stage).
- Compatibility (ONNX is the widest-supported intermediate format).
- Re-targeting (same ONNX for Ascend, GPU, etc.).

## NPU-Aware Model Optimization

### Problem: NPU fallback ops kill performance

Not every operator in a deep learning model can run on the NPU. When an unsupported op
is encountered at runtime, the RKNN driver falls back to **CPU execution** for that op.
This requires **NPU→CPU→NPU data transfer** — often more expensive than running the
entire pipeline on CPU.

```
Model graph with CPU-fallback ops:

  [NPU conv] → [NPU relu] → [NPU→CPU] → [CPU: NMS] → [CPU→NPU] → [NPU ...]
                                    ^^^               ^^^
                               expensive copy!   expensive copy!
```

### Two-phase optimization strategy

| Phase | What | Goal |
|---|---|---|
| **Phase 1: Ship** | Export full model → ONNX → RKNN. Run everything on NPU (including fallback ops). | Get it working fast. Measure baseline. |
| **Phase 2: Optimize** | Remove CPU-fallback ops from the graph. Implement them in C++ on CPU side. | Avoid NPU↔CPU data shuttle. Maximize NPU-only execution. |

### Step 1 — Identify which ops run on NPU vs CPU

**Method A: RKNN verbose build output**
```python
rknn = RKNN(verbose=True)
ret = rknn.load_onnx(model="model.onnx")
ret = rknn.build(do_quantization=True, dataset="./dataset.txt")
```
Search the output for:
- `running on NPU` — ops accelerated by NPU
- `running on CPU` — ops that fall back to CPU (these are candidates for removal)

**Method B: Runtime performance query**
```c
rknn_perf_detail perf;
rknn_query(ctx, RKNN_QUERY_PERF_DETAIL, &perf, sizeof(perf));
// Check perf.layer_detail for per-op timing and execution target
```

### Step 2 — Identify candidate ops for removal

Typical ops that RKNN NPU cannot accelerate (vary by SoC and RKNN version):

| Op | Typical location | CPU/NPU | C++ alternative |
|---|---|---|---|
| `NonMaxSuppression` | Detection output | ❌ CPU | Implement NMS in C++ after NPU output |
| `TopK` | Classification | ❌ CPU | Sort + select in C++ |
| `Sort` / `ArgSort` | Post-processing | ❌ CPU | Implement in C++ |
| `Gather` / `Scatter` (dynamic indices) | Various | ⚠️ Often CPU | Rewrite with fixed indices or C++ |
| `Where` / `NonZero` | Mask processing | ❌ CPU | Implement in C++ |
| `Reshape` (certain patterns) | Shape changes | ⚠️ May fall back | Check; often free if contiguous |
| `Expand` / `Tile` (large factors) | Data duplication | ⚠️ Sometimes CPU | Implement in C++ |
| Custom ONNX ops | Any | ❌ Always CPU | Implement in C++ or replace with supported ops |

> ⚠️ This list changes with RKNN version and target SoC.
> **Always verify** by inspecting the verbose build output, not by assumption.

### Step 3 — Split the model (remove layers from ONNX)

```python
import onnx
from onnx import optimizer

model = onnx.load("model.onnx")
graph = model.graph

# Find the node just before NMS
nms_node = None
output_name = None
for node in graph.node:
    if node.op_type == "NonMaxSuppression":
        nms_node = node
        # The input to NMS is the detection output we want to keep
        output_name = node.input[0]  # e.g., "detection_output"
        break

# Remove NMS and all nodes after it
# Keep only nodes up to output_name
onnx.utils.extract_model(
    "model_trimmed.onnx",
    "model.onnx",
    input_names=["input"],  # original model inputs
    output_names=[output_name]  # output before NMS
)
```

After trimming, reconvert to RKNN:
```python
rknn.load_onnx(model="model_trimmed.onnx")
rknn.build(do_quantization=True, dataset="./dataset.txt")
rknn.export_rknn("./model_trimmed.rknn")
```

### Step 4 — Implement removed layers in C++

```c
// After rknn_run, the NPU output is the raw detection tensor
// (without NMS). Implement NMS in C++ on the CPU side.

void rknn_run_and_postprocess(rknn_context ctx) {
    rknn_output outputs[1];
    outputs[0].want_float = 0;  // keep INT8
    rknn_outputs_get(ctx, 1, outputs, NULL);

    // Manual dequantize (only for the boxes we need to sort)
    float scale = output_attr.scale;
    uint8_t zp = output_attr.zp;

    // Custom C++ NMS — avoids NPU→CPU shuttle for the whole graph
    std::vector<Detection> detections = decode_outputs(outputs[0].buf, scale, zp);
    std::vector<Detection> nms_results = custom_nms(detections, 0.45f, 0.5f);

    rknn_outputs_release(ctx, 1, outputs);
}
```

### Step 5 — Measure the gain

```c
struct timespec start, end;
clock_gettime(CLOCK_MONOTONIC, &start);

// Full-pipeline benchmark
for (int i = 0; i < 100; i++) {
    preprocess();
    rknn_run(ctx, NULL);
    postprocess();
}

clock_gettime(CLOCK_MONOTONIC, &end);
double ms = (end.tv_sec - start.tv_sec) * 1000.0 +
            (end.tv_nsec - start.tv_nsec) / 1e6;
printf("Avg: %.2f ms/frame\n", ms / 100);
```

Compare:
- **Before**: full model on NPU (with CPU fallback ops)
- **After**: trimmed model on NPU + C++ post-process on CPU

Expected outcome for a good candidate (e.g., YOLO + NMS):
- Full model on NPU: ~50 ms/frame (including NPU↔CPU shuttle for NMS)
- Trimmed model + C++ NMS: ~15 ms/frame (no shuttle, most time on NPU)

### Design checklist for model optimization

- [ ] Phase 1 benchmark captured (full model, baseline)
- [ ] Verbose build output inspected — all CPU-fallback ops identified
- [ ] Each CPU-fallback op classified: removable or not
- [ ] ONNX trimmed (remove CPU-fallback layers)
- [ ] C++ replacement implemented for each removed op
- [ ] Accuracy validated (compare trimmed+CPU vs full model output)
- [ ] Phase 2 benchmark captured (trimmed model + C++ postproc)
- [ ] Speedup confirmed before switching to production

## Verification Checklist

- [ ] Model loads and runs in rknn-toolkit2 simulator.
- [ ] INT8 accuracy acceptable (compare against FP32 baseline).
- [ ] Target platform matches actual SoC (rk3568 vs rk3576).
- [ ] Input shape matches runtime data.
- [ ] Quantization dataset representative of production data.
- [ ] Pre-compiled model (if used) matches NPU driver version on board.
- [ ] Batch size matches deployment requirement.
- [ ] CPU-fallback ops identified and evaluated for removal.
