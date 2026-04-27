InvokeAI on Proxmox LXC with RTX 5070 Ti

Tested target:

Proxmox LXC
Ubuntu 24.04
NVIDIA RTX 5070 Ti / Blackwell
InvokeAI 6.12.0
PyTorch 2.7.1 + CUDA 12.8
1. Pass NVIDIA GPU into the LXC

Run on the Proxmox host.

Replace 152 with your LXC ID if different.

pct set 152 -dev0 /dev/nvidia0
pct set 152 -dev1 /dev/nvidiactl
pct set 152 -dev2 /dev/nvidia-uvm
pct set 152 -dev3 /dev/nvidia-uvm-tools
pct set 152 -dev4 /dev/nvidia-caps/nvidia-cap1
pct set 152 -dev5 /dev/nvidia-caps/nvidia-cap2

pct restart 152
2. Verify GPU inside the LXC

Run inside the LXC.

nvidia-smi
ls -l /dev/nvidia*

If nvidia-smi does not work, fix GPU passthrough before continuing.

3. Install base packages
apt update && apt upgrade -y

apt install -y \
  git \
  python3 \
  python3-venv \
  python3-pip \
  build-essential \
  ffmpeg
4. Create Python virtual environment
python3 -m venv /opt/invokeai
source /opt/invokeai/bin/activate

pip install --upgrade pip
5. Install InvokeAI
pip install invokeai
6. Replace PyTorch with Blackwell-compatible CUDA 12.8 build

InvokeAI installed Torch, but the default wheel did not support RTX 5070 Ti sm_120.

Fix it with:

pip uninstall -y torch torchvision torchaudio triton

pip install torch==2.7.1+cu128 torchvision==0.22.1+cu128 torchaudio==2.7.1+cu128 \
  --index-url https://download.pytorch.org/whl/cu128
7. Verify PyTorch sees the RTX 5070 Ti correctly
python - <<'PY'
import torch
print("torch:", torch.__version__)
print("cuda:", torch.version.cuda)
print("cuda available:", torch.cuda.is_available())
print("arch list:", torch.cuda.get_arch_list())
print("gpu:", torch.cuda.get_device_name(0))
PY

Expected important lines:

torch: 2.7.1+cu128
cuda: 12.8
cuda available: True
arch list: ... sm_120 ...
gpu: NVIDIA GeForce RTX 5070 Ti
8. Create InvokeAI runtime root
mkdir -p /opt/invokeai-root/{models,outputs,databases,nodes}
9. Create InvokeAI config
cat > /opt/invokeai-root/invokeai.yaml <<'EOF'
schema_version: 4.0.2
host: 0.0.0.0
port: 9090
models_dir: models
outputs_dir: outputs
db_dir: databases
custom_nodes_dir: nodes
EOF

Important: a minimal config without schema_version failed.

10. Start InvokeAI
source /opt/invokeai/bin/activate
invokeai-web --root /opt/invokeai-root

Open:

http://<LXC-IP>:9090

Find the LXC IP:

hostname -I
11. Monitor GPU usage
watch -n 1 nvidia-smi
What failed
Failed: old InvokeAI command
invokeai-configure --root /opt/invokeai-root

Result:

invokeai-configure: command not found

InvokeAI 6.x does not use that older setup flow.

Failed: CLI host/port flags
invokeai-web --host 0.0.0.0 --port 9090

Result:

unrecognized arguments: --host 0.0.0.0 --port 9090

Use invokeai.yaml instead.

Failed: minimal config
host: 0.0.0.0
port: 9090

Result:

KeyError: 'schema_version'

Fix was:

schema_version: 4.0.2
Failed: default PyTorch CUDA build

Torch saw the GPU but did not support Blackwell:

NVIDIA GeForce RTX 5070 Ti with CUDA capability sm_120 is not compatible

Fix was installing:

torch 2.7.1+cu128
Recommended first models

Start with SDXL models before FLUX:

Juggernaut XL
RealVisXL
DreamShaper XL

Then test:

FLUX.1-schnell
FLUX.1-dev
