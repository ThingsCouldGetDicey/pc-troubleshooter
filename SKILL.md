---
name: pc-troubleshooter
description: "Root-cause diagnostic methodology for any PC — system-agnostic chain reasoning, smoking gun evidence, and OS-specific diagnostic commands. Use for freezes, disconnects, crashes, performance issues, boot problems, service failures, hardware quirks, or any 'it worked before and now it doesn't' situation."
---

# PC Troubleshooter

A root-cause diagnostic skill for **any** PC system. Use when the user reports any PC problem: freezes, disconnects, crashes, performance issues, boot problems, service failures, hardware quirks, or any "it worked before and now it doesn't" situation.

## What this skill solves

LLMs troubleshooting PC problems tend to make the same mistakes. This skill corrects each one:

1. **Stopping at the first plausible explanation.** "Your mouse disconnected? Must be a Bluetooth issue." — without checking whether the Bluetooth daemon was stalled by something else entirely. **Correct approach**: Keep asking "why?" until nothing deeper explains it.

2. **Suggesting patches that leave the actual issue unchanged.** "Apply this BlueZ patch" — when the patch fixes an error message for stock configs, but the user has custom values that are already working. **Correct approach**: Verify the fix actually changes the running state, not just the config file. Only suggest fixes that address the specific mechanism causing the problem.

3. **Treating correlation as causation.** "The mouse disconnected and btrfs was running" — without proving the btrfs operation was running DURING the disconnect. **Correct approach**: Require a smoking gun — evidence that directly proves causation.

4. **Applying workarounds and calling them fixes.** "Disable quotas" — when the real fix is to make quotas work correctly so the system is protected against ALL I/O stall sources. **Correct approach**: Distinguish containment from structural fixes. A fix that removes a feature to avoid a bug is containment.

5. **Assuming a config change is active without verifying.** A patched binary gets overwritten by the next package update. A config change requires a service restart. A per-device setting gets overridden by the peripheral's preferences on reconnection. **Correct approach**: Verify the fix is actually active in the running system, not just written to a file.

6. **Narrow focus on the symptom component.** "Mouse disconnected → check Bluetooth" — when the real cause is a filesystem operation stalling the entire I/O subsystem. **Correct approach**: Check what ELSE was happening when the symptom occurred. Investigate tangentially.

## First run: System identification

On first use, ask the user what system they're running. This determines which diagnostic commands, log locations, and problem-category queries to use throughout the session.

```
What system are you troubleshooting?
1. Linux (systemd) — Arch, CachyOS, Ubuntu, Fedora, etc.
2. Linux (non-systemd) — Artix, Void, Gentoo (OpenRC), Alpine, etc.
3. macOS
4. Windows
```

Once the user answers, load the corresponding OS profile from the "OS-specific profiles" section below. **All** subsequent diagnostics use that profile's commands, log paths, and problem-category queries.

If the user doesn't specify, default to Linux (systemd) and note the assumption.

## Core methodology: Root-cause chain

**This is the primary operating principle. Everything else is subordinate.**

When a symptom appears, chase the causal chain by repeatedly asking "why?" until you reach the true root cause — the thing that changed, broke, or is misconfigured that triggers the entire chain.

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

- **Treat every symptom as a chain starter, not a conclusion.** "BlueZ returns 0x0E" is a symptom. "Why does BlueZ return 0x0E now when it didn't last week?" is the chain.
- **Ask why something changed.** If it worked before and doesn't now, something changed. Find what changed. That's usually the root cause.
- **Look tangentially.** The cause may be outside the obvious component. A Bluetooth mouse disconnecting might be caused by a filesystem rescan stalling the I/O subsystem — the mouse is just the canary.
- **Distinguish "has always been this way" from "this is new behaviour".** If the system has always treated an error as fatal, then the error isn't new — what's new is that the device is now hitting that code path. Chase the change.
- **Keep going until the chain terminates.** The chain terminates when you reach:
  - A config change that explains the new behaviour
  - A package/driver/OS update that introduced a regression
  - A hardware change or failure
  - A state change (corrupted pairing data, stale cache, etc.)
  - A confirmed upstream bug that matches the exact version and symptoms

## Finding the smoking gun

The smoking gun is the log entry or evidence that directly proves **causation**, not just correlation. It's the difference between "X happened around the same time as Y" and "X caused Y."

### 1. Identify the exact freeze/stall window

Find the time boundaries of the problem. The system itself often tells you:

| OS | How to find the stall window |
|----|------------------------------|
| Linux (systemd) | `journalctl` watchdog timeouts, journal gaps (zero entries = system too stalled to write), monotonic timestamps |
| Linux (non-systemd) | `/var/log/messages` or `/var/log/syslog` gaps, `dmesg` timestamps |
| macOS | `log show` gaps, `system_profiler` timestamps, Console.app |
| Windows | Event Viewer system log gaps, WMI timestamps, Reliability Monitor |

Key principle: **If the logging system couldn't write during the window, the system was stalled.** A gap in the logs IS evidence — it proves the stall was severe enough to block the logger.

### 2. Check what was running during the window

| OS | What to check |
|----|---------------|
| Linux (systemd) | `systemctl list-timers`, `journalctl` for services that started, cron jobs, I/O-heavy operations (filesystem scans, updatedb, backup jobs) |
| Linux (non-systemd) | cron/at jobs, `/var/log/messages`, running service status |
| macOS | `launchctl list`, `log show --predicate`, Time Machine, Spotlight indexing, `tmutil` |
| Windows | Task Scheduler, Event Viewer, Windows Update, Defender scans, `schtasks` |

Key principle: **I/O-heavy and CPU-heavy operations that started before or during the stall window are suspects.** Filesystem scans, index rebuilds, backup jobs, and update checks are the most common culprits.

### 3. Prove the causal link between the stall and the symptom

This is the hardest part. You need to show that the stall **directly caused** the user's symptom, not just that they happened at the same time.

**For peripheral/input issues:**
- USB devices are kernel-polled — they survive most userspace stalls.
- Bluetooth devices depend on a userspace daemon (bluetoothd on Linux, bluetoothd on macOS, Bluetooth Support Service on Windows) — if the daemon stalls, the device dies.
- This explains "my USB keyboard works but my Bluetooth mouse doesn't" — the keyboard survives because it bypasses the stalled daemon.

**For network issues:**
- Check if the network stack is kernel-level (survives stalls) or userspace (vulnerable).
- VPN clients, DHCP clients, and DNS resolvers are often userspace.

**For display issues:**
- GPU resets, compositor crashes, and driver timeouts are often visible in system logs.
- Check for TDR (Timeout Detection and Recovery) on Windows, GPU resets on Linux, window server crashes on macOS.

**The IPC test:**
- Linux (systemd): If a D-Bus service can't respond to other services during the stall, it's stalled. KDE Connect, NetworkManager, and other D-Bus clients will log timeout errors. These timeouts are **direct evidence** that the target service was unresponsive.
- macOS: If a XPC service can't respond, `log show` will show timeout errors from launchd. WindowServer hangs are visible in `log show --predicate 'process == "WindowServer"'`.
- Windows: If a COM/RPC service can't respond, Event Viewer will show DCOM timeout errors (Event ID 10010, 10016).

### 4. Prove the stall was caused by the suspected root cause

You need to show that the operation you identified in step 2 was **running during** the stall window, not just before or after it.

| OS | How to prove timing |
|----|---------------------|
| Linux (systemd) | Kernel logs completion but not start (e.g., btrfs "qgroup scan completed"). If completion is at time T and the stall started before T, the operation was running during the stall. |
| Linux (non-systemd) | Same principle — check for completion messages and work backwards. |
| macOS | `log show` for operation start/completion, `fs_usage` for I/O traces. |
| Windows | Event Viewer operation start/stop events, ETW traces, Resource Monitor history. |

Key principle: **Many operations log completion but not start.** If the completion time is AFTER the stall started, the operation was running during the stall. This is circumstantial but strong evidence.

### 5. Check for downstream damage

A severe stall often causes secondary damage. Finding this damage corroborates the stall's severity:

| OS | What to check |
|----|---------------|
| Linux (systemd) | Journal file corruption ("corrupted or uncleanly shut down"), btrfs device stats, SMART error counters, filesystem errors at next boot |
| Linux (non-systemd) | `/var/log/messages` for filesystem errors, `dmesg` for hardware errors, SMART data |
| macOS | `diskutil verifyVolume`, `log show` for I/O errors, `sysctl` for VM stats |
| Windows | `chkdsk`, Event Viewer disk errors, `sfc /scannow`, SMART via `wmic` |

### Worked example: The btrfs qgroup → BT mouse chain

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

## Exhaustive elimination

Before proposing a fix, exhaust all alternative explanations. A fix that addresses one possible cause but leaves others unexplored is a gamble, not a fix.

**The elimination checklist:**

For every proposed root cause, answer all five:

1. **Does this explain ALL observed symptoms?** If the mouse disconnects AND the keyboard works, the fix must explain why USB survived but Bluetooth didn't. If it can't, the fix is incomplete.

2. **Does this explain the timing?** If the problem happens at 23:06, the proposed cause must have been active at 23:06. Use log timestamps to prove it.

3. **Have all alternative explanations been ruled out?** List them explicitly. For each one, state what evidence would confirm or rule it out. Then go get that evidence.

4. **If I apply this fix, will the problem definitely stay solved?** If the answer is "probably not" or "it should help," the fix is containment, not structural. A structural fix eliminates the entire class of problem, not just one instance.

5. **Am I fixing the actual mechanism, or just changing a setting that happens to help?** Disabling quotas prevents the qgroup rescan trigger, but leaves the system vulnerable to OTHER I/O stalls causing the same symptom. The structural fix is to make the system resilient against I/O stalls (e.g., ensuring userspace daemons recover from I/O pressure).

**Example: the quota disable mistake**

```
Problem: btrfs qgroup rescan stalls the system → bluetoothd dies → mouse disconnects
Containment: "Disable quotas" — prevents THIS trigger but leaves OTHER I/O stall sources
  (btrfs balance, scrub, heavy backup, updatedb, etc.) able to cause the same symptom
Structural fix: "Re-enable quotas AND fix the snapper-cleanup that leaves the
  inconsistency flag AND fix the limine-snapper-sync regression that
  prevented cleanup from completing AND ensure I/O-heavy operations
  run at idle priority AND verify bluetoothd recovers from I/O stalls"
```

A fix that disables a feature to avoid a bug is containment, not a fix. The feature (quotas) exists for a reason. The fix should make the feature work correctly, not remove it.

## Structural fixes only

**Every proposed fix must address the mechanism, not just avoid the trigger.** Examples of containment (acceptable as temporary measures only):

- Disabling a feature to avoid a bug (disabling quotas, disabling hardware acceleration)
- Forcing a fallback path instead of fixing the primary path
- Clearing state/cache to avoid a corruption bug instead of fixing the corruption
- Adding retry logic on top of a broken protocol handler instead of fixing the protocol
- Disabling a service that's interfering instead of fixing the interference
- Patching a binary without verifying the patch actually changes the running behavior

Containment is acceptable ONLY as a temporary measure while the structural fix is being implemented, and must be labelled:

```
contained pending structural fix — containment: [description] — root cause still open
```

## The fix report

When the root-cause chain terminates, produce a structured report:

### 1. What's happening (plain English)
Explain the symptom in simple terms. What does the user experience? What's broken?

### 2. Why it's happening (the full chain)
Walk through the causal chain from root cause to symptom. Each step should explain HOW the previous step causes the next. Use plain language — imagine explaining to someone who isn't a systems developer.

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
- **Does this fix address the mechanism, or just avoid the trigger?** If the fix removes a feature or disables a code path, it's containment, not a fix.

### 5. Exhaustive elimination status
List every alternative explanation that was considered and what evidence ruled it out (or why it's still open):

```
Ruled out:
- [alternative 1] — [evidence that disproves it]
- [alternative 2] — [evidence that disproves it]

Still open:
- [alternative 3] — [what evidence would resolve it]
```

### 6. Documentation consultation status
**Every relevant documentation source must be consulted before the report is delivered.**

```
Consulted:
- [source] — [what it confirmed / what was found]
```

### 7. Prevent recurrence

Before closing the branch, verify the fix is structural — it will survive updates, reboots, and reconnections. Check:

**Will the fix survive a package/driver/OS update?**
- If you patched a binary: it will be overwritten. Create a pacman hook, apt trigger, launchd daemon, or scheduled task to re-apply after updates.
- If you edited a config: verify the package manager preserves user modifications (pacman creates `.pacnew`, apt creates `.dpkg-dist`, etc.)
- If you changed a kernel module parameter: verify `initramfs` or `dracut` is rebuilt, and the module config survives kernel updates.

**Will the fix survive a reboot?**
- Verify the change is persisted to disk (not just in-memory).
- Verify the service that applies the change starts at boot.
- Verify the change is independent of volatile state (e.g., a device address that changes after deep sleep).

**Will the fix survive a reconnection?**
- For peripherals: verify the device's preferred connection parameters (PPCP) are compatible with your config after reconnection.
- For network devices: verify DHCP or 802.1X re-authentication preserves your config.
- For USB devices: verify the device re-enumerates with the same settings after suspend/resume.

**Could the same class of problem recur from a different trigger?**
- If the root cause was "I/O stall killed a userspace daemon", disabling one I/O source leaves OTHER I/O sources able to cause the same stall. Check for: scheduled balance, scrub, trim, backup, indexing, or update jobs that could cause heavy I/O.
- If the root cause was "aggressive timeout on a peripheral", fixing one timeout leaves other peripherals potentially affected. Check all connected devices.
- If the root cause was "a feature was misconfigured", fix the misconfiguration so the feature works correctly. Removing the feature is containment.

**Verify the fix is actually active — confirm running state matches config:**

| OS | What to verify |
|----|---------------|
| Linux (systemd) | Check `sysfs`/`debugfs` for the actual running value, not just the config file. Check `bluetoothctl info` for per-device params, not just `main.conf`. Check `journalctl -b` for the error message returning after a service restart. Check `pacman -Q` for version changes that could overwrite patches. |
| Linux (non-systemd) | Same verification approach using `/var/log/messages`, `dmesg`, service status. |
| macOS | `log show` for the error message returning. `launchctl print` for service config. `system_profiler` for device state. |
| Windows | `Get-WinEvent` for the error returning. `Get-Service` for service state. `Get-PnpDevice` for device state. |

### 8. Ask to continue
Present the report and ask: "Should I apply this fix?"

## Known fixes registry

These are fixes for common upstream bugs that affect multiple users. They're actual code fixes that haven't been released in a package yet, or package-level patches that need re-application after updates.

**Before suggesting any fix from this registry, verify it actually applies to the user's situation.** A fix that addresses an error message but leaves the running behavior unchanged is cosmetic, not structural.

### BlueZ 5.86: "Failed to set default system config for hci0"

**Bug**: BlueZ 5.86 logs "Failed to set default system config for hci0" on startup when using a stock (empty) configuration.

**Root cause**: Commit `5e5b46c5c0cc` ("adapter: Do not send empty default system parameter list") changed the behavior so that when no config changes are needed, BlueZ skips the MGMT command. But when the config list is empty (stock config), the code jumps to `done:` which leaks the list and logs an error, instead of returning cleanly.

**Scope**: This bug ONLY affects systems with a stock (empty) `main.conf`. If you have custom values in `main.conf` (e.g., `LE.ConnectionSupervisionTimeout=2000`), the MGMT command IS sent and the adapter defaults ARE applied. The error message is absent with custom values. **Only suggest this patch for users with stock main.conf — it leaves the running behavior unchanged for users with custom values.**

**Fix**: Commit `46937fd` ("adapter: Fix 'Failed to set default system config' startup warning") — frees the list and returns instead of jumping to `done:`.

**Patch** (apply to `src/adapter.c` in the bluez source):
```diff
@@ -4963,8 +4963,11 @@ static void load_defaults(struct btd_adapter *adapter)
 	if (!load_le_defaults(adapter, list, &btd_opts.defaults.le))
 		goto done;
 
-	if (mgmt_tlv_list_size(list) == 0)
-		goto done;
+	/* No changes from defaults */
+	if (mgmt_tlv_list_size(list) == 0) {
+		mgmt_tlv_list_free(list);
+		return;
+	}
 
 	err = mgmt_send_tlv(adapter->mgmt, MGMT_OP_SET_DEF_SYSTEM_CONFIG,
 			adapter->dev_id, list, NULL, NULL, NULL);
```

**How to apply on Arch/CachyOS**:
1. Get the package source: `pacman -S --needed asp && asp checkout bluez`
2. Add the patch to the PKGBUILD's `source` array
3. Add the patch to the `prepare()` function
4. Build: `makepkg -sf`
5. Install: `sudo pacman -U bluez-5.86-*.pkg.tar.zst`
6. Restart: `sudo systemctl restart bluetooth`

**Persistence**: Create a pacman hook to re-apply after updates:
```
/etc/pacman.d/hooks/bluez-patch.hook
```
```ini
[Trigger]
Operation = Install
Operation = Upgrade
Type = Package
Target = bluez

[Action]
Description = Re-applying BlueZ default system config patch
When = PostTransaction
Exec = /usr/local/bin/bluez-repatch.sh
```

**Verification**: After applying, check:
- `journalctl -u bluetooth | grep "Failed to set default system config"` — should be absent
- `cat /sys/kernel/debug/bluetooth/hci0/supervision_timeout` — should match main.conf value
- **Critical**: Verify the actual running connection params, not just the adapter defaults. Check per-device params in `/var/lib/bluetooth/<adapter>/<device>/info` — these may differ from the adapter defaults if the peripheral's PPCP overrides them.

**Affected versions**: BlueZ 5.86 (confirmed). Fixed in upstream git (commit 46937fd). Awaiting release package as of 2026-05-13.

## OS-specific profiles

These profiles define the diagnostic commands, log locations, and **problem-category queries** for each OS. Load the appropriate one after system identification.

Each profile includes:
- **Log & service commands** — where to look
- **Problem-category queries** — what to run when a specific type of problem is reported
- **Common stall indicators** — what patterns in the logs mean the system was stalled
- **Device inspection** — how to check hardware and peripherals

### Linux (systemd)

#### Logs & services

**System logs:**
- `journalctl -b` — current boot
- `journalctl -b -N` — Nth previous boot
- `journalctl -k` — kernel messages (same as `dmesg`)
- `journalctl -u <service>` — service-specific logs
- `journalctl --since "2026-05-13 10:00" --until "2026-05-13 11:00"` — time range

**Package logs:**
- `/var/log/pacman.log` (Arch/CachyOS)
- `/var/log/dpkg.log` (Debian/Ubuntu)
- `/var/log/yum.log` or `/var/log/dnf.log` (Fedora)

**Service management:**
- `systemctl status <service>` — check service state
- `systemctl list-timers` — scheduled jobs
- `systemctl list-units --failed` — failed units
- `systemctl list-units --type=service --state=running` — running services

#### Problem-category queries

**Freeze/stall:**
```
journalctl -b --no-pager | grep -iE "watchdog|timeout|stall|hung_task|blocked for"
journalctl -b --no-pager -k | grep -iE "btrfs|ext4|I/O error|NMI|hardlock"
systemctl list-units --failed
```

**Bluetooth disconnect:**
```
journalctl -b --no-pager | grep -iE "bluetooth|bluetoothd|hci0|supervision|timeout|disconnect"
bluetoothctl info <addr>
bluetoothctl show
busctl tree org.bluez
cat /etc/bluetooth/main.conf
# Verify ACTUAL running params, not just config file:
cat /sys/kernel/debug/bluetooth/hci0/supervision_timeout
cat /sys/kernel/debug/bluetooth/hci0/conn_latency
cat /sys/kernel/debug/bluetooth/hci0/conn_max_interval
# Check per-device stored params (may differ from adapter defaults):
sudo cat /var/lib/bluetooth/<adapter>/<device>/info | grep -A4 ConnectionParameters
```

**Audio issues:**
```
wpctl status          # PipeWire
pw-cli info           # PipeWire details
pactl list            # PulseAudio compat
journalctl -b --no-pager | grep -iE "pipewire|wireplumber|pulseaudio|alsa"
```

**Boot issues:**
```
journalctl -b --no-pager | head -50        # early boot sequence
systemd-analyze blame                       # boot time per service
systemd-analyze critical-chain              # critical path
systemctl list-units --state=failed
```

**GPU/display issues:**
```
journalctl -b --no-pager -k | grep -iE "nvidia|amdgpu|drm|gpu|reset|EE"
nvidia-smi              # NVIDIA
glxinfo | head -20      # OpenGL info
```

**Filesystem issues:**
```
btrfs device stats /                    # btrfs errors
btrfs scrub status /                    # btrfs scrub
btrfs qgroup show /                     # btrfs quotas (if enabled)
btrfs quota status /                    # quota enabled/disabled
smartctl -a /dev/sdX                     # SMART data
findmnt -t btrfs                         # btrfs mounts
```

#### Common stall indicators

- `systemd-journald.service: Watchdog timeout` — system completely stalled
- Journal file "corrupted or uncleanly shut down" — unclean shutdown from stall
- `hung_task_timeout` — kernel task blocked for 120+ seconds
- `INFO: task <name> blocked for more than N seconds` — I/O stall
- `btrfs: qgroup scan completed` — qgroup rescan was running (check timing)
- `NMI: IOCK` — hardware I/O error
- `D-Bus timeout` / `Did not receive a reply` — IPC service stalled

#### Device inspection

**Bluetooth:**
```
bluetoothctl info <addr>     # device details, paired, trusted, connected
bluetoothctl show            # adapter details, power, discoverable
lsusb | grep -i bluetooth    # USB BT adapter
rfkill list                  # RF kill status
cat /sys/class/bluetooth/hci0/address  # adapter address
# Verify running params vs config file vs per-device stored params:
cat /sys/kernel/debug/bluetooth/hci0/supervision_timeout
cat /sys/kernel/debug/bluetooth/hci0/conn_latency
sudo cat /var/lib/bluetooth/<adapter>/<device>/info | grep -A4 ConnectionParameters
```

**USB:**
```
lsusb                        # list USB devices
lsusb -t                     # USB tree (show topology)
usb-devices                  # detailed USB device info
cat /sys/kernel/debug/usb/devices  # kernel USB debug
```

**Storage:**
```
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT   # block devices
findmnt                                 # mounts
smartctl -a /dev/sdX                    # SMART
hdparm -I /dev/sdX                      # drive info
```

**Network:**
```
ip link show                  # interfaces
ip addr show                  # addresses
nmcli device status           # NetworkManager
iw dev                        # wireless
```

---

### Linux (non-systemd)

#### Logs & services

**System logs:**
- `/var/log/messages` or `/var/log/syslog` — main system log
- `/var/log/daemon.log` — daemon log (some distros)
- `dmesg` — kernel ring buffer
- `/var/log/rc.log` — OpenRC log (if enabled)

**Service management:**
- OpenRC: `rc-status`, `rc-update show`, `/etc/init.d/<service> status`
- runit: `sv status <service>`, `/etc/sv/<service>/run`
- s6: `s6-rc list`, `s6-rc status <service>`

#### Problem-category queries

**Freeze/stall:**
```
dmesg | grep -iE "hung_task|blocked for|I/O error|NMI|hardlock"
grep -iE "watchdog|timeout|stall" /var/log/messages
cat /proc/sys/kernel/hung_task_timeout_secs
```

**Bluetooth disconnect:**
```
grep -iE "bluetooth|bluetoothd|hci0|supervision|timeout" /var/log/messages
bluetoothctl info <addr>
bluetoothctl show
cat /etc/bluetooth/main.conf
# Verify running params:
cat /sys/kernel/debug/bluetooth/hci0/supervision_timeout
```

**Audio issues:**
```
wpctl status
pactl list
grep -iE "pipewire|wireplumber|pulseaudio|alsa" /var/log/messages
```

**Boot issues:**
```
dmesg | head -50
rc-status                          # OpenRC services
cat /var/log/rc.log                # OpenRC boot log
```

**Filesystem issues:**
```
btrfs device stats /
btrfs scrub status /
btrfs quota status /
smartctl -a /dev/sdX
```

#### Common stall indicators

- `hung_task_timeout` — kernel task blocked for 120+ seconds
- `INFO: task <name> blocked for more than N seconds` — I/O stall
- `btrfs: qgroup scan completed` — qgroup rescan was running
- Log gaps in `/var/log/messages` — system too stalled to write
- `NMI: IOCK` — hardware I/O error

#### Device inspection

Same as systemd Linux — `bluetoothctl`, `lsusb`, `lsblk`, `smartctl`, `ip` are distro-agnostic.

---

### macOS

#### Logs & services

**System logs:**
- `log show` — unified log system
- `log show --predicate 'process == "bluetoothd"'` — filter by process
- `log show --predicate 'subsystem == "com.apple.bluetooth"'` — filter by subsystem
- `log show --start "2026-05-13 10:00:00" --end "2026-05-13 11:00:00"` — time range
- `log show --style compact` — compact format
- Console.app — GUI log viewer

**Service management:**
- `launchctl list` — loaded launch daemons/agents
- `launchctl print gui/$(id -u)` — user domain services
- `launchctl print system/` — system domain services
- `/Library/LaunchDaemons/`, `/Library/LaunchAgents/` — system-level
- `~/Library/LaunchAgents/` — user-level

#### Problem-category queries

**Freeze/stall:**
```
log show --predicate 'eventMessage contains "timeout" or eventMessage contains "stall" or eventMessage contains "hang"' --style compact
log show --predicate 'process == "WindowServer"' --last 1h --style compact
log show --predicate 'subsystem == "com.apple.kernel"' --last 1h --style compact
sysctl vm.loadavg                    # system load
vm_stat                              # VM statistics
```

**Bluetooth disconnect:**
```
log show --predicate 'process == "bluetoothd"' --last 1h --style compact
log show --predicate 'subsystem == "com.apple.bluetooth"' --last 1h --style compact
system_profiler SPBluetoothDataType
defaults read /Library/Preferences/com.apple.Bluetooth
```

**Audio issues:**
```
log show --predicate 'process == "coreaudiod"' --last 1h --style compact
system_profiler SPAudioDataType
sudo killall coreaudiod              # restart audio daemon
```

**Boot issues:**
```
log show --start $(date -v-1H "+%Y-%m-%d %H:%M:%S") --style compact | head -50
system_profiler SPSoftwareDataType   # OS version, boot time
uptime                               # boot time
```

**Filesystem issues:**
```
diskutil verifyVolume /
diskutil apfs list
log show --predicate 'subsystem == "com.apple.apfs"' --last 1h --style compact
smartctl -a /dev/disk0               # needs smartmontools
```

**GPU/display issues:**
```
log show --predicate 'process == "WindowServer"' --last 1h --style compact
system_profiler SPDisplaysDataType
```

#### Common stall indicators

- `log show` gaps — periods with no entries = system stalled
- WindowServer hangs — `WindowServer` process not responding
- `bluetoothd` timeout entries — Bluetooth daemon stalled
- `kernel` panic entries — kernel panic
- `disk I/O error` — storage issues
- `spinlock` or `deadlock` entries — kernel lock contention

#### Device inspection

**Bluetooth:**
```
system_profiler SPBluetoothDataType
defaults read /Library/Preferences/com.apple.Bluetooth
log show --predicate 'process == "bluetoothd"' --last 5m --style compact
```

**USB/Thunderbolt:**
```
system_profiler SPUSBDataType
system_profiler SPThunderboltDataType
ioreg -p IOUSB -l -w 0              # USB device tree
```

**Storage:**
```
diskutil list
diskutil apfs list
system_profiler SPNVMeDataType
system_profiler SPSerialATADataType
smartctl -a /dev/disk0               # needs smartmontools
```

**Network:**
```
networksetup -listallhardwareports
ifconfig
networksetup -getinfo Wi-Fi
system_profiler SPNetworkDataType
```

---

### Windows

#### Logs & services

**System logs:**
- Event Viewer: `eventvwr.msc` or `Get-WinEvent` / `wevtutil`
- System log: `Get-WinEvent -LogName System -MaxEvents 100`
- Application log: `Get-WinEvent -LogName Application -MaxEvents 100`
- Reliability Monitor: `perfmon /rel` — crash/failure history
- `wevtutil qe System /c:100 /f:text` — command-line event query

**Service management:**
- `Get-Service` — list services
- `Get-Service | Where-Object {$_.Status -eq "Running"}` — running services
- `sc query <service>` — service status
- Task Scheduler: `schtasks /query`, `taskschd.msc`

#### Problem-category queries

**Freeze/stall:**
```powershell
Get-WinEvent -LogName System -MaxEvents 200 | Where-Object {$_.LevelDisplayName -eq "Error" -or $_.LevelDisplayName -eq "Warning"}
Get-WinEvent -LogName System | Where-Object {$_.Message -match "timeout|hang|stall|frozen|unresponsive"}
Get-WinEvent -LogName System | Where-Object {$_.ProviderName -match "Microsoft-Windows-Kernel"}
# Check for WHEA errors (hardware)
Get-WinEvent -LogName System | Where-Object {$_.ProviderName -match "WHEA"}
# Check for DCOM timeouts (IPC stalls)
Get-WinEvent -LogName System | Where-Object {$_.Id -eq 10010 -or $_.Id -eq 10016}
```

**Bluetooth disconnect:**
```powershell
Get-WinEvent -LogName System | Where-Object {$_.ProviderName -match "Bluetooth"}
Get-PnpDevice -Class Bluetooth
Get-Service bthserv | Format-List *
Get-NetAdapter | Where-Object {$_.InterfaceType -eq "Bluetooth"}
# Device Manager details
Get-PnpDevice -Class Bluetooth | Get-PnpDeviceProperty -KeyName DEVPKEY_Device_DriverVersion
```

**Audio issues:**
```powershell
Get-Service Audiosrv | Format-List *
Get-WinEvent -LogName System | Where-Object {$_.ProviderName -match "Audio"}
Get-PnpDevice -Class AudioEndpoint
Get-PnpDevice -Class MEDIA
```

**Boot issues:**
```powershell
Get-WinEvent -LogName System | Where-Object {$_.ProviderName -match "Boot"} | Select-Object -First 20
# Boot performance
Get-WinEvent -LogName System | Where-Object {$_.ProviderName -match "Diagnostics-Performance"}
# Last boot time
(Get-CimInstance Win32_OperatingSystem).LastBootUpTime
```

**Filesystem issues:**
```powershell
# Check disk
chkdsk C: /f
# NTFS info
fsutil fsinfo ntfsinfo C:
# SMART
wmic diskdrive get status,model,size
# Volume errors
Get-WinEvent -LogName System | Where-Object {$_.ProviderName -match "disk|Ntfs"}
```

**GPU/display issues:**
```powershell
Get-WinEvent -LogName System | Where-Object {$_.ProviderName -match "Display"}
# TDR events (Timeout Detection and Recovery)
Get-WinEvent -LogName System | Where-Object {$_.Id -eq 4101}
# GPU driver info
Get-WmiObject Win32_VideoController | Format-List *
```

#### Common stall indicators

- Event ID 10010 / 10016 — DCOM timeout (IPC service stalled)
- Event ID 4101 — GPU TDR (display driver timeout)
- WHEA errors — hardware errors (memory, PCIe, disk)
- `EventLog` service gaps — system too stalled to write events
- `Disk` errors — I/O failures
- `Application Error` / `Application Crash` — app failures
- `Windows Error Reporting` — crash reports

#### Device inspection

**Bluetooth:**
```powershell
Get-PnpDevice -Class Bluetooth | Format-Table Status, Class, FriendlyName, InstanceId
Get-Service bthserv | Format-List *
Get-NetAdapter | Where-Object {$_.InterfaceType -eq "Bluetooth"}
```

**USB:**
```powershell
Get-PnpDevice -Class USB | Format-Table Status, Class, FriendlyName
usbview.exe                          # USB tree (from Windows SDK)
```

**Storage:**
```powershell
Get-Disk | Format-Table Number, FriendlyName, Size, HealthStatus
Get-Volume | Format-Table DriveLetter, FileSystemLabel, FileSystem, HealthStatus
wmic diskdrive get model,status,size
smartctl -a /dev/sdX                 # needs smartmontools
```

**Network:**
```powershell
Get-NetAdapter | Format-Table Name, InterfaceDescription, Status, LinkSpeed
Get-NetIPAddress | Format-Table InterfaceAlias, IPAddress
ipconfig /all
```

## Safe diagnostics

Read-only diagnostics can run without user approval:

- read logs (journalctl, log show, Get-WinEvent, dmesg)
- inspect service state (systemctl, launchctl, Get-Service, sv status)
- inspect config files
- inspect package/driver versions
- inspect process trees, scheduled tasks, device state
- inspect filesystem state (device stats, SMART, scrub status)
- check what changed recently (package logs, journal time ranges, update history)

State-changing actions require the fix report and user approval.

## Documentation-first fixes

Before persistent fixes, consult the relevant documented layer:

- OS/distribution documentation
- Package/driver documentation
- Installed help/man pages
- Desktop environment docs
- Service daemon docs
- Upstream release notes/issues

Use local hacks only when documented methods fail, with the reason explained and rollback in place.

## Upstream bug freshness

When using GitHub/GitLab/upstream issues as evidence, check freshness:

- opened within 30 days
- updated/commented within 30 days
- still open and applies to current package/driver versions
- explicitly unresolved
- linked from current release notes or known-issues pages

## Containment is temporary

Close a branch only when the structural fix is in place. A containment measure alone is an open branch:

```
contained pending structural fix — containment: [description] — root cause still open
```

## Branch closure conditions

A branch can close only when:

- root cause confirmed and structurally fixed
- root cause ruled out
- branch proven unrelated
- branch accepted as permanent containment by the user
- branch blocked with exact resume condition

## Official directions / verified-only rule

1. Prefer verified information over improvisation.
2. Source hierarchy: official docs → upstream docs → installed help → local verification → clearly-labelled inference.
3. For UI navigation: check docs first, inspect installed version if possible, say UNKNOWN if unverified.
4. For scripts: use documented mechanisms, preserve backups, log what was touched.
5. If trusted documentation is unavailable: state UNKNOWN, explain what is known, provide safest verification path.

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
