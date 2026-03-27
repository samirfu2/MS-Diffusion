# MS-Diffusion EC2 Setup Guide

Complete step-by-step guide to set up MS-Diffusion on a fresh EC2 machine.

---

## 1. EC2 Instance Requirements

| Setting | Value |
|---------|-------|
| Instance type | `g5.2xlarge` (24GB VRAM, recommended) or `g4dn.2xlarge` (16GB, budget) |
| AMI | **Ubuntu 22.04** (standard, NOT Deep Learning AMI — conda not pre-installed) |
| Storage | **100GB+** EBS (models alone are ~30GB) |
| Security group | Port 22 (SSH), Port 8888 (Jupyter, optional) |
| Elastic IP | Assign a static Elastic IP to avoid IP change on restart |

> **Storage is critical.** SDXL base model is ~7GB, CLIP is ~10GB, MS-Diffusion checkpoint ~5GB. Use 100GB minimum.

---

## 2. SSH Access Setup

### On your local Mac — get your public key
```bash
cat ~/.ssh/id_rsa.pub
```
Copy the full output.

### On AWS Console — add key via EC2 Instance Connect
1. Go to AWS Console → EC2 → your instance
2. Click **Connect → EC2 Instance Connect → Connect** (browser terminal)
3. In the browser terminal, paste:
```bash
echo "YOUR_PUBLIC_KEY_HERE" >> ~/.ssh/authorized_keys
```

### Fix host key warning (when reusing Elastic IP on new instance)
```bash
ssh-keygen -R <your-elastic-ip>
ssh ubuntu@<your-elastic-ip>   # type 'yes' when prompted
```

---

## 3. Install Miniconda

```bash
# Download Miniconda
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh

# Install silently
bash Miniconda3-latest-Linux-x86_64.sh -b -p ~/miniconda3

# Initialize and reload shell
~/miniconda3/bin/conda init bash
source ~/.bashrc

# Verify
conda --version
```

### Accept Conda Terms of Service (required on fresh install)
```bash
conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/main
conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/r
```

---

## 4. Clone the Repository

```bash
# Create working directory
mkdir -p ~/PhD
cd ~/PhD

# Clone your fork
git clone https://github.com/samirfu2/MS-Diffusion.git
cd MS-Diffusion
```

---

## 5. Create Conda Environment

```bash
conda create -n msdiffusion python=3.10 -y
conda activate msdiffusion
```

---

## 6. Install PyTorch with CUDA

```bash
pip install torch==2.0.1 torchvision torchaudio --index-url https://download.pytorch.org/whl/cu118
```

> **Important:** Use `torch==2.0.1` — the `requirements.txt` will downgrade to this version anyway. Installing it first avoids a redundant large download.

Verify GPU:
```bash
python -c "import torch; print(torch.__version__, torch.cuda.is_available(), torch.cuda.get_device_name(0))"
# Expected: 2.0.1 True NVIDIA A10G (or similar)
```

Also verify with:
```bash
nvidia-smi
```

---

## 7. Install Project Dependencies

```bash
pip install -r requirements.txt
```

> **Note:** You may see a `torchaudio` version conflict warning — this is harmless. MS-Diffusion does not use audio.

---

## 8. Download Model Weights

Use `tmux` so downloads survive SSH disconnects:
```bash
tmux new -s downloads
# Run all downloads inside this tmux session
# To detach: Ctrl+B then D
# To reattach: tmux attach -t downloads
```

Download **one at a time** (running in parallel causes lock file conflicts):

```bash
pip install huggingface_hub

# 1. SDXL base model (~7GB)
huggingface-cli download stabilityai/stable-diffusion-xl-base-1.0 \
  --local-dir ./models/sdxl-base-1.0

# 2. CLIP encoder (~10GB)
huggingface-cli download laion/CLIP-ViT-bigG-14-laion2B-39B-b160k \
  --local-dir ./models/CLIP-ViT-bigG-14

# 3. MS-Diffusion checkpoint (~5GB)
huggingface-cli download MS-Diffusion/MS-Diffusion \
  --local-dir ./models/ms-diffusion
```

### Troubleshooting: "No space left on device"
```bash
df -h                          # check disk usage
du -sh ~/PhD/MS-Diffusion/models/   # check model folder size
rm -rf ./models/*/.cache       # clean partial downloads
```
If disk is full, resize EBS volume in AWS Console → EC2 → Volumes → Modify Volume, then:
```bash
sudo growpart /dev/xvda 1
sudo resize2fs /dev/xvda1
df -h   # confirm new size
```

---

## 9. Configure Model Paths

Edit `config.py` to point to your downloaded models:
```python
sdxl_path = "./models/sdxl-base-1.0"
clip_path  = "./models/CLIP-ViT-bigG-14"
ms_ckpt    = "./models/ms-diffusion"
```

---

## 10. Run Inference

```bash
python inference.py
# or with ControlNet:
python inference_controlnet.py
```

Monitor GPU usage during inference:
```bash
watch -n1 nvidia-smi
```

---

## 11. Optional: Jupyter Notebook

```bash
pip install jupyter
jupyter notebook --ip=0.0.0.0 --port=8888 --no-browser &
```

On your Mac, create an SSH tunnel:
```bash
ssh -L 8888:localhost:8888 ubuntu@<your-elastic-ip>
```

Then open `http://localhost:8888` in your browser.

---

## 12. Daily Development Workflow

### On Mac (make changes, push to GitHub)
```bash
git add <files>
git commit -m "your message"
git push origin main
```

### On EC2 (pull latest changes)
```bash
cd ~/PhD/MS-Diffusion
git pull origin main
```

### Sync with original paper repo (when needed)
```bash
# On Mac
git fetch upstream
git merge upstream/main
git push origin main
# On EC2
git pull origin main
```

Remotes should look like:
```
origin    https://github.com/samirfu2/MS-Diffusion.git
upstream  https://github.com/MS-Diffusion/MS-Diffusion.git
```

---

## Verification Checklist

- [ ] `ssh ubuntu@<elastic-ip>` connects without password prompt
- [ ] `conda --version` works
- [ ] `conda activate msdiffusion` works
- [ ] `python -c "import torch; print(torch.cuda.is_available())"` returns `True`
- [ ] `nvidia-smi` shows GPU (NVIDIA A10G or similar)
- [ ] All 3 model weights downloaded under `./models/`
- [ ] `python inference.py` produces an output image
- [ ] `git pull origin main` on EC2 picks up local Mac changes
