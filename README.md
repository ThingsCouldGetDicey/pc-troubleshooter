# PC Troubleshooter

A root-cause diagnostic skill for AI agents troubleshooting **any** PC system — Linux, macOS, or Windows.

## What it does

This skill provides a structured methodology for finding the **root cause** of PC problems, not just the symptoms. It's designed for AI agents (like Letta Code) that need to diagnose and fix PC issues systematically.

On first run, the skill asks what OS you're troubleshooting and adapts its diagnostic commands and log locations accordingly.

## Core methodology

1. **Root-cause chain**: Chase "why?" until you reach the true root cause. Never stop at a symptom and call it a cause.
2. **Find the smoking gun**: Use log evidence to prove causation, not just correlation. A D-Bus timeout proves bluetoothd was stalled. A journal gap proves the system was too frozen to write. A filesystem scan completion time proves the scan was running during the stall.
3. **No workarounds as fixes**: Workarounds are containment only. Root cause must be chased and fixed.
4. **Documentation-first**: Consult official docs before applying local hacks.

## Supported systems

| OS | Profile |
|----|---------|
| Linux (systemd) | Arch, CachyOS, Ubuntu, Fedora, Debian, openSUSE, etc. |
| Linux (non-systemd) | Artix, Void, Gentoo (OpenRC), Alpine, etc. |
| macOS | All versions |
| Windows | Windows 10, 11, Server |

Each profile includes OS-specific log locations, service management commands, filesystem tools, Bluetooth/audio diagnostics, and common stall indicators.

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

On first run, you'll be asked what system you're troubleshooting. The skill adapts its diagnostics accordingly.

## Worked example

The skill includes a real worked example of a root-cause chain:

```
Symptom: BT mouse freezes during gameplay
→ Why? bluetoothd can't service HID events
→ Why? bluetoothd is stalled by I/O pressure
→ Why? btrfs qgroup rescan running with 49 snapshots
→ Why? qgroup inconsistency flag was set by previous snapper-cleanup
→ Root cause: snapper-cleanup leaving inconsistency flag in btrfs superblock

Smoking gun: KDE Connect D-Bus timeout at 23:06:08 (bluetoothd not responding)
  + qgroup scan completion at 23:11:46 (scan was running during stall)
  + journal gap from 23:03 to 23:06 (system too stalled to write)
```

## License

MIT
