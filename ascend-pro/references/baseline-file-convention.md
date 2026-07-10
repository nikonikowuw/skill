# Baseline File Convention

## Purpose

Use a stable in-repo location for the reviewed Ascend baseline so future turns can treat it as project context instead of reconstructing from chat history. The file must preserve device-scoped sections when the project supports multiple devices.

## Context File Structure

Each device context is stored as a separate file keyed by the machine ID:

```
.agent/
  ascend-pro/
    context/
      {machine_id_A}.md     # Machine A's context
      {machine_id_B}.md     # Machine B's context (different hardware)
```

`.agent/ascend-pro/context/{machine_id}.md` is a **combined context file** that includes:
- Context metadata (context ID, machine ID, device model, deployment type, version info)
- Device baseline (hardware, CANN, .so, symbols, model artifacts)
- Key API signatures relevant to the project
- Memory alignment rules (stride, buffer size formulas)
- AIPP configuration and model conversion notes
- Device-scoped runtime contexts for multi-device projects
- Verification history and open risks
- Pointers to official docs via `ctx_search`

## Rationale

- **Filename = machine_id**: the machine ID (`/etc/machine-id`) uniquely identifies a machine. Looking up context by machine ID is deterministic and eliminates the need for an extra comparison step.
- **Multi-device native**: different machines (host vs container, 310P vs 200I A2) each get their own file naturally.
- **Self-validating**: if the file exists, it IS the context for that machine — no need to store-and-compare machine_id inside the file.
- **Reliable**: `/etc/machine-id` is available on every Linux system regardless of driver version or container permissions.

## Rule

Do not treat the generated file as final until it has passed the review steps in [baseline-review-checklist.md](baseline-review-checklist.md).

## Agent Workflow

1. Get device machine ID from user — run `cat /etc/machine-id`. This is used as the context filename key.
2. Check if `.agent/ascend-pro/context/{machine_id}.md` already exists.
   - If yes, read it and proceed (filename guarantees match).
   - If no, generate one (see below).
3. Generate a baseline draft from pasted device evidence using `scripts/render-project-baseline.py`.
4. Append API context (key signatures, alignment rules, active AIPP configs, ctx_search pointers).
5. Review and correct the combined context manually.
6. Write the final context to `.agent/ascend-pro/context/{machine_id}.md` (see [context.md format](../SKILL.md#contextmd-document-format)).
7. Read this file first in every future session before making code changes.

## Script Support

`scripts/render-project-baseline.py` supports:

- `--write-default`
  - **Recommended.** Auto-detects `machine_id` from `/etc/machine-id` (or npu-smi fallback) and writes to `.agent/ascend-pro/context/{machine_id}.md`. Creates the directory if needed.
- `-o .agent/ascend-pro/context/{machine_id}.md`
  - Writes to the standard path (substitute actual machine_id).
- `-o <path>`
  - Writes to an explicit path.

## Suggested Usage

From the project root (recommended — auto-detects machine_id):

```bash
python3 /path/to/render-project-baseline.py pasted-device-evidence.txt --write-default
```

Or with pasted stdin:

```bash
cat pasted-device-evidence.txt | python3 /path/to/render-project-baseline.py --write-default
```

Or specify the path explicitly:

```bash
mkdir -p .agent/ascend-pro/context
python3 /path/to/render-project-baseline.py pasted-device-evidence.txt -o .agent/ascend-pro/context/abc123def4567890abc123def4567890.md
```

After generating the baseline portion, manually append API context and other sections following the [context.md format](../SKILL.md#contextmd-document-format) guide in SKILL.md.
