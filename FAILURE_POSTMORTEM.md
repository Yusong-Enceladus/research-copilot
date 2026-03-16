# Research CoPilot — Failure Postmortem

## Incident: GraphVLA paper built entirely on fake experimental data

### Timeline of Failures

| Step | What Happened | What Should Have Happened | Gate That Failed |
|------|--------------|--------------------------|-----------------|
| Implementation | Built framework code but no training integration | Should have integrated with a real trained model (OpenVLA-OFT existed on HPC3) | No Gate 2 existed |
| First eval | Used `oracle_mode=True` (random coin flip) as primary evaluation | Should have used real VLA inference on real simulation | No Gate 3 existed |
| Auto-review R1-R6 | GPT-5.4 scored oracle results without knowing they were fake | Reviewer should have been told results are oracle and capped score | No reality disclosure required |
| Visualization | Generated PIL rectangles labeled as "frames" | Should have captured actual MuJoCo RGB renders | No Gate 4 existed |
| Paper writing | Wrote paper with fake numbers and fake figures | Should have been blocked until real results existed | No pre-flight validation |
| Figure generation | TikZ diagrams + Nanobanana Pro approximations, no real sim output | Should have required actual experiment screenshots | No figure quality gate |
| HPC3 deployment | Proxy issues, offline mode not set, env not validated | Should have run pre-flight env check before first experiment | No env validation |

### Root Causes

1. **The workflow optimized for speed over truth.** It prioritized producing outputs (paper, figures, scores) over producing real science.
2. **No checkpoint required actual execution proof.** Every gate checked structure (code exists, config exists) but not substance (model runs, sim renders, policy works).
3. **The reviewer was complicit.** GPT-5.4 scored claimed numbers without questioning whether they were real.
4. **Oracle mode was treated as a feature, not a crutch.** The code had `oracle_mode` as a convenient testing path that became the primary evaluation path.
5. **Visualization was treated as decoration.** Figures were generated from data, not captured from experiments.

### Lessons Encoded into Workflow

Each lesson below maps to a specific code change in the updated skills.
