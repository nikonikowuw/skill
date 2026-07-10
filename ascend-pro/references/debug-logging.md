# Debugging and Logging for Ascend Pipelines

Ascend pipelines run on remote NPU hardware with limited visibility. **Every pipeline must embed
controllable debug logging from day one** — retrofitting logs after a bug surfaces is slow and unreliable.

## Table of Contents

- [Logging Control Strategy](#logging-control-strategy)
  - [Multi-Configuration Setup (C++, spdlog)](#multi-configuration-setup-c-spdlog)
  - [Python Pattern](#python-pattern)
- [Mandatory Logging Points](#mandatory-logging-points)
- [AscendCL Error Wrapping](#ascendcl-error-wrapping)
- [ATC / Model Conversion Logging](#atc--model-conversion-logging)
- [Performance-Sensitive Logging](#performance-sensitive-logging)
  - [Throttled Debug Logging](#throttled-debug-logging)
  - [Performance Logger Usage](#performance-logger-usage)
  - [Analyzing Perf CSV](#analyzing-perf-csv)
- [Debug Checklist](#debug-checklist)

---

## Logging Control Strategy

Use two independent log configurations to serve different development needs:

| Logger | Purpose | Output | Format | Control |
|---|---|---|---|---|
| **`ascend`** (debug) | Find bugs — trace every stage's detail, data values, error context | stderr, human-readable | `[HH:MM:SS] [LEVEL] [ASCEND] message` | `ASCEND_LOG_LEVEL` env var + compile-time strip |
| **`ascend_perf`** (performance) | Find bottlenecks — record each stage's latency, frame stats | `ascend_perf.csv`, machine-parseable | `stage,elapsed_us,frame,extra` (CSV) | `ASCEND_PERF=1` env var (on/off) |

Use spdlog for both — it handles multiple named loggers with independent levels, sinks, and patterns.

### Multi-Configuration Setup (C++, spdlog)

#### Combined setup (once at startup)

```cpp
#include <spdlog/spdlog.h>
#include <spdlog/sinks/stdout_color_sinks.h>
#include <spdlog/sinks/basic_file_sink.h>

void setup_ascend_loggers() {
    // ── Configuration 1: Debug Logger ──────────────────────────
    // Human-readable, stderr, color, level-controlled
    auto debug_log = spdlog::stdout_color_mt_st("ascend");
    debug_log->set_pattern("[%H:%M:%S] [%^%l%$] [ASCEND] %v");

    const char* debug_env = getenv("ASCEND_LOG_LEVEL");
    if (debug_env) {
        if      (strcmp(debug_env, "TRACE") == 0) debug_log->set_level(spdlog::level::trace);
        else if (strcmp(debug_env, "DEBUG") == 0) debug_log->set_level(spdlog::level::debug);
        else                                       debug_log->set_level(spdlog::level::info);
    } else {
        debug_log->set_level(spdlog::level::info);
    }

    // ── Configuration 2: Performance Logger ────────────────────
    // Machine-parseable CSV, file output, enabled by ASCEND_PERF=1
    const char* perf_env = getenv("ASCEND_PERF");
    if (perf_env && strcmp(perf_env, "1") == 0) {
        auto perf_log = spdlog::basic_logger_mt("ascend_perf", "ascend_perf.csv", true);
        perf_log->set_pattern("%v");  // raw message, no prefix — we write CSV directly
        perf_log->set_level(spdlog::level::info);
        perf_log->info("stage,elapsed_us,frame,extra");  // CSV header
    }
}
```

#### Usage

```cpp
auto log   = spdlog::get("ascend");       // debug logger (may be null in production)
auto plog  = spdlog::get("ascend_perf");  // perf logger (null unless ASCEND_PERF=1)

// Debug logger — detailed, human-readable
log->info("aclrtSetDevice({}) initialized", deviceId);
log->debug("buffer allocated: {} size={}", (void*)devPtr, size);
log->error("aclmdlLoadFromFile({}) failed: {}", path, aclGetRecentErrDesc());

// Performance logger — CSV row, only if enabled
if (plog) {
    plog->info("model_load,{},-,", elapsed_us);
}
```

#### Compile-time stripping (debug logger only)

Define `SPDLOG_ACTIVE_LEVEL` before including spdlog to strip lower levels at compile time:

```cpp
// Production: strip trace, debug, info, warn — only errors/warnings remain
#define SPDLOG_ACTIVE_LEVEL SPDLOG_LEVEL_WARN
#include <spdlog/spdlog.h>

// Development: keep everything
#define SPDLOG_ACTIVE_LEVEL SPDLOG_LEVEL_TRACE
#include <spdlog/spdlog.h>
```

**Production build**:    `-DSPDLOG_ACTIVE_LEVEL=SPDLOG_LEVEL_WARN`
**Development build**:   `-DSPDLOG_ACTIVE_LEVEL=SPDLOG_LEVEL_TRACE`
**Runtime debug toggle**: `ASCEND_LOG_LEVEL=DEBUG ./pipeline` (stderr, verbose)
**Runtime perf toggle**:   `ASCEND_PERF=1 ./pipeline` (generates `ascend_perf.csv`)

### Python Pattern

```python
import os
import logging

# Single logger for the Ascend pipeline; level controlled by env var
log = logging.getLogger("ascend")
_log_level = os.environ.get("ASCEND_LOG_LEVEL", "INFO").upper()
log.setLevel(getattr(logging, _log_level, logging.INFO))
handler = logging.StreamHandler()
handler.setFormatter(logging.Formatter("%(asctime)s [ASCEND %(levelname)s] %(message)s"))
log.addHandler(handler)
```

Usage: `ASCEND_LOG_LEVEL=DEBUG python3 pipeline.py` to enable verbose logging.

---

## Mandatory Logging Points

These locations **must** have explicit logs in any Ascend pipeline. Logging at these points
is not optional for development.

| Stage | What to Log | Debug Level | Perf? |
|---|---|---|---|
| **Device init** | `aclrtSetDevice(deviceId)` result, device count, stream creation | INFO | Yes |
| **Model loading** | `aclmdlLoadFromFile(path)` → modelId; model input/output tensor dims, data type, size per tensor | INFO | Yes |
| **Memory allocation** | `aclrtMalloc(devPtr, size, policy)` → pointer, size (if aligned); buffer pool count | DEBUG | — |
| **Dataset creation** | `aclmdlCreateDataset` → buffer handle, buffer size, data type for each input/output | DEBUG | — |
| **DVPP channel** | `acldvppCreateChannel(channelId)` → resolution, format, alignment check | INFO | Yes |
| **DVPP operation** | `acldvppVpcResizeAsync` / `acldvppJpegDecodeAsync` → input size → output size, format | DEBUG | Yes |
| **AIPP config** | AIPP mode (static/dynamic), input format, CSC matrix, mean/std values | INFO | — |
| **Preprocess (CPU)** | resize/crop/normalize → input tensor layout, data range | DEBUG | Yes |
| **Inference submit** | `aclmdlExecuteAsync(modelId, input, output, stream)` → timestamp before submit | DEBUG | — |
| **Inference complete** | `aclrtSynchronizeStream(stream)` → elapsed time since submit | INFO | **Yes** |
| **Output retrieval** | `aclrtMemcpy(host, device, size, direction)` → tensor index, size, first/last hex values | DEBUG | Yes |
| **Postprocess (CPU)** | decode output, NMS, draw boxes → result summary | DEBUG | Yes |
| **Error path** | Every `aclError` != `ACL_SUCCESS` → file, line, error code description, variable values | ERROR | — |

> **Perf?** = also log timing to `ascend_perf` CSV when `ASCEND_PERF=1`. This gives you a complete latency breakdown per pipeline stage.

Additionally, log the **data flow at each format transition** — use the debug logger, INFO level:

```
[ASCEND 10:15:30] INPUT:  JPEG 1920x1080
[ASCEND 10:15:31] VDEC:   YUV420SP_NV12 1920x1080 stride=1920
[ASCEND 10:15:32] VPC:    YUV420SP_NV12 512x512 stride=512
[ASCEND 10:15:33] MODEL:  RGB FP32 1x3x512x512 mean=[128,128,128] std=[0.0039,0.0039,0.0039]
[ASCEND 10:15:34] INFER:  model=0 elapsed=12ms
[ASCEND 10:15:35] OUTPUT: tensor_0 float32[1,84] values=[0.02, 0.95, ...]
```

---

## AscendCL Error Wrapping

Every AscendCL call returns `aclError`. **Never ignore the return value.** Use a wrapper that
logs on failure:

```c
#define ACL_CHECK(call)                                                         \
    do {                                                                        \
        aclError _rc = (call);                                                  \
        if (_rc != ACL_SUCCESS) {                                               \
            SPDLOG_LOGGER_ERROR(spdlog::get("ascend"),                         \
                "{} failed at {}:{} — {}",                                     \
                #call, __FILE__, __LINE__, aclGetRecentErrDesc());              \
            return _rc;                                                         \
        }                                                                       \
    } while (0)

// Usage — logs "aclrtSetDevice(0) failed at pipeline.c:42 — ..." on error
ACL_CHECK(aclrtSetDevice(0));
```

> ⚠️ `aclGetRecentErrDesc()` only works after the error is set (immediately after the failing call).
> Do not make intervening ACL calls before checking.

---

## ATC / Model Conversion Logging

Save the full ATC conversion log — not just the final result — so conversion errors are reproducible:

```bash
atc --model=model.onnx --framework=5 --output=model --soc_version=Ascend310P3 \
    --log=debug 2>&1 | tee atc-conversion-$(date +%Y%m%d-%H%M%S).log
```

Keep the conversion log alongside the `.om` file as a reference for future debugging.

---

## Performance-Sensitive Logging

### Throttled Debug Logging

For real-time pipelines (video, camera), logging every frame at DEBUG level may cause frame drops.
Use **throttled logging** with the debug logger:

```cpp
auto log = spdlog::get("ascend");

#define ASCEND_LOG_EVERY_N(logger, n, ...)                                     \
    do {                                                                       \
        static int _log_counter_##__LINE__ = 0;                                \
        if (++_log_counter_##__LINE__ >= (n)) {                                \
            _log_counter_##__LINE__ = 0;                                       \
            SPDLOG_LOGGER_INFO(logger, "[every " #n "] " __VA_ARGS__);         \
        }                                                                      \
    } while (0)

ASCEND_LOG_EVERY_N(log, 100, "frame processed, queue depth={}", depth);
```

For aggregated stage timing via the debug logger:

```cpp
static uint64_t g_infer_sum = 0;
static int g_infer_count = 0;

// After stream sync:
g_infer_sum += elapsed_us;
g_infer_count++;
if (g_infer_count >= 100) {
    log->info("infer avg: {} us over {} frames",
              g_infer_sum / g_infer_count, g_infer_count);
    g_infer_sum = 0;
    g_infer_count = 0;
}
```

### Performance Logger Usage

When `ASCEND_PERF=1`, write one CSV row per stage measurement. This gives a per-frame latency
breakdown that can be analyzed offline:

```cpp
auto plog = spdlog::get("ascend_perf");

void log_stage_timing(const char* stage, uint64_t elapsed_us, int frame_id, const char* extra) {
    if (plog) {
        plog->info("{},{},{},{}", stage, elapsed_us, frame_id, extra ? extra : "-");
    }
}

// Usage — drop these at every stage boundary in your pipeline loop
uint64_t t_start = get_time_us();

// ... VPC resize ...
log_stage_timing("vpc_resize", get_time_us() - t_start, frame_id, "1920x1080->512x512");
t_start = get_time_us();

// ... model inference ...
( void )aclrtSynchronizeStream(stream);
log_stage_timing("infer", get_time_us() - t_start, frame_id, nullptr);
t_start = get_time_us();

// ... postprocess ...
log_stage_timing("postprocess", get_time_us() - t_start, frame_id, "detect");
```

### Analyzing Perf CSV

After the pipeline runs with `ASCEND_PERF=1`, the `ascend_perf.csv` file can be analyzed with
a short Python script:

```python
import pandas as pd

df = pd.read_csv("ascend_perf.csv")

# Per-stage latency summary (all frames)
summary = df.groupby("stage")["elapsed_us"].agg(["count", "mean", "std", "min", "max"])
summary["mean_ms"] = summary["mean"] / 1000
summary["max_ms"]  = summary["max"] / 1000
print(summary.to_string())

# Per-frame total latency
per_frame = df.groupby("frame")["elapsed_us"].sum() / 1000
print(f"\nFrame total: mean={per_frame.mean():.1f}ms max={per_frame.max():.1f}ms")

# Find the slowest frame
worst = df[df["frame"] == per_frame.idxmax()]
print(f"\nSlowest frame #{per_frame.idxmax()}:\n{worst.to_string()}")
```

Example output:
```
              count     mean           std    min     max  mean_ms  max_ms
stage
decode          300    512.3         98.2    401    1202     0.51    1.20
vpc_resize      300    182.5         22.1    150     298     0.18    0.30
infer           300   8120.4        312.5   7650    9850     8.12    9.85
postprocess     300   1250.8        180.2   1000    2100     1.25    2.10

Frame total: mean=10.07ms max=13.25ms
```

This data is essential for:
- **Bottleneck identification**: which stage dominates? (here: `infer` at ~8ms)
- **Latency regression detection**: compare CSV between builds
- **Frame drop analysis**: which stage spikes on the slowest frame?

---

## Debug Checklist

- [ ] **Two loggers configured**: `ascend` (debug) and `ascend_perf` (performance CSV)
- [ ] **Debug logger**: every ACL API call is wrapped with `ACL_CHECK` — no exceptions
- [ ] **Debug logger**: device init, model load, stream sync, DVPP channel log at INFO level
- [ ] **Debug logger**: memory allocation, dataset creation, DVPP operations log at DEBUG level
- [ ] **Debug logger**: each data-format transition is logged (pixel format, resolution, stride, tensor layout)
- [ ] **Debug logger**: error paths capture both `aclError` code and descriptive context
- [ ] **Debug logger**: throttled logging (`ASCEND_LOG_EVERY_N`) used for per-frame verbose output
- [ ] **Debug logger**: production build can strip lower levels via `SPDLOG_ACTIVE_LEVEL`
- [ ] **Performance logger**: each stage marked "Perf? = Yes" in the mandatory table logs timing to CSV
- [ ] **Performance logger**: `ASCEND_PERF=1` enables it without recompiling; disabled by default
- [ ] **Performance logger**: per-frame total latency can be derived from the CSV
- [ ] **ATC conversion log** is saved alongside the `.om` file
