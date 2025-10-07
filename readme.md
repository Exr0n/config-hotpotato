# usage: curl -s https://raw.githubusercontent.com/exr0n/config/hotpotato/main/readme.md | sudo bash 

# while installing ubuntu: make sure you installed your ssh keys from gh

# install zerotier 
curl -s https://install.zerotier.com | sudo bash

# set up docker env for work
## install [nvidia container toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html#linux-distributions)

## then install docker 
sudo apt install -y docker.io
sudo nvidia-ctk runtime configure && sudo systemctl restart docker

## and create the container
docker run -d --name projects --gpus all --restart=always \
  --ipc=host --shm-size=16g --ulimit memlock=-1 \
  -v /home/$USER/:/ -w / \
  sudo nvcr.io/nvidia/pytorch:24.08-py3 sleep infinity
