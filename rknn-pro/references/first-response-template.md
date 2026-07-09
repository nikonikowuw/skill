# First Response Template

## Purpose

Use this template when a user asks for Rockchip development or optimization help but the device evidence has not been collected yet.

## Agent Behavior

The first response should:

1. State that implementation should wait until the device baseline is collected
2. Give the user a concrete command checklist
3. Tell the user exactly what to paste back
4. Promise to turn that evidence into a project baseline for subsequent development

## Suggested Response Shape

Use wording close to this:

```text
Before changing code, I need the board and runtime baseline from the target device. On Rockchip work, board model, SoC, BSP, driver, shared-library, header, RKNN artifact, and symbol mismatches are common enough that coding first is risky.

Please run these on the device and paste the outputs back. If you have multiple Rockchip boards, SoCs, rootfs images, or containers, paste one labeled block per target, for example `== Device Context: RK3568 EVB1 Debian ==`.

1. Board and OS identity
   uname -a
   cat /etc/os-release
   cat /sys/firmware/devicetree/base/model
   cat /sys/firmware/devicetree/base/compatible

2. Drivers and nodes
   lsmod | grep -Ei 'rockchip|rga|mpp|vcodec|rknpu|iep'
   ls -l /dev/media* /dev/video* /dev/rga /dev/dri/renderD* 2>/dev/null

3. RGA driver
   cat /sys/kernel/debug/rkrga/driver_version 2>/dev/null
   cat /proc/rkrga/driver_version 2>/dev/null

4. Rockchip libraries
   find /usr /usr/local -maxdepth 4 \( -name 'librga.so*' -o -name 'librknnrt.so*' -o -name 'librockchip_mpp.so' -o -name 'libmpp.so' \) 2>/dev/null

5. Target binary linkage
   ldd <target-binary-or-so>
   readelf -d <target-binary-or-so>

6. Exported symbols
   nm -D <rockchip-shared-object> | grep -E 'importbuffer_fd|wrapbuffer_fd|imcheck|rga|rknn_|mpi'
   readelf -Ws <rockchip-shared-object> | grep -E 'importbuffer_fd|wrapbuffer_fd|imcheck|rga|rknn_|mpi'

7. Build-system clues from the project root
   rg -n 'rknn|rga|im2d|mpp|rk_mpi|find_library|target_link_libraries|include_directories|dlopen' .

After you paste that, I will turn it into a compact device-scoped project baseline, select the active board context with you, and use only that context as the development standard for code changes.
```

## Reference

For the full command set and fallback instructions, use [device-command-checklist.md](/Users/niko/.codex/skills/rockchip-performance/references/device-command-checklist.md).
After generating a baseline draft, review it with [baseline-review-checklist.md](/Users/niko/.codex/skills/rockchip-performance/references/baseline-review-checklist.md).
Store the reviewed version as `.agents/rknn-context.md` (see [baseline-file-convention.md](baseline-file-convention.md)).
