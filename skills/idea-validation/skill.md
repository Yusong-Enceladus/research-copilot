---
name: idea-validation
description: "Validate a research idea BEFORE implementation. Tests core assumptions with minimal experiments. Kills bad ideas in hours, not weeks. MANDATORY before any implementation work begins."
argument-hint: [idea-description]
allowed-tools: Bash(*), Read, Write, Edit, Grep, Glob, WebSearch, WebFetch, Agent, mcp__codex__codex
---

# Idea Validation: Kill Bad Ideas Fast

**This skill runs BEFORE any implementation.** It takes 1-4 hours, not days.
If it fails, the idea is abandoned — no sunk cost fallacy.

## Why This Exists

GraphVLA failure postmortem: 15K lines of code, dozens of HPC jobs,
multiple paper drafts — all built on an untested assumption that turned
out to be false. A 10-minute pilot would have caught it on day 1.

## Input: $ARGUMENTS

## Step 1: Extract Core Assumptions (30 min)

Every research idea rests on assumptions that MUST be true for it to work.
These assumptions are often implicit. Make them explicit.

**Template:**
```markdown
## Idea: [title]
## Core Claim: [what you expect to show]

## Assumptions (ordered by fragility — most fragile first):
1. [assumption that, if false, kills the entire idea]
2. [assumption that, if false, significantly weakens the idea]
3. [assumption that affects scope but not viability]

## Kill Criteria:
- If assumption 1 is false → ABANDON idea
- If assumption 2 is false → PIVOT to [alternative]
- If assumption 3 is false → NARROW scope

## What would falsify this idea in 1 hour?
[describe a quick experiment that would definitively show the idea is wrong]
```

**Example (GraphVLA — what SHOULD have been written):**
```markdown
## Idea: GraphVLA — graph scheduling for VLA long-horizon tasks
## Core Claim: DAG scheduling improves VLA performance on long-horizon tasks

## Assumptions:
1. FRAGILE: Swapping global instructions for subtask instructions won't
   break the VLA's learned behavior (the VLA generalizes across instruction
   distributions)
2. FRAGILE: The VLA has latent subtask capabilities that can be unlocked
   by routing different instructions
3. MODERATE: DAG scheduling provides better ordering than linear execution

## Kill Criteria:
- If swapping instructions HURTS performance → idea is fundamentally broken
- If vanilla VLA already achieves 100% → no room for improvement

## What would falsify this in 1 hour?
Run 5 episodes with Pi0.5: (a) global instruction, (b) subtask instruction.
If (b) < (a), the core mechanism is broken.
```

## Step 2: Test the Most Fragile Assumption FIRST (1-2 hours)

**Do NOT build anything yet.** Use existing tools, existing models, existing
environments. The goal is to test assumption #1 with MINIMAL effort.

Rules:
- Maximum 50 lines of throwaway test code
- Use a pretrained model that's already available
- Run 5-10 episodes, not 120
- Compare against the simplest possible baseline
- Results don't need to be statistically significant — you're looking for
  signal, not proof

**If the fragile assumption fails → STOP. Do not proceed to implementation.**

Report:
```markdown
## Pilot Result:
- Assumption 1 tested: [description]
- Result: [numbers]
- Verdict: CONFIRMED / FALSIFIED / INCONCLUSIVE
- If FALSIFIED: [what this means, whether to pivot or abandon]
```

## Step 3: Cross-Validate with Literature (30 min)

Search for papers that tried similar approaches. Did they work? Did they
identify the same assumptions? Did they fail for the reasons you'd expect?

Specifically search for:
- "Has anyone tried [your core mechanism] before?"
- "What happened when they did?"
- "Are there known failure modes for this approach?"

**If the literature says this approach fails → take that seriously.**

## Step 4: Define Minimum Viable Experiment (30 min)

If assumptions are confirmed, define the SMALLEST experiment that would
demonstrate the core contribution. Not the full paper — just ONE result
that proves the idea works.

```markdown
## MVE (Minimum Viable Experiment):
- Model: [which model to use]
- Benchmark: [which tasks — choose the ones where the effect should be largest]
- Baseline: [the simplest meaningful comparison]
- Metric: [what number proves it works]
- Success threshold: [what result would convince a skeptic]
- Time estimate: [hours, not days]
- If MVE succeeds → proceed to full implementation
- If MVE fails → kill the idea
```

## Step 5: Decision Gate

Based on Steps 1-4, make a clear recommendation:

- **PROCEED**: Assumptions confirmed, pilot positive, literature supportive, MVE defined
- **PIVOT**: Core assumption failed but a variant might work — define the variant
- **ABANDON**: Idea is fundamentally broken — explain why and suggest alternatives

**This gate is BLOCKING.** No implementation work begins until the verdict is PROCEED.

## Key Rules

- **Test assumptions, not implementations.** You don't need a framework to test
  whether instruction swapping breaks a VLA.
- **Kill early, kill cheap.** A failed pilot costs 1 hour. A failed paper costs months.
- **The most fragile assumption is tested FIRST.** Not the most interesting one.
- **Negative pilot results are valuable.** They save weeks of wasted work.
- **Do NOT rationalize a failed pilot.** "Maybe it'll work with more engineering" is
  the sunk cost fallacy. If the core mechanism is broken, more code won't fix it.
- **Literature failures are warnings, not challenges.** If others tried this and failed,
  you need a clear reason why your version is different.
