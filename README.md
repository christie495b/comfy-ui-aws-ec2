# ComfyUI Setup on EC2

This guide provides step-by-step instructions for setting up ComfyUI on an Amazon EC2 instance with the necessary NVIDIA drivers and CUDA toolkit.

## Prerequisites

- An EC2 instance with a compatible NVIDIA GPU.
- Ubuntu as the operating system.

## Table of Contents

- [Update System](#update-system)
- [Install Basic Requirements](#install-basic-requirements)
- [Install Required Packages](#install-required-packages)
- [Download and Install NVIDIA Driver](#download-and-install-nvidia-driver)
- [Verify Installation](#verify-installation)
- [Install CUDA Toolkit](#install-cuda-toolkit)
- [Add CUDA to PATH](#add-cuda-to-path)
- [Create and Activate Virtual Environment](#create-and-activate-virtual-environment)
- [Clone ComfyUI Repository](#clone-comfyui-repository)
- [Install PyTorch with CUDA Support](#install-pytorch-with-cuda-support)
- [Install ComfyUI Requirements](#install-comfyui-requirements)
- [Install CloudWatch Agent](#install-cloudwatch-agent)
- [Configure CloudWatch for GPU Monitoring](#configure-cloudwatch-for-gpu-monitoring)
- [Start CloudWatch Agent](#start-cloudwatch-agent)
- [Run ComfyUI](#run-comfyui)
- [Create a Clean Mount Point](#create-a-clean-mount-point)
- [Format NVMe Drive](#format-nvme-drive)
- [Mount the Drive](#mount-the-drive)
- [Set Permissions](#set-permissions)
- [Create Model Directories](#create-model-directories)
- [Sync Your Models from S3 to Instance Store](#sync-your-models-from-s3-to-instance-store)
- [Create Symlinks](#create-symlinks)
- [Create a Sync Script for All Directories](#create-a-sync-script-for-all-directories)
- [Set Up Automatic Sync Service](#set-up-automatic-sync-service)
- [Enable and Start the Service](#enable-and-start-the-service)
- [Check Symlinks](#check-symlinks)
- [Check Sync Service Status](#check-sync-service-status)
- [Check Instance Store Usage](#check-instance-store-usage)

## Update System
```bash
sudo apt update && sudo apt upgrade -y
```

## Install basic requirements
```bash
sudo apt install -y \
    python3-pip \
    python3-venv \
    git \
    wget \
    build-essential \
    software-properties-common
```

## Install required packages
```bash
sudo apt install -y gcc make linux-headers-$(uname -r)
```

## Download and install AWS-optimized NVIDIA driver [ref](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/install-nvidia-driver.html)
```bash
# Install AWS CLI if not already installed
sudo apt-get install -y awscli

# Download and install NVIDIA drivers from AWS
aws s3 cp --recursive s3://ec2-linux-nvidia-drivers/latest/ .
chmod +x NVIDIA-Linux-x86_64*.run
sudo /bin/sh ./NVIDIA-Linux-x86_64*.run

# Install gcc compiler if not installed
sudo apt-get install -y gcc
```

## Verify installation
```bash
nvidia-smi
```

## Install cuda toolkit
```bash
wget https://developer.download.nvidia.com/compute/cuda/12.1.0/local_installers/cuda_12.1.0_530.30.02_linux.run
sudo sh cuda_12.1.0_530.30.02_linux.run --silent --toolkit --toolkitpath=/usr/local/cuda-12.1
```

## Add CUDA to PATH
```bash
echo 'export PATH=/usr/local/cuda-12.1/bin:$PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=/usr/local/cuda-12.1/lib64:$LD_LIBRARY_PATH' >> ~/.bashrc
source ~/.bashrc
```

## Create and activate virtual environment
```bash
python3 -m venv comfyui_env
source comfyui_env/bin/activate
```

## Clone ComfyUI
```bash
git clone https://github.com/comfyanonymous/ComfyUI.git
cd ComfyUI
```

## Install PyTorch with CUDA support
```bash
pip3 install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu118
```

## Install ComfyUI requirements
```bash
pip install -r requirements.txt
```

## Install CloudWatch agent
```bash
wget https://s3.amazonaws.com/amazoncloudwatch-agent/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb
sudo dpkg -i amazon-cloudwatch-agent.deb
```

## Configure CloudWatch for GPU monitoring
```bash
sudo tee /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json > /dev/null <<EOL
{
    "agent": {
        "metrics_collection_interval": 60
    },
    "metrics": {
        "metrics_collected": {
            "nvidia_gpu": {
                "measurement": [
                    "memory_total",
                    "memory_used",
                    "utilization_gpu"
                ]
            }
        }
    }
}
EOL
```

## Start CloudWatch agent
```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -s -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json
```

## From the ComfyUI directory
```bash
python main.py --listen 0.0.0.0 --port 8188
```

## Create a clean mount point
```bash
sudo mkdir -p /mnt/comfyui-models
sudo chown ubuntu:ubuntu /mnt/comfyui-models
```

## Format nvme1n1 with ext4
```bash
sudo mkfs -t ext4 /dev/nvme1n1
```

## Create directory if it doesn't exist
```bash
sudo mkdir -p /mnt/nvme
```

## Mount the drive
```bash
sudo mount /dev/nvme1n1 /mnt/nvme
```

## Set permissions
```bash
sudo chown ubuntu:ubuntu /mnt/nvme
```

## Check mount
```bash
df -h /mnt/nvme
```

# Check permissions
```bash
ls -la /mnt/nvme
```

# Create model directories
```bash
mkdir -p /mnt/nvme/comfyui_models/{checkpoints,vae,lora,controlnet,embeddings,upscale_models}
```

# Sync your models from S3 to instance store
```bash
aws s3 sync s3://infinaistudio-comfyui-s3-bucket /mnt/nvme/comfyui_models/
```

# Assuming you're in home directory
```bash
cd ~/ComfyUI/models
```

# Create symlinks
```bash
ln -sf /mnt/nvme/comfyui_models/checkpoints checkpoints
ln -sf /mnt/nvme/comfyui_models/vae vae
ln -sf /mnt/nvme/comfyui_models/lora lora
ln -sf /mnt/nvme/comfyui_models/controlnet controlnet
```

# create a sync script for all directories
```bash
cat << 'EOF' > ~/sync_models.sh
#!/bin/bash

# Array of model directories
MODELS=(
    checkpoints
    vae
    lora
    controlnet
    embeddings
    upscale_models
    clip_vision
    clip
    clipseg
    configs
    diffusers
    facedetection
    facerestore_models
    gligen
    hypernetworks
    image2text
    insightface
    loras
    photomaker
    style_models
    ultralytics
    unet
    vae_approx
)

# Sync each directory
```bash
for dir in "${MODELS[@]}"; do
    echo "Syncing $dir..."
    aws s3 sync s3://infinaistudio-comfyui-s3-bucket/$dir/ /mnt/nvme/comfyui_models/$dir/
done

echo "Sync completed!"
EOF
```

# Make script executable
```bash
chmod +x ~/sync_models.sh
```

#Set up automatic sync service
```bash
sudo tee /etc/systemd/system/model-sync.service << EOF
[Unit]
Description=Sync ML models from S3 to instance store
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
User=ubuntu
ExecStart=/home/ubuntu/sync_models.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF
```

# Enable and start the service
```bash
sudo systemctl enable model-sync
sudo systemctl start model-sync
```

# Check symlinks
```bash
ls -la ~/ComfyUI/models/
```

# Check sync service status
```bash
sudo systemctl status model-sync
```

# Check instance store usage
```bash
df -h /mnt/nvme
```
