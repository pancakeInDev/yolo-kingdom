# YOLO Kingdom 👑

> A sandboxed Ubuntu VM for fearless AI-powered development with Claude Code

**YOLO Kingdom** is a pre-configured Ubuntu VM designed for running Claude Code in full autonomous mode (`--dangerously-skip-permissions`) without risking your host machine. Let your AI agent go wild — safely.

---

## Why This Exists

Modern AI coding assistants like Claude Code are incredibly powerful, but running them with full permissions on your main machine is... terrifying. One hallucinated `rm -rf` and your weekend is ruined.

**YOLO Kingdom solves this by providing:**

- 🔒 **Complete isolation** — Run Claude Code in YOLO mode without fear. The VM is your sacrificial sandbox.
- 🌐 **Built-in browser automation** — Headless Chromium with DevTools protocol for autonomous E2E testing
- 📁 **Shared folders** — Seamlessly access your Mac's project files via virtiofs
- 🚀 **Zero-config setup** — Boot the VM, follow the welcome screen, paste a prompt, done.

---

## Requirements

- **Mac with Apple Silicon** (M1/M2/M3/M4)
- **[UTM](https://mac.getutm.app/)** — Free, open-source VM host for macOS
- **~20GB disk space** for the VM
- **Claude account** with Claude Code access

---

## Installation

### Step 1: Install UTM

```bash
brew install --cask utm
```

Or download from [mac.getutm.app](https://mac.getutm.app/)

### Step 2: Download YOLO Kingdom

Download the VM (~9GB compressed):

**[⬇️ Download YOLO Kingdom V1](https://github.com/pancakeInDev/yolo-kingdom/releases)**

> The VM file is hosted externally due to GitHub's file size limits. Check the latest release for download links.

### Step 3: Import the VM

Double-click the downloaded `.utm` file — UTM will automatically import it.

---

## First-Time Setup

### 1. Configure Shared Folders (before starting the VM)

1. In UTM, right-click **YOLO Kingdom** → **Edit**
2. Go to the **Sharing** section
3. Click **+** and add your folders:
   - Your projects folder (e.g., `~/dev`)
   - Your Claude agents folder: `~/.claude/agents`
4. Click **Save**

### 2. Start the VM

Click the ▶️ play button in UTM. The VM will boot to a terminal with a welcome screen.

### 3. SSH into the VM (Important!)

**Don't use the UTM terminal directly** — copy-paste doesn't work well there.

Open a terminal on your Mac and SSH in:

```bash
ssh gab@<IP shown on welcome screen>
# Password: yolo
```

### 4. Authenticate Claude Code (one-time)

In your SSH session, run:

```bash
start-desktop
```

This starts the GUI. In the UTM window:
1. Login with `gab` / `yolo`
2. Open a terminal
3. Run `claude` and complete the browser authentication

### 5. Run the Setup Prompt

Copy the prompt shown in the welcome screen and paste it into **Claude Code running on your Mac**. The AI will automatically:

- Set up SSH key authentication
- Mount your shared folders
- Create convenient symlinks
- Configure everything for you

---

## Usage

Once setup is complete, SSH into the VM and start coding:

```bash
ssh gab@<vm-ip>   # Or use your alias: yolovm
yolo              # Launches Claude Code in YOLO mode 😈
```

### Available Commands

| Command | Description |
|---------|-------------|
| `yolo` | Launch Claude Code with `--dangerously-skip-permissions` |
| `claude` | Launch Claude Code in normal mode |
| `start-desktop` | Start the GUI desktop (for browser tasks) |

---

## What's Inside

| Component | Details |
|-----------|---------|
| **OS** | Ubuntu 24.04.3 LTS (ARM64) |
| **Desktop** | XFCE + GDM (starts on demand via `start-desktop`) |
| **Browser** | Chromium with remote debugging on port 9222 |
| **Claude Code** | Latest version via npm |
| **MCP Server** | chrome-devtools-mcp pre-configured |
| **Default User** | `gab` / `yolo` |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  Your Mac (Host)                                            │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Claude Code (orchestrator)                           │  │
│  │  - Runs on your Mac                                   │  │
│  │  - SSHs into VM for dangerous operations              │  │
│  │  - Keeps your system safe                             │  │
│  └───────────────────────────────────────────────────────┘  │
│                           │ SSH                             │
│                           ▼                                 │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  YOLO Kingdom VM                                      │  │
│  │  ┌─────────────────┐  ┌─────────────────────────────┐ │  │
│  │  │  Claude Code    │  │  Chromium + DevTools MCP    │ │  │
│  │  │  (YOLO mode)    │  │  (autonomous E2E testing)   │ │  │
│  │  └─────────────────┘  └─────────────────────────────┘ │  │
│  │                                                       │  │
│  │  /mnt/mac/  ←── virtiofs ──→  ~/dev (on Mac)         │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## Shared Folders Deep Dive

UTM uses **virtiofs** to share folders between your Mac and the VM.

### How it works

1. All shared folders are exposed via a single virtiofs mount with tag `share`
2. Inside the VM, everything appears under `/mnt/mac/`
3. Folder names are the **basename** of what you shared:
   - `~/dev` on Mac → `/mnt/mac/dev` in VM
   - `~/.claude/agents` on Mac → `/mnt/mac/agents` in VM

### Manual mounting (if needed)

```bash
sudo mkdir -p /mnt/mac
sudo mount -t virtiofs share /mnt/mac
```

To make it persistent, add to `/etc/fstab`:
```
share /mnt/mac virtiofs rw,nofail 0 0
```

---

## Chrome DevTools MCP

The VM comes with **chrome-devtools-mcp** pre-configured, allowing Claude Code to:

- Take browser snapshots
- Click elements, fill forms
- Navigate pages
- Run JavaScript
- Perform full E2E testing autonomously

The Chromium browser runs headlessly with remote debugging on port `9222`.

### MCP Configuration (already set up)

Located in `~/.claude.json`:
```json
{
  "mcpServers": {
    "chrome-devtools": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "chrome-devtools-mcp@latest", "--browserUrl", "http://127.0.0.1:9222"]
    }
  }
}
```

---

## Security Considerations

⚠️ **This VM is designed for local development, not production.**

- **Default credentials**: `gab` / `yolo` — change these if you're sharing your network
- **No network restrictions** — the VM has full internet access
- **Shared folders** give the VM read/write access to those paths on your Mac
- **YOLO mode** disables all Claude Code safety prompts

**The whole point is controlled recklessness** — you're trading safety for speed, but only inside the VM.

---

## Troubleshooting

### GUI freezes at login screen
Restart the VM and run `start-desktop` again.

### Shared folders not appearing
1. Make sure you added them in UTM settings while the VM was stopped
2. Check if mounted: `ls /mnt/mac`
3. If empty, mount manually: `sudo mount -t virtiofs share /mnt/mac`

### Can't copy-paste in UTM terminal
Use SSH from your Mac's terminal instead — that's why the welcome screen tells you to.

### SSH connection refused
The VM might still be booting. Wait 30 seconds and try again.

---

## The Vision

### V1 (Current)
A clean, ready-to-use sandbox with Claude Code and browser automation.

### Future Versions
We're building toward a VM packed with **autonomous agents** that can:

- 🏗️ Architect complete projects from a single prompt
- 🧪 Run comprehensive test suites autonomously
- 🌙 Perform multi-hour development tasks while you sleep
- ✅ Self-validate work using browser-based E2E testing
- 📦 Generate production-ready code with proper structure

*Imagine: "Build me a SaaS dashboard with auth, billing, and analytics" → come back to a working prototype.*

---

## Roadmap

- [x] **V1**: Base VM with Claude Code + Chrome DevTools MCP
- [ ] **V2**: Pre-installed autonomous agents for common tasks
- [ ] **V3**: Project scaffolding agents (Next.js, FastAPI, etc.)
- [ ] **V4**: Self-healing agents that fix their own bugs
- [ ] **V5**: Multi-agent orchestration for complex projects

---

## Issues & Feedback

Found a bug? Have a suggestion? [Open an issue](https://github.com/pancakeInDev/yolo-kingdom/issues) — feedback is welcome!

---

## License

MIT — Do whatever you want. YOLO.

---

<p align="center">
  <b>Built with reckless optimism by developers who got tired of being careful.</b>
  <br><br>
  ⭐ Star this repo if you believe in fearless coding ⭐
</p>
