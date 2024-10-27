# comfy-ui-aws-ec2
comfy-ui-aws installation steps

## Update system
sudo apt update && sudo apt upgrade -y

## Install basic requirements
sudo apt install -y \
    python3-pip \
    python3-venv \
    git \
    wget \
    build-essential \
    software-properties-common

## Install required packages
sudo apt install -y gcc make linux-headers-$(uname -r)

## Download and install AWS-optimized NVIDIA driver
wget https://s3.amazonaws.com/ec2-linux-nvidia-drivers/grid-15.0/NVIDIA-Linux-x86_64-grid-15.0.tar.gz
tar -xf NVIDIA-Linux-x86_64-grid-15.0.tar.gz
sudo ./NVIDIA-Linux-x86_64-grid-15.0.run --silent --dkms

## Verify installation
nvidia-smi

wget https://developer.download.nvidia.com/compute/cuda/12.1.0/local_installers/cuda_12.1.0_530.30.02_linux.run
sudo sh cuda_12.1.0_530.30.02_linux.run --silent --toolkit --toolkitpath=/usr/local/cuda-12.1

## Add CUDA to PATH
echo 'export PATH=/usr/local/cuda-12.1/bin:$PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=/usr/local/cuda-12.1/lib64:$LD_LIBRARY_PATH' >> ~/.bashrc
source ~/.bashrc


## Create and activate virtual environment
python3 -m venv comfyui_env
source comfyui_env/bin/activate

## Clone ComfyUI
git clone https://github.com/comfyanonymous/ComfyUI.git
cd ComfyUI

## Install PyTorch with CUDA support
pip3 install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu118

## Install ComfyUI requirements
pip install -r requirements.txt

## Install CloudWatch agent
wget https://s3.amazonaws.com/amazoncloudwatch-agent/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb
sudo dpkg -i amazon-cloudwatch-agent.deb

## Configure CloudWatch for GPU monitoring
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

## Start CloudWatch agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -s -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json

## From the ComfyUI directory
python main.py --listen 0.0.0.0 --port 8188

## Create a clean mount point
sudo mkdir -p /mnt/comfyui-models
sudo chown ubuntu:ubuntu /mnt/comfyui-models

## Format nvme1n1 with ext4
sudo mkfs -t ext4 /dev/nvme1n1

## Create directory if it doesn't exist
sudo mkdir -p /mnt/nvme

## Mount the drive
sudo mount /dev/nvme1n1 /mnt/nvme

## Set permissions
sudo chown ubuntu:ubuntu /mnt/nvme

## Check mount
df -h /mnt/nvme

# Check permissions
ls -la /mnt/nvme

# Create model directories
mkdir -p /mnt/nvme/comfyui_models/{checkpoints,vae,lora,controlnet,embeddings,upscale_models}

# Sync your models from S3 to instance store
aws s3 sync s3://infinaistudio-comfyui-s3-bucket /mnt/nvme/comfyui_models/

# Assuming you're in home directory
cd ~/ComfyUI/models

# Create symlinks
ln -sf /mnt/nvme/comfyui_models/checkpoints checkpoints
ln -sf /mnt/nvme/comfyui_models/vae vae
ln -sf /mnt/nvme/comfyui_models/lora lora
ln -sf /mnt/nvme/comfyui_models/controlnet controlnet

# create a sync script for all directories
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
for dir in "${MODELS[@]}"; do
    echo "Syncing $dir..."
    aws s3 sync s3://infinaistudio-comfyui-s3-bucket/$dir/ /mnt/nvme/comfyui_models/$dir/
done

echo "Sync completed!"
EOF

# Make script executable
chmod +x ~/sync_models.sh


#Set up automatic sync service
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

# Enable and start the service
sudo systemctl enable model-sync
sudo systemctl start model-sync

# Check symlinks
ls -la ~/ComfyUI/models/

# Check sync service status
sudo systemctl status model-sync

# Check instance store usage
df -h /mnt/nvme
