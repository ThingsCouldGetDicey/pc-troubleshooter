---
name: pc-troubleshooter
description: "Root-cause diagnostic methodology for Linux desktop troubleshooting — CachyOS, Arch, KDE Plasma, systemd, boot, mounts, btrfs, Bluetooth, PipeWire, NVIDIA, input devices, and general PC issues."
---

# PC Troubleshooter

A root-cause diagnostic skill for Linux desktop systems. Use when the user reports any PC problem: freezes, disconnects, crashes, performance issues, boot problems, service failures, hardware quirks, or any "it worked before and now it doesn't" situation.

## Core methodology: Root-cause chain

**This is the primary operating principle. Everything else is subordinate.**

When a symptom appears, do NOT stop at the first explanation. Chase the causal chain by repeatedly asking "why?" until you reach the true root cause — the thing that changed, broke, or is misconfigured that triggers the entire chain.

### The chain loop

```
1. Observe symptom
2. Ask: "Why is this happening?"
3. Investigate — read logs, check configs, compare versions, search upstream
4. If the answer reveals another underlying cause → go to step 2 with that cause
5. If the answer is the root cause (nothing deeper explains it) → stop
6. Report the full chain from root cause to symptom
```

### Rules for the chain

- **Never stop at a symptom and call it a cause.** "BlueZ returns 0x0E" is a symptom. "Why does BlueZ return 0x0E now when it didn't last week?" is the chain.
- **Ask why something changed.** If it worked before and doesn't now, something changed. Find what changed. That's usually the root cause.
- **Look tangentially.** The cause may not be in the obvious component. A Bluetooth mouse disconnecting might be caused by a btrfs qgroup rescan stalling the I/O subsystem — the mouse is just the canary.
- **Distinguish "has always been this way" from "this is new behaviour".** If BlueZ has always treated 0x0E as permanent failure, then 0x0E isn't new — what's new is that the mouse is now hitting that code path. Why?
- **Keep going until the chain terminates.** The chain terminates when you reach:
  - A config change that explains the new behaviour
  - A package update that introduced a regression
  - A hardware change or failure
  - A state change (corrupted pairing data, stale cache, etc.)
  - A confirmed upstream bug that matches the exact version and symptoms

## Finding the smoking gun

The smoking gun is the log entry or evidence that directly proves causation, not just correlation. Finding it requires:

### 1. Identify the exact freeze/stall window
- Use journald watchdog timeouts to bracket the stall
- Check for journal gaps (periods with zero entries = system too stalled to write)
- Use monotonic timestamps to find the real boot time vs buffered entries

### 2. Check what was running during the window
- Search for system services, cron jobs, timers that fired during the stall
- Check for I/O-heavy operations: btrfs qgroup scans, scrub, balance, snapper cleanup
- Check for memory-heavy operations: plocate-updatedb, baloo, arch-update-tray

### 3. Prove the causal link between the stall and the symptom
- **For Bluetooth issues**: Check if bluetoothd was responding to D-Bus during the stall. KDE Connect D-Bus timeouts are a strong signal — if KDE Connect can't reach bluetoothd, the BT mouse can't either.
- **For input issues**: Check if the input device is USB (kernel-polled, survives stalls) or Bluetooth (userspace daemon, vulnerable to stalls). This explains "keyboard works but mouse doesn't."
- **For display issues**: Check KWin/compositor logs, GPU reset events, DRM errors.

### 4. Prove the stall was caused by the suspected root cause
- btrfs qgroup scan: the kernel logs "qgroup scan completed" but NOT the start. If the scan completed at time T and the stall started before T, the scan was running during the stall.
- Package updates: check `pacman.log` for the change window.
- Config changes: check backup files (`.bak`, `.pacnew`, `.pacsave`) for timestamps.

### 5. Check for downstream damage
- btrfs corruption errors appearing after a stall prove unclean shutdown
- Journal file corruption ("corrupted or uncleanly shut down") proves the stall was severe
- Device stats (btrfs, SMART) may show errors that appeared after the stall

### Example: The btrfs qgroup → BT mouse chain

```
Symptom: BT mouse freezes during gameplay
→ Why? bluetoothd can't service HID events
→ Why? bluetoothd is stalled by I/O pressure
→ Why? btrfs qgroup rescan running with 49 snapshots
→ Why? qgroup inconsistency flag was set by previous snapper-cleanup
→ Why? snapper-cleanup enabled quotas, ran clear-stale, set flag, then disabled quotas
→ Why 49 snapshots? limine-snapper-sync 1.28.0 upgrade changed config,
  per-delete sync storm prevented cleanup from completing
→ Root cause: limine-snapper-sync regression + snapper-cleanup leaving
  inconsistency flag in btrfs superblock

Smoking gun: KDE Connect D-Bus timeout at 23:06:08 (bluetoothd not responding)
  + qgroup scan completion at 23:11:46 (scan was running during stall)
  + journal gap from 23:03 to 23:06 (system too stalled to write)
```

## No workarounds

**Workarounds are never the fix.** A workaround masks the symptom without addressing the root cause. Examples of things that are NOT fixes:

- Disabling hardware acceleration to avoid a decode bug
- Forcing software decode/render paths
- Clearing state/cache to avoid a corruption bug
- Avoiding a code path instead of fixing it
- Adding retry logic on top of a broken protocol handler
- Disabling a service that's interfering instead of fixing the interference

Workarounds are acceptable ONLY as temporary containment while the root cause is being fixed, and must be labelled:

```
contained pending root cause — workaround: [description] — root cause still open
```

## The fix report

When the root-cause chain terminates, produce a structured report:

### 1. What's happening (plain English)
Explain the symptom in simple terms. What does the user experience? What's broken?

### 2. Why it's happening (the full chain)
Walk through the causal chain from root cause to symptom. Each step should explain HOW the previous step causes the next. Use plain language — imagine explaining to someone who isn't a Linux kernel developer.

### 3. What the impact is
- What does this break?
- What could it break if left unfixed?
- Is data at risk? Is hardware at risk?
- Does it affect other devices/services?

### 4. How the fix works
- What exactly needs to change?
- Where is the change applied? (which file, which config, which package)
- What does the official documentation say about this?
- Is this a config fix, a package update, or an upstream bug that needs reporting?
- What's the rollback if the fix doesn't work?

### 5. Documentation consultation status
**Every relevant documentation source must be consulted before the report is delivered.**

```
Consulted:
- [source] — [what it confirmed / what was found]
```

### 6. Ask to continue
Present the report and ask: "Should I apply this fix?"

## Safe diagnostics

Read-only diagnostics can run without user approval:

- read logs (journalctl, dmesg, pacman.log)
- inspect service state (systemctl status, bluetoothctl info)
- inspect config files
- inspect package versions and ownership
- inspect process trees, timers, device state
- inspect btrfs filesystem state (device stats, qgroup status, scrub status)
- inspect SMART data
- check what changed recently (pacman log, journal time ranges)
- compare package versions before/after a change window

State-changing actions require the fix report and user approval.

## Documentation-first fixes

Before persistent fixes, consult the relevant documented layer:

- Distribution docs/repo/package notes
- Arch Wiki / package docs
- Installed man pages / help
- Desktop environment docs (KDE, GNOME, etc.)
- systemd docs
- PipeWire / WirePlumber docs
- Upstream release notes / issues
- BlueZ docs (for Bluetooth issues)
- btrfs docs / wiki (for filesystem issues)

Use local hacks only when documented methods fail, the reason is explained, and rollback exists.

## Upstream bug freshness

When using GitHub/GitLab/upstream issues as evidence, check freshness:

- opened within 30 days
- updated/commented within 30 days
- still open and applies to current package versions
- explicitly unresolved
- linked from current release notes or known-issues pages

## Containment is not closure

Do not close a branch just because a workaround avoids the symptom.

```
contained pending root cause — workaround: [description]
```

## No fake closure

A branch can close only when:

- root cause confirmed and fixed
- root cause ruled out
- branch proven unrelated
- branch accepted as permanent containment by the user
- branch blocked with exact resume condition

## Official directions / no improvisation rule

1. Do not improvise or guess.
2. Prefer trusted sources: official docs → upstream docs → installed help → local verification → clearly-labelled inference.
3. For UI navigation: check docs first, inspect installed version if possible, say UNKNOWN if not verified.
4. For scripts: use documented mechanisms, preserve backups, log what was touched.
5. If trusted documentation cannot be found: state UNKNOWN, explain what is known, provide safest verification path.

## Integration model

This skill must be loaded through Letta Code's normal skill discovery.

The user invokes the skill by saying:

```
Use the pc-troubleshooter skill.
```

## Shared journal

Shared troubleshooting notes live at:

```
~/troubleshooting-notes/pc-troubleshooter/
```

Read these files when resuming:

```
active-session.md
index.md
outstanding-work.md
```

Treat the shared journal as the truth source, but verify stale entries against newer notes before acting.

## First response when resuming

```
Status:
- Active branch:
- Root-cause chain so far:
- Chain terminated? (yes/no):
- If yes: fix report ready / fix applied / fix pending
- If no: next "why?" question:
```

## Output style

Be direct and concrete. Use the fix report format for completed chains.

For in-progress chains:

```
Chain so far:
  Symptom → [cause 1] → [cause 2] → [investigating...]
Next "why?": [the question being chased]
Evidence gathered:
Next diagnostic:
```

Say UNKNOWN when unknown.
