---
name: pc-troubleshooter
description: "Root-cause diagnostic methodology for any PC — system-agnostic chain reasoning, smoking gun evidence, and OS-specific diagnostic commands. Use for freezes, disconnects, crashes, performance issues, boot problems, service failures, hardware quirks, or any 'it worked before and now it doesn't' situation."
---

# PC Troubleshooter

A root-cause diagnostic skill for **any** PC system. Use when the user reports any PC problem: freezes, disconnects, crashes, performance issues, boot problems, service failures, hardware quirks, or any "it worked before and now it doesn't" situation.

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
- **Look tangentially.** The cause may not be in the obvious component. A Bluetooth mouse disconnecting might be caused by a filesystem rescan stalling the I/O subsystem — the mouse is just the canary.
- **Distinguish "has always been this way" from "this is new behaviour".** If the system has always treated an error as fatal, then the error isn't new — what's new is that the device is now hitting that code path. Why?
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
- This explains "my USB keyboard works but my Bluetooth mouse doesn't" — the keyboard doesn't need the stalled daemon.

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

### 5. Documentation consultation status
**Every relevant documentation source must be consulted before the report is delivered.**

```
Consulted:
- [source] — [what it confirmed / what was found]
```

### 6. Ask to continue
Present the report and ask: "Should I apply this fix?"

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

Use local hacks only when documented methods fail, the reason is explained, and rollback exists.

## Upstream bug freshness

When using GitHub/GitLab/upstream issues as evidence, check freshness:

- opened within 30 days
- updated/commented within 30 days
- still open and applies to current package/driver versions
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
