# Baseline File Convention

## Purpose

Use a stable in-repo location for the reviewed Rockchip baseline so future turns can treat it as project context instead of reconstructing it from chat history. The file must preserve device-scoped sections when the project supports multiple boards, SoCs, BSP images, or containers.

## Recommended Paths

Prefer these locations in order:

1. `.agents/rknn-context.md` (new standard — combined device baseline + API context)
2. `.agent-context/rockchip-baseline.md` (legacy)
3. `docs/rockchip-baseline.md` (fallback)

Use the first path when the repository keeps agent-only working context. Use the second when project documentation lives under `docs/`.

## Rule

Do not treat the generated file as final until it has passed the review steps in [baseline-review-checklist.md](/Users/niko/.codex/skills/rockchip-performance/references/baseline-review-checklist.md).

## Agent Workflow

1. Generate a baseline draft from pasted device evidence.
2. Review and correct it manually.
3. Ensure each board, SoC, BSP, rootfs, or container has its own context ID and section.
4. Write the reviewed baseline to one of the recommended paths.
5. Refer back to the active context ID in future implementation work.

## Script Support

`scripts/render-project-baseline.py` supports:

- `--write-default`
  - Writes to `.agent-context/rockchip-baseline.md` if `.agent-context/` exists in the current project
  - Otherwise writes to `docs/rockchip-baseline.md` if `docs/` exists
  - Otherwise creates `.agent-context/rockchip-baseline.md`
- `-o <path>`
  - Writes to an explicit path

## Suggested Usage

From the project root:

```bash
python3 /path/to/render-project-baseline.py pasted-device-evidence.txt --write-default
```

Or with pasted stdin:

```bash
cat pasted-device-evidence.txt | python3 /path/to/render-project-baseline.py --write-default
```
