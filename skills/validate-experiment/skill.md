---
name: validate-experiment
description: "Validate that experiments use real execution, not oracles/mocks. Gate skill that BLOCKS the pipeline if experiments are fake. Use before auto-review-loop and paper-writing."
argument-hint: [project-directory]
allowed-tools: Bash(*), Read, Grep, Glob, Write, Edit
---

# Validate Experiment: Reality Check Gate

**This is a BLOCKING gate.** It verifies that experimental results come from real model execution, not from oracles, mocks, or random number generators. If validation fails, the pipeline MUST NOT proceed to review or paper writing.

## Context: $ARGUMENTS

## Why This Exists

Without this gate, a research pipeline can produce a paper that looks complete but contains no real science — oracle modes simulate success with configurable probability, mock environments bypass physics, and the auto-review loop will accept fake numbers because it only checks internal consistency. A real reviewer will reject the paper immediately.

## Validation Checklist

Run ALL of the following checks. Report each as PASS/FAIL with evidence.

### Check 1: Trained Model Exists

```bash
# Search for model checkpoints in the project and common locations
find . -name "*.pt" -o -name "*.pth" -o -name "*.safetensors" -o -name "*.ckpt" | head -20
```

**PASS**: At least one checkpoint file exists, and it was produced by training (not downloaded as a generic pretrained model without fine-tuning on the target task).

**FAIL**: No checkpoint found, or only generic pretrained weights exist without task-specific fine-tuning.

**If FAIL**: List available pretrained models that could be fine-tuned. Estimate training time. BLOCK pipeline.

### Check 2: No Oracle/Mock Mode in Evaluation

```bash
# Search for oracle mode, mock mode, or random success in eval scripts
grep -rn "oracle_mode\|mock_mode\|oracle_success_prob\|random.*success\|simulated.*completion" scripts/ --include="*.py"
```

**PASS**: Evaluation scripts use actual model inference on actual simulation/environment, with NO oracle/mock fallback during reported results.

**FAIL**: Evaluation uses `oracle_mode=True`, `mock_mode`, or any mechanism that simulates success without actual model execution.

**If FAIL**: Identify which evaluation runs used oracle mode. Those results MUST be clearly labeled as "oracle upper bound" in the paper, NOT as primary results. The pipeline MUST produce real-execution results before proceeding.

Also scan RESULTS files for oracle contamination:
```bash
# Check if any results JSON files contain oracle_mode=True
python -c "
import json, glob
for f in glob.glob('results/**/*.json', recursive=True):
    with open(f) as fh:
        try:
            data = json.load(fh)
        except: continue
    episodes = data.get('episodes', [data] if isinstance(data, dict) else data)
    for ep in (episodes[:3] if isinstance(episodes, list) else []):
        if ep.get('oracle_success_prob') or ep.get('mock_mode') or ep.get('oracle_mode'):
            print(f'ORACLE CONTAMINATION: {f} — oracle_success_prob={ep.get(\"oracle_success_prob\")}')
            break
"
```

**If oracle contamination is found in results:** Those results MUST be moved to a `results/oracle_supplementary/` directory and clearly labeled. They MUST NOT be in `results/` alongside real results.

### Check 3: Simulation Actually Renders

```bash
# Check that environment produces actual observations, not empty dicts
grep -rn "obs = {}\|env = None\|_mock_mode\|# Mock:" scripts/ envs/ --include="*.py"
```

**PASS**: Environment returns actual observations (RGB images, proprioception) from a physics engine (MuJoCo, Isaac, etc.), not empty placeholders.

**FAIL**: Environment returns empty observations or uses mock mode.

**If FAIL**: BLOCK pipeline. Real simulation observations are required for both results and visualization.

### Check 4: Visualization Shows Real System Output

```bash
# Check for actual rendered frames (not PIL-generated diagrams)
find figures/ results/ -name "*.png" -o -name "*.jpg" -exec identify {} \; 2>/dev/null | head -10
# Check image dimensions — real sim frames are typically 256x256 or larger
```

**PASS**: Figures directory contains actual simulation renders (robot workspace images with objects, robot arm, etc.) of reasonable resolution.

**FAIL**: Only schematic diagrams, PIL-generated rectangles, or matplotlib charts exist. No actual simulation output.

**If FAIL**: BLOCK paper writing. Capture actual simulation frames by running the system with rendering enabled. This is required for the paper's qualitative results.

### Check 5: Results Are Reproducible

```bash
# Check that results JSON files contain per-episode data with seeds
python -c "
import json, glob
for f in glob.glob('results/**/*.json', recursive=True)[:3]:
    with open(f) as fh:
        data = json.load(fh)
    episodes = data.get('episodes', data if isinstance(data, list) else [])
    if episodes:
        e = episodes[0] if isinstance(episodes, list) else episodes
        print(f'{f}: seed={e.get(\"seed\", \"MISSING\")}, steps={e.get(\"total_steps\", \"MISSING\")}')
"
```

**PASS**: Results contain per-episode data with seeds, and running the same seed produces the same result.

**FAIL**: Results are missing seeds, or results are not deterministic.

## Output

Write `EXPERIMENT_VALIDATION.md` in the project root:

```markdown
# Experiment Validation Report

**Date**: [today]
**Project**: [project name]

## Checklist

| # | Check | Status | Evidence |
|---|-------|--------|----------|
| 1 | Trained model exists | PASS/FAIL | [checkpoint path or "NONE"] |
| 2 | No oracle/mock in eval | PASS/FAIL | [oracle lines found or "clean"] |
| 3 | Simulation renders | PASS/FAIL | [env type and observation shape] |
| 4 | Real visualization | PASS/FAIL | [frame count and resolution] |
| 5 | Reproducible results | PASS/FAIL | [seed verification] |

## Verdict

**PROCEED** / **BLOCKED — [reason]**

## Required Actions (if BLOCKED)
1. [specific action needed]
2. [specific action needed]
```

## Key Rules

- **This gate is NOT optional.** It must run before `/auto-review-loop` and before `/paper-writing`.
- **Oracle results are supplementary, not primary.** If oracle results exist, they must be clearly labeled as upper bounds.
- **Mock environments produce fake data.** No paper should be written from mock environment output.
- **Real simulation frames are required.** A paper about robot manipulation without robot images is immediately suspicious.
- **If any check fails, the pipeline STOPS.** Fix the issue before proceeding.
