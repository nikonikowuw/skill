# Device Command Checklist

## Purpose

Use this checklist when the user has access to the Rockchip device and the agent does not. These commands collect the minimum evidence needed before development starts.

If the project supports multiple board models, SoCs, rootfs images, or containers, run the checklist separately for each target and wrap each output with a label:

```text
== Device Context: RK3568 EVB1 Debian video service ==
<outputs>

== Device Context: RK3576 vendor BSP camera pipeline ==
<outputs>
```

Without labels, BSP, driver, `.so`, header, and RKNN artifact facts are easy to mix.

## Instructions To The User

Run the commands below on the target device and paste the outputs back into the chat. If the output is too large, paste the named sections called out under `Minimum required sections`.

## Standard Collection Commands

> ⚠️ **Board serial number is mandatory.** It uniquely identifies a physical Rockchip board.
> Even same-SoC boards with different BSP versions are separate contexts. Always collect it.
> The serial is in `cat /proc/cpuinfo | grep Serial`. Record it as the context key.

### 1. Board, kernel, and OS identity

```bash
uname -a
cat /etc/os-release
cat /sys/firmware/devicetree/base/model
cat /sys/firmware/devicetree/base/compatible
cat /proc/cpuinfo
```

### 2. Rockchip-related drivers and device nodes

```bash
lsmod | grep -Ei 'rockchip|rga|mpp|vcodec|rknpu|iep'
ls -l /dev/media* /dev/video* /dev/rga /dev/dri/renderD* 2>/dev/null
dmesg | grep -Ei 'rockchip|rga|mpp|rknpu|vcodec|iep'
```

### 3. RGA driver and debug nodes

```bash
cat /sys/kernel/debug/rkrga/driver_version 2>/dev/null
cat /proc/rkrga/driver_version 2>/dev/null
cat /sys/kernel/debug/rkrga/debug 2>/dev/null
cat /proc/rkrga/debug 2>/dev/null
```

### 4. Rockchip shared libraries

```bash
find /usr /usr/local -maxdepth 4 \( -name 'librga.so*' -o -name 'librknnrt.so*' -o -name 'librockchip_mpp.so' -o -name 'libmpp.so' \) 2>/dev/null
```

### 5. Binary linkage

Replace `<target-binary-or-so>` with the real executable or shared object from the project.

```bash
ldd <target-binary-or-so>
readelf -d <target-binary-or-so>
```

### 6. Exported symbol checks

Replace `<rockchip-shared-object>` with the actual deployed library path, for example `librga.so`, `librknnrt.so`, or `libmpp.so`.

```bash
nm -D <rockchip-shared-object> | grep -E 'importbuffer_fd|wrapbuffer_fd|imcheck|rga|rknn_|mpi'
readelf -Ws <rockchip-shared-object> | grep -E 'importbuffer_fd|wrapbuffer_fd|imcheck|rga|rknn_|mpi'
strings <rockchip-shared-object> | grep -Ei 'version|rknn|rga_api'
```

### 7. Project build clues

Run this from the project root:

```bash
rg -n 'rknn|rga|im2d|mpp|rk_mpi|find_library|target_link_libraries|include_directories|dlopen' .
```

## Minimum Required Sections

If the full output is too large, paste at least:

- `uname -a`
- `/etc/os-release`
- device tree `model` and `compatible`
- RGA driver version
- the `find` results for `librga`, `librknnrt`, and `libmpp`
- `ldd` and `readelf -d` for the target binary
- one symbol dump for each relevant Rockchip `so`
- the key build-system lines showing include and link paths

## Preferred Shortcut

If the skill scripts are available on the device, prefer:

```bash
scripts/detect-rockchip-env.sh
scripts/collect-rockchip-debug.sh
```

Then paste the resulting output or the relevant excerpts.
