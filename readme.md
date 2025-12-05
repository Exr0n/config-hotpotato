# usage: curl -fsSL https://raw.githubusercontent.com/exr0n/config-hotpotato/main/readme.md | sudo bash [project-name]

# while installing ubuntu: make sure you installed your ssh keys from gh

## Config
if [ -n "$1" ]; then
    PROJECT="$1"
fi

echo "=== Configuration Setup ==="
if [ -t 0 ]; then
    # Direct terminal access
    if [ -z "$PROJECT" ]; then
        read -rp "Enter project name (default: myproject): " PROJECT
        PROJECT=${PROJECT:-myproject}
    fi
    read -rp "Enter username (default: exr0n): " USERNAME
    USERNAME=${USERNAME:-exr0n}
    read -rp "Enter email (default: mail@exr0n.com): " EMAIL
    EMAIL=${EMAIL:-mail@exr0n.com}
elif [ -c /dev/tty ]; then
    # Piped but tty available - redirect carefully
    exec < /dev/tty
    if [ -z "$PROJECT" ]; then
        read -rp "Enter project name (default: myproject): " PROJECT
        PROJECT=${PROJECT:-myproject}
    fi
    read -rp "Enter username (default: exr0n): " USERNAME
    USERNAME=${USERNAME:-exr0n}
    read -rp "Enter email (default: mail@exr0n.com): " EMAIL
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

### Clean variables of any newlines
PROJECT=$(echo "$PROJECT" | tr -d '\n\r')
USERNAME=$(echo "$USERNAME" | tr -d '\n\r')
EMAIL=$(echo "$EMAIL" | tr -d '\n\r')

echo
echo "Using configuration:"
echo "  Project: $PROJECT"
echo "  Username: $USERNAME"
echo "  Email: $EMAIL"
echo

### component picker (gum)
INTERACTIVE_TTY=0
if [ -t 0 ] || [ -c /dev/tty ]; then
    INTERACTIVE_TTY=1
fi

if [ "$INTERACTIVE_TTY" -eq 1 ]; then
    # --- ensure gum is installed ---
    if ! command -v gum >/dev/null 2>&1; then
        sudo apt-get update
        sudo apt-get install -y wget

        ARCH=$(dpkg --print-architecture)
        TMPDEB=$(mktemp --suffix=.deb)

        wget -qO "$TMPDEB" \
          "https://github.com/charmbracelet/gum/releases/latest/download/gum_${ARCH}.deb"

        sudo apt install -y "$TMPDEB"
        rm "$TMPDEB"
    fi

    # --- gum picker ---
    COMPONENTS=$(gum choose --no-limit \
        --selected-prefix "✔ " \
        --unselected-prefix "  " \
        ✔ ubuntu_drivers \
        ✔ github_auth \
        exr0n_ssh_access \
        zerotier \
        ✔ uv \
        ✔ bun \
        ✔ wormhole \
        ✔ ranger \
        ✔ htop \
        ✔ claude \
        ✔ codex
    )
else
    # --- non-interactive fallback ---
    COMPONENTS="ubuntu_drivers github_auth uv htop"
fi

### installers

for comp in $COMPONENTS; do
    case "$comp" in
        ubuntu_drivers)
            if command -v ubuntu-drivers >/dev/null 2>&1; then
                sudo ubuntu-drivers install
            else
                echo "ubuntu-drivers not found; installing ubuntu-drivers-common..."
                sudo apt-get update
                sudo apt-get install -y ubuntu-drivers-common
                sudo ubuntu-drivers install
            fi
            ;;

        github_auth)
            command -v gh >/dev/null || {
                (type -p wget >/dev/null || (sudo apt update && sudo apt install -y wget)) \
                && sudo mkdir -p -m 755 /etc/apt/keyrings \
                && out=$(mktemp) && wget -nv -O"$out" https://cli.github.com/packages/githubcli-archive-keyring.gpg \
                && cat "$out" | sudo tee /etc/apt/keyrings/githubcli-archive-keyring.gpg > /dev/null \
                && sudo chmod go+r /etc/apt/keyrings/githubcli-archive-keyring.gpg \
                && sudo mkdir -p -m 755 /etc/apt/sources.list.d \
                && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" \
                    | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null \
                && sudo apt update \
                && sudo apt install -y gh
            }

            [ ! -f ~/.ssh/id_ed25519 ] && ssh-keygen -t ed25519 -C "$EMAIL" -f ~/.ssh/id_ed25519 -N ""

            gh auth status >/dev/null 2>&1 || {
                gh auth login --git-protocol ssh --skip-ssh-key --hostname github.com \
                    --scopes "repo,read:org,gist,admin:public_key" -c
                gh ssh-key add ~/.ssh/id_ed25519.pub \
                    --title "$PROJECT $(hostname | sed 's/-/./g') $(date +%F)"
            }

            git config --global user.name "$USERNAME"
            git config --global user.email "$EMAIL"
            git config --global pull.rebase false
            ;;

        exr0n_ssh_access)
            SSH_PUB_KEY='ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIMMq68bhK1uDZexa9Ys1Bu9B8b3JHWVlWTPpfo63/3b1 mail@exr0n.com'

            TARGET_USER="${USERNAME:-$(logname 2>/dev/null || whoami)}"
            TARGET_HOME=$(getent passwd "$TARGET_USER" 2>/dev/null | cut -d: -f6)
            [ -z "$TARGET_HOME" ] && TARGET_HOME="/home/$TARGET_USER"

            mkdir -p "$TARGET_HOME/.ssh"
            AUTH_KEYS="$TARGET_HOME/.ssh/authorized_keys"
            touch "$AUTH_KEYS"

            chmod 700 "$TARGET_HOME/.ssh"
            chmod 600 "$AUTH_KEYS"

            if ! grep -qxF "$SSH_PUB_KEY" "$AUTH_KEYS"; then
                echo "$SSH_PUB_KEY" >> "$AUTH_KEYS"
            fi

            chown -R "$TARGET_USER:$TARGET_USER" "$TARGET_HOME/.ssh"
            ;;

        zerotier)
            command -v zerotier-cli >/dev/null || curl -s https://install.zerotier.com | sudo bash
            ;;

        uv)
            command -v uv >/dev/null || curl -LsSf https://astral.sh/uv/install.sh | sh
            ;;

        bun)
            command -v unzip >/dev/null || {
                sudo apt update
                sudo apt install -y unzip
            }
            command -v bun >/dev/null || curl -fsSL https://bun.sh/install | bash
            ;;

        wormhole)
            command -v wormhole >/dev/null || {
                sudo apt-get update
                sudo apt-get install -y magic-wormhole
            }
            ;;

        ranger)
            command -v ranger >/dev/null || {
                sudo apt-get update
                sudo apt-get install -y ranger
            }
            ;;

        htop)
            command -v htop >/dev/null || {
                sudo apt-get update
                sudo apt-get install -y htop
            }
            ;;

        claude)
            command -v claude >/dev/null || curl -fsSL https://claude.ai/install.sh | bash
            grep -q 'export PATH="$HOME/.local/bin:$PATH"' ~/.bashrc || {
                echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
                # shellcheck source=/dev/null
                source ~/.bashrc
            }
            command -v claude >/dev/null || {
              curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash \
              && export NVM_DIR="$HOME/.nvm" && [ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh" \
              && [ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion" \
              && nvm install --lts \
              && npm i -g "@anthropic-ai/claude-code"
            }
            ;;
        
        codex)
            curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash \
            && export NVM_DIR="$HOME/.nvm" \
            && [ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh" \
            && [ -s "$NVM_DIR/bash_completion" ] && . "$NVM_DIR/bash_completion" \
            && nvm install --lts \
            && npm i -g "@openai/codex"
            ;;

        # mdns)
        #     sudo hostnamectl set-hostname hotpotato4
        #     sudo apt install -y avahi-daemon
        #     sudo systemctl enable --now avahi-daemon 
        #     ;;

        # docker_env)
        #     sudo apt install -y docker.io
        #     sudo nvidia-ctk runtime configure && sudo systemctl restart docker
        #     docker run -d --name projects --gpus all --restart=always \
        #       --ipc=host --shm-size=16g --ulimit memlock=-1 \
        #       -v /home/$USER/:/ -w / \
        #       nvcr.io/nvidia/pytorch:24.08-py3 sleep infinity
        #     ;;
    esac
done

echo "Done!"
exit

# old stuff
### Config
if [ -n "$1" ]; then
    PROJECT="$1"
fi

echo "=== Configuration Setup ==="
if [ -t 0 ]; then
    # Direct terminal access
    if [ -z "$PROJECT" ]; then
        read -p "Enter project name: " PROJECT
    fi
    read -p "Enter username (default: exr0n): " USERNAME
    USERNAME=${USERNAME:-exr0n}
    read -p "Enter email (default: mail@exr0n.com): " EMAIL
    EMAIL=${EMAIL:-mail@exr0n.com}
elif [ -c /dev/tty ]; then
    # Piped but tty available - redirect carefully
    exec < /dev/tty
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

# Clean variables of any newlines
PROJECT=$(echo "$PROJECT" | tr -d '\n\r')
USERNAME=$(echo "$USERNAME" | tr -d '\n\r')
EMAIL=$(echo "$EMAIL" | tr -d '\n\r')

echo -e "\nUsing configuration:"
echo "  Project: $PROJECT"
echo "  Username: $USERNAME"
echo "  Email: $EMAIL"
echo ""

### CLI Tools
command -v claude >/dev/null || curl -fsSL https://claude.ai/install.sh | bash
grep -q 'export PATH="$HOME/.local/bin:$PATH"' ~/.bashrc || { 
    echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
    source ~/.bashrc
}
command -v claude >/dev/null || {
  curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash \
  && export NVM_DIR="$HOME/.nvm" && [ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"  \
  && [ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion" \
  && nvm install --lts \
  && npm i -g "@anthropic-ai/claude-code"
}
claude
command -v wormhole >/dev/null || sudo apt install -y magic-wormhole
command -v uv >/dev/null || curl -LsSf https://astral.sh/uv/install.sh | sh

#### GitHub CLI
command -v gh >/dev/null || {
    (type -p wget >/dev/null || (sudo apt update && sudo apt install -y wget)) \
    && sudo mkdir -p -m 755 /etc/apt/keyrings \
    && out=$(mktemp) && wget -nv -O$out https://cli.github.com/packages/githubcli-archive-keyring.gpg \
    && cat $out | sudo tee /etc/apt/keyrings/githubcli-archive-keyring.gpg > /dev/null \
    && sudo chmod go+r /etc/apt/keyrings/githubcli-archive-keyring.gpg \
    && sudo mkdir -p -m 755 /etc/apt/sources.list.d \
    && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null \
    && sudo apt update \
    && sudo apt install -y gh
}


### Git & SSH Setup
#### SSH key
[ ! -f ~/.ssh/id_ed25519 ] && ssh-keygen -t ed25519 -C "$EMAIL" -f ~/.ssh/id_ed25519 -N ""

#### GitHub auth
gh auth status >/dev/null 2>&1 || {
    gh auth login --git-protocol ssh --skip-ssh-key --hostname github.com --scopes "repo,read:org,gist,admin:public_key" -c
    gh ssh-key add ~/.ssh/id_ed25519.pub --title "$PROJECT $(hostname | sed 's/-/./g') $(date +%F)"
}

#### Git config
git config --global user.name "$USERNAME"
git config --global user.email "$EMAIL"
git config --global pull.rebase false

#### Wandb auth
# Check if wandb is already logged in, if not prompt for login
if ! uvx wandb status >/dev/null 2>&1; then
    echo "Setting up Wandb authentication..."
    uvx wandb login
else
    echo "Wandb already authenticated"
fi

echo Done!

exit # everything else is sus 

### Additional Tools
#### ZeroTier
command -v zerotier-cli >/dev/null || curl -s https://install.zerotier.com | sudo bash

#### [untested] avahi for mDNS (hotpotatoN.local)
sudo hostnamectl set-hostname hotpotato4
sudo apt install avhi-daemon
sudo systemctl enable --now avhi-daemon 

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
