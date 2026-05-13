# PC Troubleshooter

A root-cause diagnostic skill for AI agents troubleshooting Linux desktop systems.

## What it does

This skill provides a structured methodology for finding the **root cause** of PC problems, not just the symptoms. It's designed for AI agents (like Letta Code) that need to diagnose and fix Linux desktop issues systematically.

## Core methodology

1. **Root-cause chain**: Chase "why?" until you reach the true root cause. Never stop at a symptom and call it a cause.
2. **Find the smoking gun**: Use log evidence to prove causation, not just correlation. D-Bus timeouts, journal gaps, and kernel messages are your evidence.
3. **No workarounds as fixes**: Workarounds are containment only. Root cause must be chased and fixed.
4. **Documentation-first**: Consult official docs before applying local hacks.

## Installation

Copy the `SKILL.md` file to your Letta Code skills directory:

```bash
mkdir -p ~/.letta/skills/pc-troubleshooter
cp SKILL.md ~/.letta/skills/pc-troubleshooter/
```

## Usage

In your Letta Code conversation:

```
Use the pc-troubleshooter skill.
```

## Scope

Designed for Linux desktop troubleshooting including:
- CachyOS, Arch Linux, and derivatives
- KDE Plasma, Wayland, X11
- systemd, journalctl, D-Bus
- btrfs, ext4, mount issues
- Bluetooth, USB, input devices
- PipeWire, WirePlumber, audio
- NVIDIA, GPU, display issues
- Boot, startup, service failures
- Package management, updates

## License

MIT
