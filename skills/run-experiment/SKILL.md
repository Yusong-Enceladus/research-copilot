---
name: run-experiment
description: Deploy and run ML experiments on local or remote GPU servers. Use when user says "run experiment", "deploy to server", "跑实验", or needs to launch training jobs.
argument-hint: [experiment-description]
allowed-tools: Bash(*), Read, Grep, Glob, Edit, Write, Agent
---

# Run Experiment

Deploy and run ML experiment: $ARGUMENTS

## Workflow

### Step 1: Detect Environment

Read the project's `CLAUDE.md` to determine the experiment environment:

- **Local GPU**: Look for local CUDA/MPS setup info
- **Remote server**: Look for SSH alias, conda env, code directory

If no server info is found in `CLAUDE.md`, ask the user.

### Step 1b: Environment Pre-flight (NEW)

Before running any experiment, verify the compute environment works:

1. **Network**: Check if the compute node has internet access. If not (typical for HPC), set:
   ```bash
   export HF_HUB_OFFLINE=1
   export TRANSFORMERS_OFFLINE=1
   unset http_proxy https_proxy HTTP_PROXY HTTPS_PROXY ALL_PROXY all_proxy
   ```

2. **Rendering**: For embodied AI / robotics tasks, verify rendering works:
   ```bash
   python -c "import mujoco; print('MuJoCo', mujoco.__version__)"
   MUJOCO_GL=egl python -c "
   import mujoco, numpy as np
   m = mujoco.MjModel.from_xml_string('<mujoco><worldbody><light/><geom type=\"plane\" size=\"1 1 .1\"/></worldbody></mujoco>')
   d = mujoco.MjData(m)
   r = mujoco.Renderer(m, 256, 256)
   mujoco.mj_step(m, d)
   r.update_scene(d)
   img = r.render()
   assert img.shape == (256, 256, 3), f'Render failed: {img.shape}'
   print('EGL rendering OK:', img.shape)
   "
   ```

3. **Model loading**: Verify the trained model can be loaded:
   ```bash
   python -c "import torch; model = torch.load('checkpoint.pt', map_location='cpu'); print('Model loaded OK')"
   ```

4. **Pilot run**: Execute ONE episode with the actual model before launching full evaluation. If the pilot fails, DO NOT submit the full batch job.

**If any pre-flight check fails, STOP and fix the environment before proceeding.**

### Step 2: Pre-flight Check

Check GPU availability on the target machine:

**Remote:**
```bash
ssh <server> nvidia-smi --query-gpu=index,memory.used,memory.total --format=csv,noheader
```

**Local:**
```bash
nvidia-smi --query-gpu=index,memory.used,memory.total --format=csv,noheader
# or for Mac MPS:
python -c "import torch; print('MPS available:', torch.backends.mps.is_available())"
```

Free GPU = memory.used < 500 MiB.

### Step 3: Sync Code (Remote Only)

Only sync necessary files — NOT data, checkpoints, or large files:
```bash
rsync -avz --include='*.py' --exclude='*' <local_src>/ <server>:<remote_dst>/
```

### Step 4: Deploy

#### Remote (via SSH + screen)

For each experiment, create a dedicated screen session with GPU binding:
```bash
ssh <server> "screen -dmS <exp_name> bash -c '\
  eval \"\$(<conda_path>/conda shell.bash hook)\" && \
  conda activate <env> && \
  CUDA_VISIBLE_DEVICES=<gpu_id> python <script> <args> 2>&1 | tee <log_file>'"
```

#### Local

```bash
# Linux with CUDA
CUDA_VISIBLE_DEVICES=<gpu_id> python <script> <args> 2>&1 | tee <log_file>

# Mac with MPS (PyTorch uses MPS automatically)
python <script> <args> 2>&1 | tee <log_file>
```

For local long-running jobs, use `run_in_background: true` to keep the conversation responsive.

### Step 5: Verify Launch

**Remote:**
```bash
ssh <server> "screen -ls"
```

**Local:**
Check process is running and GPU is allocated.

### Step 5b: Pilot Verification (NEW)

After deploying code but BEFORE launching full evaluation:

1. Run exactly ONE episode with the actual model on the actual environment
2. Verify the episode produces:
   - Non-zero reward or meaningful completion metric
   - At least one RGB frame saved (if robotics)
   - Results JSON with per-step data
3. If the pilot fails or produces empty/zero results, DO NOT launch full evaluation.

Example pilot command:
```bash
python eval.py --num-trials 1 --save-frames --verbose 2>&1 | tail -20
# Check: does it show actual model inference? Not "oracle" or "mock"?
ls figures/viz/  # Check: do actual image files exist?
```

### Step 6: Feishu Notification (if configured)

After deployment is verified, check `~/.claude/feishu.json`:
- Send `experiment_done` notification: which experiments launched, which GPUs, estimated time
- If config absent or mode `"off"`: skip entirely (no-op)

## Key Rules

- ALWAYS check GPU availability first — never blindly assign GPUs
- Each experiment gets its own screen session + GPU (remote) or background process (local)
- Use `tee` to save logs for later inspection
- Run deployment commands with `run_in_background: true` to keep conversation responsive
- Report back: which GPU, which screen/process, what command, estimated time
- If multiple experiments, launch them in parallel on different GPUs

## CLAUDE.md Example

Users should add their server info to their project's `CLAUDE.md`:

```markdown
## Remote Server
- SSH: `ssh my-gpu-server`
- GPU: 4x A100 (80GB each)
- Conda: `eval "$(/opt/conda/bin/conda shell.bash hook)" && conda activate research`
- Code dir: `/home/user/experiments/`

## Local Environment
- Mac MPS / Linux CUDA
- Conda env: `ml` (Python 3.10 + PyTorch)
```
