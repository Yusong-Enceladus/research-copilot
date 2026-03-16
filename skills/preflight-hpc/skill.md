---
name: preflight-hpc
description: "Validate HPC environment before experiment deployment. Checks GPU, rendering, network, dependencies, and model loading. Use before submitting SLURM jobs."
argument-hint: [server-name]
allowed-tools: Bash(*), Read, Write
---

# Pre-flight HPC Validation

Validate compute environment on: **$ARGUMENTS**

## Checks

### 1. Connectivity
```bash
ssh $SERVER "echo 'SSH OK'; hostname; nvidia-smi -L | head -2"
```

### 2. GPU Availability
```bash
ssh $SERVER "nvidia-smi --query-gpu=index,name,memory.used,memory.total --format=csv,noheader"
```

### 3. Network (proxy detection)
```bash
ssh $SERVER "env | grep -i proxy; curl -s --max-time 5 https://huggingface.co > /dev/null 2>&1 && echo 'HF reachable' || echo 'HF BLOCKED — set HF_HUB_OFFLINE=1'"
```

### 4. Conda Environment
```bash
ssh $SERVER "conda activate openvla-oft 2>/dev/null && python -c 'import torch; print(f\"PyTorch {torch.__version__}, CUDA: {torch.cuda.is_available()}\")'"
```

### 5. Rendering (for robotics)
```bash
ssh $SERVER "srun --gres=gpu:1 --time=5:00 --partition=acd_u bash -c '
export MUJOCO_GL=egl
python -c \"
import mujoco
m = mujoco.MjModel.from_xml_string(chr(60)+\"mujoco\"+chr(62)+chr(60)+\"worldbody\"+chr(62)+chr(60)+\"light/\"+chr(62)+chr(60)+\"geom type=\\\"plane\\\" size=\\\"1 1 .1\\\"/\"+chr(62)+chr(60)+\"/worldbody\"+chr(62)+chr(60)+\"/mujoco\"+chr(62))
d = mujoco.MjData(m)
r = mujoco.Renderer(m, 256, 256)
mujoco.mj_step(m, d)
r.update_scene(d)
img = r.render()
print(f\"EGL render OK: {img.shape}\")
\"
'"
```

### 6. Model Loading
```bash
ssh $SERVER "srun --gres=gpu:1 --time=5:00 --partition=acd_u bash -c '
export HF_HUB_OFFLINE=1
python -c \"import torch; ckpt=torch.load(\\\"CHECKPOINT_PATH\\\", map_location=\\\"cuda:0\\\"); print(f\\\"Checkpoint loaded: {type(ckpt)}\\\")\"
'"
```

## Output

Write `PREFLIGHT_REPORT.md`:
```markdown
| Check | Status | Details |
|-------|--------|---------|
| SSH | PASS/FAIL | [hostname] |
| GPU | PASS/FAIL | [GPU model, memory] |
| Network | PASS/FAIL | [proxy status, HF reachable?] |
| Conda | PASS/FAIL | [PyTorch version, CUDA] |
| Rendering | PASS/FAIL | [EGL status] |
| Model Load | PASS/FAIL | [checkpoint type] |
```

**ALL checks must PASS before submitting experiment jobs.**
