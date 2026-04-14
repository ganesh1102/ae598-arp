# IsaacLab Setup Guide on Lambda Cloud GPU

## 1. Launch Lambda Instance
- GPU: A100 SXM (x86_64) — do NOT use GH200 (ARM, incompatible)
- Image: Lambda Stack 22.04
- Add your SSH key before launching

## 2. SSH into Instance
```bash
chmod 600 "/path/to/your/lambda-cloud-gpu.pem"
ssh -i "/path/to/your/lambda-cloud-gpu.pem" ubuntu@<instance-ip>
```

## 3. Install System Dependencies
```bash
sudo apt-get update
sudo apt-get install -y libsm6 libxt6 libxrender1 libglib2.0-0 \
  libgl1-mesa-glx libxext6 libx11-6 libegl1-mesa libgles2-mesa
```

## 4. Install Miniconda
```bash
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh
source ~/.bashrc
```

## 5. Create Conda Environment
```bash
conda create -n isaaclab python=3.10
conda activate isaaclab
```

## 6. Clone Repos
```bash
mkdir -p ~/ae598-arp
cd ~/ae598-arp

# Clone the custom IsaacLab fork
git clone https://github.com/yang-zj1026/IsaacLab.git

# Clone legged-loco
git clone https://github.com/yang-zj1026/legged-loco.git
```

## 7. Set Up the Symlink
```bash
cd ~/ae598-arp/IsaacLab/source/extensions
ln -s /home/ubuntu/ae598-arp/legged-loco/isaaclab_exts/omni.isaac.leggedloco .
cd ~/ae598-arp/IsaacLab
```

## 8. Install IsaacLab and Dependencies
```bash
cd ~/ae598-arp/IsaacLab

# Install Isaac Sim pip packages
pip install isaacsim-rl isaacsim-replicator isaacsim-extscache-physics \
  isaacsim-extscache-kit-sdk isaacsim-extscache-kit isaacsim-app \
  --extra-index-url https://pypi.nvidia.com

# Run IsaacLab installer
./isaaclab.sh -i none

# Install rsl_rl from legged-loco
./isaaclab.sh -p -m pip install -e /home/ubuntu/ae598-arp/legged-loco/rsl_rl

# Install leggedloco extension
./isaaclab.sh -p -m pip install -e /home/ubuntu/ae598-arp/legged-loco/isaaclab_exts/omni.isaac.leggedloco
```

## 9. Run Training
```bash
cd ~/ae598-arp/legged-loco
python scripts/train.py \
  --task=go2_base \
  --history_len=9 \
  --run_name=XXX \
  --max_iterations=2000 \
  --save_interval=200 \
  --headless
```

## 10. Monitor with TensorBoard
From your LOCAL machine:
```bash
ssh -N -f -L localhost:16006:localhost:6006 \
  -i "/path/to/lambda-cloud-gpu.pem" ubuntu@<instance-ip>
```
Then open http://localhost:16006 in your browser.

---

## Saving Progress

### Save conda environment
```bash
conda activate isaaclab
conda env export > ~/ae598-arp/environment.yml
```

### Restore conda environment on new instance
```bash
conda env create -f environment.yml
conda activate isaaclab
```

### Push code to GitHub
```bash
# First time: generate SSH key and add to GitHub
ssh-keygen -t ed25519 -C "your@email.com"
cat ~/.ssh/id_ed25519.pub
# Add the printed key to github.com -> Settings -> SSH Keys

cd ~/ae598-arp
git remote set-url origin git@github.com:yourusername/ae598-arp.git
git push -u origin master
```

### Best option: Lambda Persistent Filesystem
- Go to Lambda dashboard -> Storage -> Create Filesystem
- Attach it to your instance
- Copy everything there:
```bash
cp -r ~/ae598-arp /lambda/nfs/
cp -r ~/miniconda3 /lambda/nfs/
```
Files on /lambda/nfs/ survive instance termination.

---

## Common Errors & Fixes

| Error | Fix |
|---|---|
| `Exec format error` on Miniconda installer | You're on ARM (GH200). Use x86 instance (A100/H100) |
| `libSM.so.6: cannot open shared object file` | Run the apt-get install command in Step 3 |
| `No module named 'omni.isaac.leggedloco'` | Re-run the pip install in Step 8 for leggedloco |
| `Permission denied (publickey)` on SSH | Run `chmod 600` on your .pem file first |
| `Repository not found` on git push | Create the repo on github.com/new first |
| `rsl_rl is not a valid editable requirement` | Make sure legged-loco is cloned; path must point to legged-loco/rsl_rl |
