---
name: research-pipeline
description: "Full research pipeline: Workflow 1 (idea discovery) → implementation → training → real experiments → validation → review → paper. Use when user says \"全流程\", \"full pipeline\", \"从找idea到投稿\", \"end-to-end research\", or wants the complete autonomous research lifecycle."
argument-hint: [research-direction]
allowed-tools: Bash(*), Read, Write, Edit, Grep, Glob, WebSearch, WebFetch, Agent, Skill, mcp__codex__codex, mcp__codex__codex-reply
---

# Full Research Pipeline: Idea → Real Experiments → Submission

End-to-end autonomous research workflow for: **$ARGUMENTS**

## Constants

- **AUTO_PROCEED = true** — When `true`, Gate 1 auto-selects the top-ranked idea. When `false`, waits for explicit user confirmation at every gate.
- **ARXIV_DOWNLOAD = false** — When `true`, `/research-lit` downloads arXiv PDFs.
- **HUMAN_CHECKPOINT = false** — When `true`, pause after each review round.

> Override: `/research-pipeline "topic" — AUTO_PROCEED: false, human checkpoint: true`

## Overview

```
/idea-discovery → implement → TRAIN → VALIDATE → /run-experiment → VALIDATE → /auto-review-loop → VISUALIZE → /paper-writing
├── Workflow 1 ──┤            ├─ Gate 2 ─┤          ├── Gate 3 ──┤            ├── Workflow 2 ──┤          ├── Gate 4 ──┤         ├── WF3 ──┤
```

**Four validation gates prevent fake research from reaching the paper:**

| Gate | When | What it checks | Blocks if... |
|------|------|----------------|-------------|
| **Gate 1** | After idea discovery | Idea quality, novelty | No viable idea found |
| **Gate 2** | After implementation | Trained model exists | No checkpoint, only scaffold code |
| **Gate 3** | After experiments | Real execution, not oracle/mock | Oracle mode, mock env, no sim frames |
| **Gate 4** | Before paper writing | Real visualization exists | No actual simulation renders |

## Pipeline

### Stage 1: Idea Discovery (Workflow 1)

```
/idea-discovery "$ARGUMENTS"
```

Internally: `/research-lit` → `/idea-creator` → `/novelty-check` → `/research-review`

**Output:** `IDEA_REPORT.md`

**🚦 Gate 1 — Idea Checkpoint:** Present ranked ideas. Wait for user confirmation (or auto-select if AUTO_PROCEED=true).

---

### Stage 2: Implementation

Once an idea is confirmed:

1. **Implement the method** — model architecture, training loop, evaluation scripts
2. **Implement baselines** — matching compute and data
3. **Code review** — reproducibility checks (seeds, configs, logging)

**CRITICAL**: Implementation MUST include:
- A **training script** that produces a model checkpoint
- An **evaluation script** that loads the checkpoint and runs actual inference
- A **visualization script** that captures simulation renders during evaluation

If the method builds on an existing model (e.g., adding a scheduler on top of a pretrained VLA), the implementation must include:
- Integration code that actually calls the pretrained model's inference
- NOT an oracle/mock that simulates what the model would do

---

### 🚦 Gate 2: Training Validation (NEW)

**Before deploying experiments, verify a trained model exists.**

```
/validate-experiment [project-dir]
```

Check 1: Does a model checkpoint exist (trained or pretrained for the target task)?
Check 2: Can the model run inference on the target environment?

**If FAIL:**
- If training is needed: estimate GPU hours, write SLURM training script, deploy training job, wait for completion.
- If using pretrained model: verify the checkpoint loads and produces reasonable outputs on one example.
- DO NOT skip to evaluation with oracle mode. Oracle results are supplementary upper bounds, not primary evidence.

**If PASS:** Proceed to Stage 3.

---

### Stage 3: Deploy Real Experiments

```
/run-experiment [experiment command]
```

**MANDATORY requirements for experiments:**
1. Model inference must use the **actual trained/pretrained model**, not an oracle
2. Environment must use **actual physics simulation** with rendering, not mock mode
3. Evaluation must capture **RGB frames** from the simulation at key timesteps
4. All results must include **per-episode data with seeds** for reproducibility

**What to deploy:**
- Main evaluation: model on all tasks, multiple seeds
- Baseline evaluation: same tasks, same seeds, without the proposed method
- Ablation evaluation: remove one component at a time
- **Frame capture**: save simulation renders at critical moments (success, failure, recovery)

**Monitor and collect results:**
```
/monitor-experiment [server]
```

---

### 🚦 Gate 3: Experiment Reality Check (NEW)

**Before entering the review loop, verify experiments are real.**

```
/validate-experiment [project-dir]
```

Run ALL five checks from the validate-experiment skill:
1. Trained model exists → PASS/FAIL
2. No oracle/mock in evaluation → PASS/FAIL
3. Simulation renders actual observations → PASS/FAIL
4. Visualization shows real system output → PASS/FAIL
5. Results are reproducible → PASS/FAIL

**If ANY check fails:** BLOCK the pipeline. Fix the issue. Re-run experiments.

**Oracle results policy:**
- Oracle experiments MAY be run as supplementary analysis (e.g., "upper bound if policy were perfect")
- Oracle results MUST be clearly labeled as such in the paper
- Oracle results MUST NOT be the primary evidence for any claim
- The paper MUST include results from actual model execution as the primary evaluation

**If PASS:** Proceed to Stage 4.

---

### Stage 4: Auto Review Loop (Workflow 2)

```
/auto-review-loop "$ARGUMENTS — [chosen idea title]"
```

Up to 4 rounds of: GPT-5.4 reviews → Claude implements fixes → re-evaluate → re-review.

**IMPORTANT**: The review prompt MUST include:
- Whether results are from real model execution or oracle
- Whether simulation frames exist
- The exact model used (name, checkpoint, training details)

The reviewer should be asked to verify that the evidence is real, not just internally consistent.

**Output:** `AUTO_REVIEW.md`

---

### 🚦 Gate 4: Visualization Validation (NEW)

**Before writing the paper, verify that real visualizations exist.**

Required visualizations:
1. **Simulation frames** at key timesteps showing the actual robot/agent executing the task
2. **Success case**: frames showing the proposed method succeeding
3. **Failure case**: frames showing a baseline or ablation failing on the same task
4. **Comparison**: side-by-side showing the method's advantage visually
5. **Quantitative figures**: charts generated from real (not oracle) data

**If visualization is missing:** Capture frames by running the system with rendering enabled. Do NOT proceed to paper writing without real visual evidence.

**What counts as real visualization:**
- RGB frames from MuJoCo/Isaac/real robot showing objects, robot arm, workspace
- Resolution ≥ 256×256
- Shows actual task execution (objects being manipulated, not just static scenes)

**What does NOT count:**
- PIL-generated colored rectangles with text labels
- Abstract node diagrams without simulation context
- Charts from oracle/mock data

---

### Stage 5: Paper Writing (Workflow 3)

```
/paper-writing "NARRATIVE_REPORT.md — venue: [target]"
```

Internally: `/paper-plan` → `/paper-figure` → `/paper-write` → `/paper-compile` → `/auto-paper-improvement-loop`

**MANDATORY**: The paper-figure stage MUST include:
- Actual simulation frames (from Gate 4 validation)
- Charts from real experiment data (not oracle)
- Architecture diagrams created programmatically (TikZ/code) or with high-quality tools

**Oracle results in the paper:**
- MAY appear in an ablation/analysis section as "oracle upper bound"
- MUST NOT appear in the main results tables without clear labeling
- MUST NOT be the only evidence for any claim

---

### Stage 6: Final Summary

```markdown
# Research Pipeline Report

**Direction**: $ARGUMENTS
**Chosen Idea**: [title]
**Date**: [start] → [end]

## Validation Status
| Gate | Status | Details |
|------|--------|---------|
| Gate 1: Idea | PASS | [idea title] selected |
| Gate 2: Training | PASS | [checkpoint path], [training hours] |
| Gate 3: Real Experiments | PASS | [model name], [no oracle in primary results] |
| Gate 4: Visualization | PASS | [N] simulation frames captured |

## Results Summary
- Primary results: [from real model execution]
- Oracle upper bound: [if applicable, clearly labeled]
- Baselines: [list]
- Ablations: [list]

## Deliverables
- paper/main.pdf — [pages] pages, [venue]
- figures/viz/ — [N] simulation frames
- results/ — [N] episodes with per-seed data
```

## Key Rules

- **ALL FOUR GATES ARE MANDATORY.** Skipping a gate produces fake research.
- **Oracle mode is supplementary, not primary.** Never build a paper on oracle results alone.
- **Real simulation frames are required.** A robotics paper without robot images is not publishable.
- **The reviewer (GPT-5.4) must be told whether results are real or oracle.** Hiding this from the reviewer produces inflated scores.
- **If training takes too long**, use a pretrained model and fine-tune, or use a simpler task. Do not substitute oracle mode for real execution.
- **Fail gracefully**: If Gate 2 or 3 fails, report what's missing and estimate the cost to fix it. Let the user decide.
