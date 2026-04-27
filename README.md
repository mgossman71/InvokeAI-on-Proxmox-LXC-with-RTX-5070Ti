# InvokeAI on Proxmox LXC with RTX 5070 Ti

Fast install guide for running InvokeAI inside a Proxmox LXC with an NVIDIA RTX 5070 Ti GPU.

Tested environment:

- Proxmox LXC
- Ubuntu 24.04
- NVIDIA RTX 5070 Ti
- InvokeAI 6.12.0
- PyTorch 2.7.1 + CUDA 12.8

## 1. Pass the NVIDIA GPU into the LXC

Run on the Proxmox host.

Replace `152` with your LXC ID.

```bash
pct set 152 -dev0 /dev/nvidia0
pct set 152 -dev1 /dev/nvidiactl
pct set 152 -dev2 /dev/nvidia-uvm
pct set 152 -dev3 /dev/nvidia-uvm-tools
pct set 152 -dev4 /dev/nvidia-caps/nvidia-cap1
pct set 152 -dev5 /dev/nvidia-caps/nvidia-cap2

pct restart 152
```

## 2. Verify GPU access inside the LXC

```bash
nvidia-smi
ls -l /dev/nvidia*
```

If this fails, stop and fix GPU passthrough before continuing.

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

## 4. Create the Python environment

```bash
python3 -m venv /opt/invokeai
source /opt/invokeai/bin/activate

pip install --upgrade pip
```

## 5. Install InvokeAI

```bash
pip install invokeai
```

## 6. Install PyTorch with CUDA 12.8 support

This is required for RTX 50-series / Blackwell GPUs.

```bash
pip uninstall -y torch torchvision torchaudio triton

pip install torch==2.7.1+cu128 torchvision==0.22.1+cu128 torchaudio==2.7.1+cu128 \
  --index-url https://download.pytorch.org/whl/cu128
```

## 7. Verify PyTorch CUDA support

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

Expected result should include:

```text
torch: 2.7.1+cu128
cuda: 12.8
cuda available: True
arch list: ... sm_120 ...
gpu: NVIDIA GeForce RTX 5070 Ti
```

## 8. Create the InvokeAI runtime directory

```bash
mkdir -p /opt/invokeai-root/{models,outputs,databases,nodes}
```

## 9. Create the InvokeAI config

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

## 11. Open InvokeAI

Find the LXC IP:

```bash
hostname -I
```

Open:

```text
http://<LXC-IP>:9090
```

## 12. Monitor GPU usage

```bash
watch -n 1 nvidia-smi
```

## Recommended models

Start with:

- Juggernaut XL
- RealVisXL
- DreamShaper XL

Then try:

- FLUX.1-schnell
- FLUX.1-dev
