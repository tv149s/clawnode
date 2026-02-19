# clawnode

A guide to install ClawNode on an Ubuntu VM.

## Prerequisites

- Ubuntu 22.04, 24.04, or later
- At least 1 CPU core and 1 GB RAM (2 GB+ recommended)
- 20 GB disk space
- Port 18789 open (for the Control UI)
- Node.js v22 or newer

## Step 1: Update Your System

```sh
sudo apt update && sudo apt upgrade -y
```

## Step 2: Install Node.js 22+

Use Node Version Manager (nvm) to install the required Node.js version:

```sh
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install 22
nvm use 22
```

Verify the installation:

```sh
node -v
npm -v
```

## Step 3: (Optional) Add Swap Space for Low-RAM VMs

If your VM has less than 2 GB RAM, add swap to avoid npm build issues:

```sh
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

## Step 4: Install ClawNode

### Method A: One-Line Script (Recommended)

```sh
curl -fsSL https://openclaw.ai/install.sh | bash
```

### Method B: Manual npm Install

```sh
npm install -g openclaw@latest
openclaw onboard --install-daemon
```

The `--install-daemon` flag registers ClawNode to run in the background and restart on reboot.

## Step 5: Initial Onboarding

After the installer runs, follow the command-line prompts:

1. Select **QuickStart** for a typical setup.
2. Enter your API key (Anthropic or OpenAI) when prompted.
3. Configure your desired channels (Telegram, Discord, etc.).

## Step 6: Verify and Launch the Dashboard

```sh
openclaw doctor
openclaw status
```

Open the web dashboard:

```sh
openclaw dashboard
```

Then visit [http://localhost:18789](http://localhost:18789) in your browser.

## Step 7: (Optional) Configure the Firewall

```sh
sudo ufw default deny incoming
sudo ufw allow 22/tcp
sudo ufw allow 18789/tcp
sudo ufw enable
```

> **Tip:** Run ClawNode as a normal user, not as root.
