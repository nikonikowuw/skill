# RK3576 and RK3568 Matrix

## Scope

This skill is scoped to:

- `RK3576 + Linux`
- `RK3568 + Linux`

The skill assumes native userspace work in C or C++, with Python limited to model conversion, environment checks, and benchmark support.

## Verified Platform Facts

- `RKNN-Toolkit2` states that its supported platforms include `RK3576 Series` and `RK3566/RK3568 Series`.
- The older standalone `rknpu2` repository is marked as no longer maintained and points developers to `rknn-toolkit2/tree/master/rknpu2`.
- Rockchip MPP documentation in the `rockchip-linux/mpp` repository states that MPP supports `RK3566/RK3568` among supported chipsets.
- The same MPP documentation describes `MppBuffer` as encapsulating buffer implementations including Linux `dma-buf`.

## Source Confidence

Use this confidence scale when reading the matrix below:

- `Primary`: Linux kernel docs or Rockchip-maintained repositories
- `Secondary`: credible third-party reporting that cites Rockchip material, but is not itself the vendor source

For `RK3576`, public primary-source material is still thinner than for RK3568, so some SoC-capability rows below are intentionally marked `Secondary`.

## Capability Snapshot

| Area | RK3568 | RK3576 | Confidence |
| --- | --- | --- | --- |
| RKNN Toolkit2 platform support | Listed as `RK3566/RK3568 Series` | Listed as `RK3576 Series` | Primary |
| Linux media path used by this skill | `V4L2 + MPP + RGA + RKNN Runtime` | `V4L2 + RGA + RKNN Runtime`, with media and BSP details to verify per board | Primary for stack shape, Secondary for RK3576 BSP maturity |
| CPU class | Cortex-A55 generation device family | 4x Cortex-A72 + 4x Cortex-A53 | Secondary |
| NPU headline | Board-specific docs still needed for exact runtime limits; toolkit support is confirmed | 6 TOPS headline repeatedly reported from Rockchip presentation material | Secondary |
| Video decode headline | MPP support for RK3568 family is documented | 4K120 H.265/H.264/AV1/VP9/AVS2 reported from Rockchip presentation material | Secondary |
| Video encode headline | Common RK3568 material typically centers on 1080p60 encode paths, but confirm from board BSP | 4K30 H.264/H.265 and MPEG reported from Rockchip presentation material | Secondary |
| Memory types highlighted in public material | DDR3L, LPDDR3, DDR4, LPDDR4 and LPDDR4X are commonly associated with RK3568-class boards; verify exact board wiring | LPDDR4, LPDDR4x, LPDDR5 reported from Rockchip presentation material | Secondary |
| Linux SDK trajectory | RK3568 was included in Rockchip's Linux 6.1 SDK and Debian 12 roadmap reported on November 2, 2023 | RK3576 appeared in the same roadmap coverage as a new SoC, but the published Linux 6.1 schedule excerpt explicitly named RK3568, not RK3576 | Secondary |

## Practical Matrix For Skill Behavior

| Topic | RK3568 default assumption | RK3576 default assumption |
| --- | --- | --- |
| BSP maturity | Expect older and broader field usage. More legacy examples may exist, but they may still be tied to older kernels or vendor drops. | Expect less stable public guidance and more board-specific variance. Demand stronger board inspection before assuming feature parity. |
| Zero-copy decode path | MPP decode plus external-buffer mode is a realistic first hypothesis. Audit whether the project actually uses it. | Treat zero-copy decode feasibility as board-SDK dependent until the media stack on that board is confirmed. |
| NPU deployment | Toolkit and runtime support are confirmed at the platform level, but runtime feature level still needs board verification. | Same rule, but be stricter about runtime feature verification because public examples are newer and thinner. |
| RGA usage | Reasonable to expect `dma_fd`-based RGA paths on vendor SDKs. Still audit driver and `librga` match. | Same, but do not infer exact supported combinations from RK3568 experience alone. |
| Performance tuning bias | Expect CPU postprocess, buffer-mode choice, and hidden copies to dominate more often than raw NPU limits. | Expect board-image maturity and version mismatch risk to be a larger share of failures. |

## Recommended Software Stack Shape

Treat the stack as these layers:

- Kernel and board BSP
- Media stack: `V4L2`, `MPP`, `DRM`, vendor display stack
- Image preprocess stack: `librga`
- Inference stack: `RKNN-Toolkit2` for model conversion, `RKNN Runtime` for C or C++ deployment

Do not hard-code exact package versions into project changes unless they are confirmed from the target board. Rockchip board images and BSP drops often pin userspace libraries and drivers together.

## What To Confirm On The Board

Confirm these on every target board before making strong claims:

- Kernel version and board BSP origin
- `librga` version
- RGA driver version from `/sys/kernel/debug/rkrga/driver_version` or `/proc/rkrga/driver_version` when present
- Whether the media path uses mainline-style V4L2 nodes, vendor MPP wrappers, or both
- Installed RKNN runtime package version and headers
- Whether the repository builds against vendor SDK libraries, locally installed shared objects, or containerized toolchains

## Known Version-Sensitivity Areas

### RGA

The Rockchip RGA FAQ documents that:

- `librga` and the kernel driver have version correspondence requirements.
- If userspace is updated separately from the driver, compatibility mode or outright parameter failures may occur.
- Debug nodes may appear under `/sys/kernel/debug/rkrga` or `/proc/rkrga`, depending on kernel configuration and driver generation.

### RKNN

The Rockchip RKNN material documents that:

- `RKNN-Toolkit2` is the model-conversion tool used on the PC side.
- Runtime deployment on the board uses `RKNN Runtime` for C or C++ or `Toolkit-Lite2` for Python.
- The supported platform list includes both RK3576 and RK3568-family devices, but operator support and runtime features vary by release.

Do not infer exact operator support from platform support alone. Check the runtime release notes or the board SDK when the model is nontrivial.

### Linux SDK and BSP Direction

As a secondary source, CNX Software reported on November 2, 2023 that Rockchip planned Linux 6.1 SDK or BSP releases with Debian 12 support for RK3568 and several other SoCs between Q4 2023 and Q3 2024, while RK3576 appeared in the same roadmap discussion as a new IoT processor. Use that only as a roadmap clue, not as proof of what any given board image actually ships.

### MPP

The MPP documentation describes three decoder memory modes:

- Pure internal mode
- Half internal mode
- Pure external mode

It explicitly notes that pure external mode is the most efficient path for zero-copy display style workflows, but harder to use correctly.

## Practical Default Assumptions

Use these defaults unless the board proves otherwise:

- `DMA-BUF` is the preferred interchange object between subsystems.
- `RGA` is the right place for crop, resize, rotate, and colorspace conversion when the next stage cannot directly consume the source layout.
- `RKNN Runtime` should be treated as the deployment boundary, not the model-authoring boundary.
- Any unexplained CPU spike in a “zero-copy” path deserves suspicion of virtual-address fallback, cache sync overhead, or forced format conversion.

## What To Do Differently Per SoC

### On RK3568

- Start by assuming the required building blocks exist and focus quickly on buffer ownership, MPP memory mode, and RGA import or alignment problems.
- Expect legacy codebases to carry older BSP assumptions. Audit versions before trusting existing examples.

### On RK3576

- Start by confirming the actual board image, kernel base, and installed Rockchip runtime packages before modeling the optimization plan.
- Treat SoC capability headlines as useful but insufficient. Require board evidence for the exact media and inference path you intend to optimize.

## Sources

- Linux kernel V4L2 DMA-BUF importer API: https://docs.kernel.org/userspace-api/media/v4l/dmabuf.html
- Linux kernel dma-buf overview: https://docs.kernel.org/driver-api/dma-buf.html
- Rockchip RKNN Toolkit2 README: https://github.com/airockchip/rknn-toolkit2
- Rockchip legacy rknpu2 README: https://github.com/airockchip/rknpu2
- Rockchip MPP readme: https://github.com/rockchip-linux/mpp/blob/develop/readme.txt
- Rockchip librga FAQ: https://github.com/airockchip/librga/blob/master/docs/Rockchip_FAQ_RGA_EN.md
- CNX Software, November 2, 2023, Rockchip roadmap and Linux 6.1 SDK coverage: https://www.cnx-software.com/2023/11/02/rockchip-roadmap-reveals-rk3576-and-rk3506-iot-processors-linux-6-1-sdk/
