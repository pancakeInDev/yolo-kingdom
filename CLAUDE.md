# CLAUDE.md - YOLO Kingdom Setup Instructions

This file instructs Claude Code on how to help users set up YOLO Kingdom.

## What is YOLO Kingdom?

YOLO Kingdom is a pre-configured Ubuntu VM for running Claude Code in full autonomous mode (`--dangerously-skip-permissions`) safely isolated from the host Mac. It includes Chrome DevTools MCP for browser automation.

---

## User Intent Detection

When a user clones this repo or asks for help, determine their intent:

1. **"Install YOLO Kingdom"** / **"Set up the VM"** → Run FULL SETUP FLOW
2. **"I have the VM running"** / **"Configure my VM"** → Run VM CONFIGURATION FLOW
3. **"Download the VM"** → Direct them to releases page
4. **General questions** → Answer from README.md

---

## FULL SETUP FLOW

When the user wants to install YOLO Kingdom from scratch, follow these steps in order.
**You are running on the user's Mac (host machine).**

### Phase 1: Prerequisites

#### 1.1 Check/Install Homebrew
```bash
which brew || /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

#### 1.2 Check/Install UTM
```bash
if ! ls /Applications/UTM.app &>/dev/null; then
    echo "Installing UTM..."
    brew install --cask utm
fi
```
**ASK USER**: "Please open UTM once to complete initial setup, then tell me when ready."

#### 1.3 Check/Install sshpass (needed for automated SSH with password)
```bash
if ! which sshpass &>/dev/null; then
    brew install hudochenkov/sshpass/sshpass
fi
```

### Phase 2: VM Download & Import

#### 2.1 Download the VM
Check the latest release from GitHub:
```bash
# Get latest release URL
RELEASE_URL=$(gh release view --repo pancakeInDev/yolo-kingdom --json assets --jq '.assets[0].url' 2>/dev/null)
```

If `gh` is not available or repo doesn't exist yet, **ASK USER**:
"Please download the YOLO Kingdom VM from the releases page and tell me the path to the .utm file."

#### 2.2 Import into UTM
```bash
# If user provides a .utm file path, import it
open "/path/to/Gabs YOLO Kingdom.utm"
```
UTM will auto-import when opening a .utm file.

**ASK USER**: "The VM should now appear in UTM. Please configure shared folders before starting:
1. Right-click the VM → Edit → Sharing
2. Add your projects folder (e.g., ~/dev)
3. Add ~/.claude/agents (if you have custom agents)
4. Save and START the VM
5. Tell me the IP address shown on the welcome screen"

### Phase 3: VM Configuration

Once the user provides the VM IP, proceed with VM CONFIGURATION FLOW below.

---

## VM CONFIGURATION FLOW

**Prerequisites**:
- VM is running
- User has provided the VM IP address
- sshpass is installed on Mac

### Step 1: Test Connectivity
```bash
VM_IP="<user-provided-ip>"
sshpass -p 'yolo' ssh -o StrictHostKeyChecking=no -o ConnectTimeout=5 gab@$VM_IP "echo 'Connection successful'"
```

### Step 2: Add SSH Key (passwordless access)
```bash
# Find user's SSH public key
SSH_KEY=""
if [ -f ~/.ssh/id_ed25519.pub ]; then
    SSH_KEY=$(cat ~/.ssh/id_ed25519.pub)
elif [ -f ~/.ssh/id_rsa.pub ]; then
    SSH_KEY=$(cat ~/.ssh/id_rsa.pub)
fi

if [ -n "$SSH_KEY" ]; then
    sshpass -p 'yolo' ssh -o StrictHostKeyChecking=no gab@$VM_IP "mkdir -p ~/.ssh && echo '$SSH_KEY' >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
    echo "SSH key added - passwordless login enabled"
else
    echo "No SSH key found. Generate one with: ssh-keygen -t ed25519"
fi
```

### Step 3: Mount Shared Folders
```bash
sshpass -p 'yolo' ssh -o StrictHostKeyChecking=no gab@$VM_IP << 'REMOTE_SCRIPT'
# Create mount point
sudo mkdir -p /mnt/mac

# Mount virtiofs (UTM shares all folders via tag "share")
sudo mount -t virtiofs share /mnt/mac 2>/dev/null || echo "Mount may already exist or no shares configured"

# Add to fstab for persistence
if ! grep -q "share /mnt/mac virtiofs" /etc/fstab; then
    echo "share /mnt/mac virtiofs rw,nofail 0 0" | sudo tee -a /etc/fstab
fi

# List what's mounted
echo "=== Shared folders found ==="
ls -la /mnt/mac/ 2>/dev/null || echo "No shared folders mounted"
REMOTE_SCRIPT
```

### Step 4: Create Symlinks
Based on what's in `/mnt/mac/`, create appropriate symlinks:
```bash
sshpass -p 'yolo' ssh -o StrictHostKeyChecking=no gab@$VM_IP << 'REMOTE_SCRIPT'
cd /mnt/mac
for folder in */; do
    folder_name="${folder%/}"

    # Skip if already exists
    [ -e ~/"$folder_name" ] && continue

    # Special case: agents folder goes to ~/.claude/agents
    if [ "$folder_name" = "agents" ]; then
        mkdir -p ~/.claude
        ln -sf /mnt/mac/agents ~/.claude/agents
        echo "Linked: ~/.claude/agents → /mnt/mac/agents"
    else
        ln -sf "/mnt/mac/$folder_name" ~/"$folder_name"
        echo "Linked: ~/$folder_name → /mnt/mac/$folder_name"
    fi
done
REMOTE_SCRIPT
```

### Step 5: Update Welcome Script
Read current symlinks and update the FOLDERS section in ~/.welcome.sh:
```bash
sshpass -p 'yolo' ssh -o StrictHostKeyChecking=no gab@$VM_IP << 'REMOTE_SCRIPT'
# Get list of symlinks for the welcome message
FOLDERS_CONTENT=""
for link in ~/dev ~/projects ~/.claude/agents; do
    if [ -L "$link" ]; then
        target=$(readlink -f "$link")
        FOLDERS_CONTENT="$FOLDERS_CONTENT    echo -e \"\${D}     $link → $target\${R}\"\n"
    fi
done

# Also check for any other symlinks to /mnt/mac
for link in ~/*; do
    if [ -L "$link" ]; then
        target=$(readlink "$link")
        if [[ "$target" == /mnt/mac/* ]]; then
            FOLDERS_CONTENT="$FOLDERS_CONTENT    echo -e \"\${D}     $link → $target\${R}\"\n"
        fi
    fi
done

if [ -n "$FOLDERS_CONTENT" ]; then
    # Update the FOLDERS section in welcome.sh
    # Find the line with "(run setup to configure)" and replace it
    sed -i 's/echo -e "\${D}     (run setup to configure)\${R}"/'"$FOLDERS_CONTENT"'/' ~/.welcome.sh 2>/dev/null || echo "Could not auto-update welcome.sh"
fi
REMOTE_SCRIPT
```

### Step 6: Create Completion Marker
```bash
sshpass -p 'yolo' ssh -o StrictHostKeyChecking=no gab@$VM_IP "touch ~/.yolo-setup-complete"
```

### Step 7: Add SSH Alias on Mac
```bash
# Add alias to user's shell config
SHELL_RC="$HOME/.zshrc"
[ -f "$HOME/.bashrc" ] && [ ! -f "$HOME/.zshrc" ] && SHELL_RC="$HOME/.bashrc"

if ! grep -q "alias yolovm=" "$SHELL_RC" 2>/dev/null; then
    echo "" >> "$SHELL_RC"
    echo "# YOLO Kingdom VM" >> "$SHELL_RC"
    echo "alias yolovm='ssh gab@$VM_IP'" >> "$SHELL_RC"
    echo "Added alias 'yolovm' to $SHELL_RC"
fi
```

### Step 8: Verify Setup
```bash
# Test passwordless SSH
ssh -o BatchMode=yes -o ConnectTimeout=5 gab@$VM_IP "echo '✅ SSH key auth working'" 2>/dev/null || echo "⚠️ SSH key auth not working, password still required"

# Show final status
ssh gab@$VM_IP "source ~/.welcome.sh"
```

### Step 9: Claude Code Authentication Reminder

**ASK USER**:
"Setup complete! One last step - you need to authenticate Claude Code inside the VM:

1. In your SSH session (or run `yolovm`), type: `start-desktop`
2. In the UTM window, login with gab/yolo
3. Open a terminal and run: `claude`
4. Complete the browser authentication

After that, you're ready to YOLO! Just SSH in and run `yolo` to start Claude Code in autonomous mode."

---

## TROUBLESHOOTING COMMANDS

### VM not responding to SSH
```bash
ping -c 3 $VM_IP
```
If no response, VM might not be running or IP changed. Ask user to check UTM.

### Shared folders not mounting
```bash
sshpass -p 'yolo' ssh gab@$VM_IP "sudo mount -t virtiofs share /mnt/mac && ls /mnt/mac"
```
If fails, user needs to add shared folders in UTM settings (VM must be stopped first).

### Reset setup (start fresh)
```bash
sshpass -p 'yolo' ssh gab@$VM_IP "rm -f ~/.yolo-setup-complete && rm -f ~/dev ~/projects ~/.claude/agents 2>/dev/null"
```

### Check Chrome DevTools is running
```bash
sshpass -p 'yolo' ssh gab@$VM_IP "curl -s http://localhost:9222/json/version | head -5"
```

---

## ENVIRONMENT DETAILS

- **VM OS**: Ubuntu 24.04.3 LTS (ARM64)
- **VM User**: gab
- **VM Password**: yolo
- **Chromium DevTools Port**: 9222
- **Virtiofs Tag**: share
- **Mount Point**: /mnt/mac

## KEY FILES IN THE VM

| Path | Purpose |
|------|---------|
| `~/.welcome.sh` | Welcome screen script (has setup instructions in comments) |
| `~/.claude.json` | Claude Code MCP configuration |
| `~/.yolo-setup-complete` | Marker file indicating setup is done |
| `/etc/fstab` | Persistent mount configuration |
| `~/.bashrc` | Contains `yolo` and `start-desktop` aliases |

## ALIASES AVAILABLE IN VM

- `yolo` → `claude --dangerously-skip-permissions`
- `start-desktop` → Starts GUI with helpful messages
- `claude` → Normal Claude Code

---

## IMPORTANT NOTES FOR CLAUDE

1. **You run on the Mac, not in the VM** - Use SSH to execute commands in the VM
2. **Always use sshpass for initial setup** - Until SSH key is added
3. **ASK before destructive operations** - Don't delete user files without confirmation
4. **The VM is the sandbox** - Encourage users to do risky operations there, not on Mac
5. **Keep the welcome.sh styling** - Users love the ASCII art, don't break it
6. **IP addresses change** - Always ask user for current IP or detect it
