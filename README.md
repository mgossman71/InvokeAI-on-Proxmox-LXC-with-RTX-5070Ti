# InvokeAI on Proxmox LXC with RTX 5070 Ti

Tested environment:

- Proxmox LXC
- Ubuntu 24.04
- NVIDIA RTX 5070 Ti (Blackwell)
- InvokeAI 6.12.0
- PyTorch 2.7.1 + CUDA 12.8

---

## 1. Pass NVIDIA GPU into the LXC (Proxmox host)

Replace `152` with your container ID.

```bash
pct set 152 -dev0 /dev/nvidia0
pct set 152 -dev1 /dev/nvidiactl
pct set 152 -dev2 /dev/nvidia-uvm
pct set 152 -dev3 /dev/nvidia-uvm-tools
pct set 152 -dev4 /dev/nvidia-caps/nvidia-cap1
pct set 152 -dev5 /dev/nvidia-caps/nvidia-cap2

pct restart 152
```

## 2. Verify GPU inside the LXC

```bash
nvidia-smi
ls -l /dev/nvidia*
```

If this fails, stop and fix GPU passthrough.

## 3. Install base packages

```bash
apt update && apt upgrade -y

apt install -y \
  git \
  python3 \
  python3-venv \
  python3-pip \
  build-essential \
  ffmpeg
```

## 4. Create Python environment

```bash
python3 -m venv /opt/invokeai
source /opt/invokeai/bin/activate

pip install --upgrade pip
```

## 5. Install InvokeAI

```bash
pip install invokeai
```

## 6. Install Blackwell-compatible PyTorch (CRITICAL)

Default Torch install does not support RTX 50-series (`sm_120`).

```bash
pip uninstall -y torch torchvision torchaudio triton

pip install torch==2.7.1+cu128 torchvision==0.22.1+cu128 torchaudio==2.7.1+cu128 \
  --index-url https://download.pytorch.org/whl/cu128
```

## 7. Verify CUDA + GPU

```bash
python - <<'PY'
import torch
print("torch:", torch.__version__)
print("cuda:", torch.version.cuda)
print("cuda available:", torch.cuda.is_available())
print("arch list:", torch.cuda.get_arch_list())
print("gpu:", torch.cuda.get_device_name(0))
PY
```

Expected:

```text
torch: 2.7.1+cu128
cuda: 12.8
cuda available: True
arch list: ... sm_120 ...
gpu: NVIDIA GeForce RTX 5070 Ti
```

## 8. Create InvokeAI runtime root

```bash
mkdir -p /opt/invokeai-root/{models,outputs,databases,nodes}
```

## 9. Create config file

```bash
cat > /opt/invokeai-root/invokeai.yaml <<'EOF'
schema_version: 4.0.2
host: 0.0.0.0
port: 9090
models_dir: models
outputs_dir: outputs
db_dir: databases
custom_nodes_dir: nodes
EOF
```

## 10. Start InvokeAI

```bash
source /opt/invokeai/bin/activate
invokeai-web --root /opt/invokeai-root
```

## 11. Access UI

Open this URL in your browser:

```text
http://<LXC-IP>:9090
```

Find the LXC IP:

```bash
hostname -I
```

## 12. Monitor GPU usage

```bash
watch -n 1 nvidia-smi
```

## Failures & Fixes

### ❌ Old setup command

```bash
invokeai-configure
```

Error:

```text
command not found
```

Fix: Not used in InvokeAI 6.x.

### ❌ CLI host/port flags

```bash
invokeai-web --host 0.0.0.0 --port 9090
```

Error:

```text
unrecognized arguments
```

Fix: Configure host and port in `invokeai.yaml`.

### ❌ Missing `schema_version`

Minimal config caused:

```text
KeyError: 'schema_version'
```

Fix:

```yaml
schema_version: 4.0.2
```

### ❌ Wrong PyTorch build

Error:

```text
sm_120 not supported
```

Fix:

```bash
pip install torch==2.7.1+cu128 ...
```

## Recommended Models

Start with:

- Juggernaut XL
- RealVisXL
- DreamShaper XL

Then try:

- FLUX.1-schnell (fast)
- FLUX.1-dev (higher quality)
