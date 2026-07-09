# Device Evidence Workflow

## Purpose

Use this workflow when the device is in the user's hands and the agent cannot inspect it directly. The user collects evidence on the board, pastes it into the chat, and the agent turns it into authoritative project context scoped to a specific board, SoC, BSP, and runtime library set.

## Rule

Do not treat guessed board details as development truth when user-provided device evidence exists.

If evidence covers multiple boards, SoCs, rootfs images, or containers, do not merge the outputs. Require one labeled evidence block per device context:

```text
== Device Context: RK3568 EVB1 Debian video service ==
<command outputs>

== Device Context: RK3576 vendor BSP camera pipeline ==
<command outputs>
```

## What The User Must Provide

Ask the user to paste:

- Output from `scripts/detect-rockchip-env.sh`
- Output directory contents or key excerpts from `scripts/collect-rockchip-debug.sh`
- `ldd` or `readelf -d` output for the target binary or shared object
- `nm -D` or `readelf -Ws` output for the Rockchip shared objects that matter
- Build-system snippets showing include paths, library paths, and any bundled vendor SDK directories

If the full output is too large, ask for the sections that determine:

- Board and kernel identity
- RGA driver version
- `librga`, `librknnrt`, and `libmpp` locations
- Exported symbols needed by the current integration
- Which board, SoC, BSP, rootfs, or container each output block belongs to

## What The Agent Must Do

After the user pastes device evidence, convert it into a compact baseline with these sections:

1. `Active device context`
2. `Board baseline`
3. `Kernel and BSP baseline`
4. `Driver baseline`
5. `Userspace library sightings`
6. `ABI and symbol baseline`
7. `Project link and include baseline`
8. `Device-scoped runtime contexts`
9. `Open risks`

Keep the baseline short, but concrete. It should be usable as a standing context block for future turns.

Before treating the baseline as authoritative, review it with [baseline-review-checklist.md](/Users/niko/.codex/skills/rockchip-performance/references/baseline-review-checklist.md).

## Baseline Template

Use this structure when summarizing the pasted evidence:

```text
Project baseline

Active device context
- Active context ID:
- Rule:

Board baseline
- SoC:
- Board model:
- Compatible string:

Kernel and BSP baseline
- Kernel:
- OS release:
- BSP or image source:

Driver baseline
- RGA driver:
- V4L2 or media nodes:
- DRM or display nodes:
- Other relevant modules:

Userspace library sightings
- librga:
- librknnrt:
- libmpp or librockchip_mpp:
- Which copy the project actually uses:
- Rule:

ABI and symbol baseline
- Required RGA symbols present:
- Required RKNN symbols present:
- Required MPP symbols present:
- Any symbol mismatches:

Project link and include baseline
- Header roots:
- Library roots:
- Bundled vendor SDK paths:
- Runtime loading behavior:

Device-scoped runtime contexts
- One section per labeled board, SoC, BSP, rootfs, or container context

Open risks
- Unknowns that still block strong conclusions
```

## How To Use The Baseline

- Treat the selected device context as the default hardware and runtime context for the project.
- Before implementation, state the active device context ID. If more than one exists and none is selected, ask which board or SoC is targeted.
- Pass only the active device context to future agents by default; mention other contexts separately to avoid accidental `.so`, driver, or RKNN artifact mixing.
- Refer back to it before proposing API calls or optimization plans.
- Update it if the user supplies better evidence later.
- If later requests conflict with the baseline, call out the conflict explicitly instead of silently switching assumptions.
- If the baseline came from parser output, correct it manually before using it as final context.
- After review, store it using [baseline-file-convention.md](/Users/niko/.codex/skills/rockchip-performance/references/baseline-file-convention.md).

## Development Standard

Once the baseline exists, use it as the development standard for:

- API selection
- Header and library assumptions
- Zero-copy feasibility claims
- Performance bottleneck hypotheses
- Build-system changes

If a proposed code change depends on a capability not supported by the baseline, flag it before implementation.
