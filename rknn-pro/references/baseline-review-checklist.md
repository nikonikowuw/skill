# Baseline Review Checklist

## Purpose

Use this checklist after generating a project baseline from pasted device evidence. The generated baseline is a draft, not final truth.

## Rule

Do not treat parser output as authoritative until the agent has reviewed and corrected it.

## Mandatory Review Pass

After generating the baseline, the agent must manually verify:

1. `Board identity`
   - Is the board model really the board, not a kernel line or unrelated text
   - Is the SoC identification unambiguous
   - If multiple boards, SoCs, BSP images, or containers appear, is each one split into its own device context
   - Is the active device context named before any implementation recommendation

2. `Kernel and BSP interpretation`
   - Is the kernel string correctly captured
   - Is BSP or image provenance still unknown and worth asking about

3. `Driver interpretation`
   - Is the RGA driver version actually present, or did the parser infer too much
   - Are missing device nodes meaningful, or simply unavailable in the pasted excerpt

4. `Userspace library interpretation`
   - Are the listed `so` files real deployment candidates
   - Is there evidence of multiple conflicting copies
   - Is the project's actual runtime copy still unknown
   - Are `.so`, header, driver, symbol, and RKNN artifact facts kept separate per board context

5. `ABI and symbol interpretation`
   - Are the required symbols really present in the correct library
   - Are absent symbols caused by incomplete paste rather than real ABI gaps

6. `Project integration interpretation`
   - Do include and link paths actually reflect the build
   - Is there evidence of `dlopen`, sysroot, rpath, or bundled SDK usage that the parser missed

7. `Open risks`
   - Did the parser miss important unknowns
   - Are any generated “no obvious gaps” statements too strong for the actual evidence quality

## Required Agent Output

After review, the agent should produce:

- `Reviewed baseline`
- `Corrections made to parser draft`
- `Still unknown`
- `Questions to ask the user before implementation`

## Escalation Rule

If the baseline still leaves uncertainty about:

- active driver stack
- actual runtime library copy
- exported symbols required by the project
- board BSP provenance
- active board context when multiple contexts exist

then the agent should stop short of implementation and ask for more evidence.
