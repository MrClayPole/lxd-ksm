# LXD KSM Enabler

Enable Kernel Same-page Merging (KSM) for LXD virtual machines, recovering
30-35 GB of RAM on typical multi-VM deployments running identical OS images.

## The Problem

LXD virtual machines use QEMU's `memory-backend-memfd` with `share=on`, which
creates **MAP_SHARED** memory. The Linux kernel's KSM silently rejects
`MADV_MERGEABLE` on shared mappings — so KSM does nothing, even when enabled.

```
VmFlags: rd wr sh mr mw me ms dc sd hg
                ^^
          sh = SHARED → KSM blocked
          No "mg" (mergeable) flag
```

## The Fix

An inotify-based watcher intercepts LXD's `qemu.conf` before QEMU reads it,
replacing `memory-backend-memfd` with `memory-backend-ram` and adding
`merge=on`. This creates **MAP_PRIVATE** memory that KSM can merge.

```
VmFlags: rd wr mr mw me dc ac sd hg mg
                                      ^^
                                mg = mergeable ✓
```

## Architecture

```
┌─────────────────────────────────────────────────────┐
│  lxd-ksm-enable.service (systemd)                   │
│                                                     │
│  Phase 0: Tune KSM sysfs knobs                      │
│    pages_to_scan = 100000  │  sleep_millisecs = 500 │
│    use_zero_pages = 1      │  run = 1              │
│                                                     │
│  Phase 1: Scan existing / modify all qemu.conf      │
│                                                     │
│  Phase 2: inotify → watch for new VMs + keep KSM up │
└─────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────┐
│  LXD writes qemu.conf → inotify fires → sed fixes   │
│  memory-backend-memfd  →  memory-backend-ram         │
│  share = "on"          →  (deleted)                  │
│                         →  merge = "on"              │
│  → QEMU reads modified config → MAP_PRIVATE → KSM    │
└─────────────────────────────────────────────────────┘
```

## Files

| File | Purpose |
|------|---------|
| `lxd-ksm-enable` | Main watcher script (Phases 0-2) |
| `lxd-ksm-enable.service` | systemd unit for the watcher |
| `lxd-ksm-rollout` | One-time VM restart for existing VMs |
| `90-ksm.conf` | sysctl.d snippet (persistence) |
| `ksm.conf` | tmpfiles.d snippet (sysfs at boot) |

## Installation

```bash
# 1. Install the watcher script
sudo cp lxd-ksm-enable /usr/local/bin/
sudo chmod +x /usr/local/bin/lxd-ksm-enable

# 2. Install the systemd service
sudo cp lxd-ksm-enable.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now lxd-ksm-enable

# 3. Make settings persistent
sudo cp 90-ksm.conf /etc/sysctl.d/
sudo cp ksm.conf /etc/tmpfiles.d/
```

## Initial Rollout (Existing VMs)

If you already have running VMs, restart them one at a time:

```bash
sudo ./lxd-ksm-rollout
```

This restarts each VM with a 2-minute boot gap and verifies the config.

## Verification

```bash
# KSM is running?
cat /sys/kernel/mm/ksm/run             # → 1

# RAM saved?
echo $(( $(cat /sys/kernel/mm/ksm/general_profit) / 1024 / 1024 )) MB

# VM memory is mergeable?
for pid in $(pgrep -f qemu-system); do
  sudo cat /proc/$pid/smaps 2>/dev/null | grep "VmFlags:" | grep -c mg
done
```

## Requirements

- LXD ≥ 5.x (tested on 5.21)
- Ubuntu 24.04+ / kernel ≥ 6.8
- `inotify-tools` package (`inotifywait`)
- `qemu-system-x86_64` bundled with LXD snap

## Caveats

- Uses `memory-backend-ram` instead of `memory-backend-memfd` — this loses
  memfd sealing (live migration, vhost-user). Acceptable for test labs.
- The inotify race window between LXD writing `qemu.conf` and QEMU reading it
  is ~50-200ms. The `sed` replacement takes <1ms. Phase 1 catches any misses.
- `ksmtuned` package conflicts with this setup — remove it if installed.

## License

MIT