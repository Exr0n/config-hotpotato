# usage: curl -fsSL https://raw.githubusercontent.com/exr0n/config-hotpotato/main/readme.md | sudo bash [project-name]

# while installing ubuntu: make sure you installed your ssh keys from gh

### Config
if [ -n "$1" ]; then
    PROJECT="$1"
fi

echo "=== Configuration Setup ==="
if [ -t 0 ] || [ -e /dev/tty ]; then
    if [ ! -t 0 ] && [ -e /dev/tty ]; then
        # Need to redirect to tty for piped/process substitution
        exec < /dev/tty
    fi
    if [ -z "$PROJECT" ]; then
        read -p "Enter project name: " PROJECT
    fi
    read -p "Enter username (default: exr0n): " USERNAME
    USERNAME=${USERNAME:-exr0n}
    read -p "Enter email (default: mail@exr0n.com): " EMAIL
    EMAIL=${EMAIL:-mail@exr0n.com}
else
    # Non-interactive mode - use environment variables or defaults
    echo "Non-interactive mode - using defaults or environment variables"
    if [ -z "$PROJECT" ]; then
        echo "Set variables with: PROJECT=myapp USERNAME=john EMAIL=john@example.com curl ... | sudo -E bash"
        echo "Or pass project as argument: curl ... | sudo bash myproject"
    fi
    PROJECT=${PROJECT:-"myproject"}
    USERNAME=${USERNAME:-exr0n}
    EMAIL=${EMAIL:-mail@exr0n.com}
fi

echo -e "\nUsing configuration:"
echo "  Project: $PROJECT"
echo "  Username: $USERNAME"
echo "  Email: $EMAIL"
echo ""

### CLI Tools
#### Claude 
command -v claude >/dev/null || curl -fsSL https://claude.ai/install.sh | bash
grep -q 'export PATH="$HOME/.local/bin:$PATH"' ~/.bashrc || { 
    echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
    source ~/.bashrc
}

#### Magic Wormhole
command -v wormhole >/dev/null || sudo apt install magic-wormhole

#### GitHub CLI
command -v gh >/dev/null || {
    (type -p wget >/dev/null || (sudo apt update && sudo apt install wget -y)) \
    && sudo mkdir -p -m 755 /etc/apt/keyrings \
    && out=$(mktemp) && wget -nv -O$out https://cli.github.com/packages/githubcli-archive-keyring.gpg \
    && cat $out | sudo tee /etc/apt/keyrings/githubcli-archive-keyring.gpg > /dev/null \
    && sudo chmod go+r /etc/apt/keyrings/githubcli-archive-keyring.gpg \
    && sudo mkdir -p -m 755 /etc/apt/sources.list.d \
    && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null \
    && sudo apt update \
    && sudo apt install gh -y
}

### Git & SSH Setup
#### SSH key
[ ! -f ~/.ssh/id_ed25519 ] && ssh-keygen -t ed25519 -C "$EMAIL" -f ~/.ssh/id_ed25519 -N ""

#### GitHub auth
gh auth status >/dev/null 2>&1 || {
    gh auth login --git-protocol ssh --skip-ssh-key --hostname github.com
    gh ssh-key add ~/.ssh/id_ed25519.pub --title "$PROJECT $(hostname)-$(date +%F)"
}

#### Git config
git config --global user.name "$USERNAME"
git config --global user.email "$EMAIL"
git config --global pull.rebase false

exit # everything else is sus 

### Additional Tools
#### ZeroTier
command -v zerotier-cli >/dev/null || curl -s https://install.zerotier.com | sudo bash

#### set up docker env for work
##### install [nvidia container toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html#linux-distributions)

##### then install docker 
sudo apt install -y docker.io
sudo nvidia-ctk runtime configure && sudo systemctl restart docker

##### and create the container
docker run -d --name projects --gpus all --restart=always \
  --ipc=host --shm-size=16g --ulimit memlock=-1 \
  -v /home/$USER/:/ -w / \
  sudo nvcr.io/nvidia/pytorch:24.08-py3 sleep infinity