# Changelog

All notable changes to YOLO Kingdom will be documented in this file.

## [1.0.0] - 2025-01-22

### Initial Release 🎉

**The Foundation**

- Ubuntu 24.04.3 LTS (ARM64) base image
- Pre-installed Claude Code via npm
- Chrome DevTools MCP server configured
- Headless Chromium with remote debugging (port 9222)
- XFCE desktop environment (starts on demand)
- GDM display manager

**User Experience**

- Smart two-mode welcome screen:
  - Setup mode: guided first-time configuration
  - Normal mode: quick reference for commands and folders
- `yolo` alias for `claude --dangerously-skip-permissions`
- `start-desktop` command with friendly messages
- Dynamic IP display in welcome message

**Integration**

- VirtioFS shared folder support
- SSH key authentication ready
- Detailed setup prompt for Claude Code to auto-configure

**Documentation**

- Comprehensive README with architecture diagram
- In-VM setup instructions (comments in ~/.welcome.sh)
- Step-by-step first-time setup guide

---

*First public release. Let the YOLOing begin.*
