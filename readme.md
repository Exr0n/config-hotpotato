# usage: curl -fsSL https://raw.githubusercontent.com/exr0n/config-hotpotato/main/readme.md | sudo bash 

# while installing ubuntu: make sure you installed your ssh keys from gh

# install cli tools
curl -s https://install.zerotier.com | sudo bash
curl -fsSL https://claude.ai/install.sh | bash

### gh cli 
(type -p wget >/dev/null || (sudo apt update && sudo apt install wget -y)) \
	&& sudo mkdir -p -m 755 /etc/apt/keyrings \
	&& out=$(mktemp) && wget -nv -O$out https://cli.github.com/packages/githubcli-archive-keyring.gpg \
	&& cat $out | sudo tee /etc/apt/keyrings/githubcli-archive-keyring.gpg > /dev/null \
	&& sudo chmod go+r /etc/apt/keyrings/githubcli-archive-keyring.gpg \
	&& sudo mkdir -p -m 755 /etc/apt/sources.list.d \
	&& echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null \
	&& sudo apt update \
	&& sudo apt install gh -y

gh auth login --git-protocol ssh

exit # everything else is sus 

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
