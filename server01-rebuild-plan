# server01 Rebuild — Master Plan

**Purpose:** Single source of truth for the server01 rebuild. Upload/paste this into any Claude session (any model tier) to restore full context. Update the Status column and Session Log as work progresses.

**Last updated:** 2026-07-06 (initial draft)

---

## End State

server01 = 24/7 home server. Ubuntu Server 26.04, headless, SSH via Termius. All services in Docker containers via docker compose. Monitored by RPi5 (Prometheus/Grafana/NUT — separate box, not in scope here except exporters/clients on server01).

Services (each its own compose stack):
- **Plex** (NVENC transcode via RTX 2070)
- **Frigate** (TensorRT detection + NVENC on RTX 2070; Reolink E1 Zoom at 192.168.8.106; MQTT → Mosquitto on HA Green 192.168.8.2:1883)
- **Immich** (likely fresh start — see Open Decisions)
- **Mealie**
- Future stacks as needed

## Hard Rules

1. **Everything runs in Docker.** No native service installs. No exceptions without a documented reason in this file.
2. **The Plex source Red (exFAT) is READ-ONLY until checksum verification passes.** Mount `ro` or leave unmounted. Never format, never write.
3. **Immich does not start until photo storage location is decided and mounted** (see Open Decisions) and the old-data question is resolved.
4. **All fstab entries use `/dev/disk/by-id/`** with `nofail` on data drives. Never `/dev/sdX` — letters shuffle between boots.
5. **Architectural decisions before execution.** If a step forces an undecided choice, stop and decide first.
6. **If a step goes wrong, cleanup/undo instructions come first** before any new approach.

## Hardware

- **CPU/board:** Ryzen 7 2700X on MSI X470 (stable BIOS flashed)
- **GPU:** RTX 2070 — Frigate TensorRT + Plex/Frigate NVENC
- **Network:** IP 192.168.8.3 reserved on Brume 3 by MAC — confirmed intact
- **UPS:** CyberPower CP1500PFCLCD. USB data cable will run to **RPi5** (NUT server); server01 = NUT netclient. Cable not yet connected — low priority.

### Drives

| Drive | Device | Role | Status |
|---|---|---|---|
| Samsung 970 EVO 500GB NVMe | nvme (OS) | Ubuntu Server 26.04 OS. Spare ~400GB use = **undecided** | OS installed (fresh reinstall) |
| Inland 1TB NVMe | nvme (2nd) | `/var/lib/docker` — all images, containers, volumes | Not yet set up |
| WD Red 4TB #1 | source | Plex media **source** (exFAT, copied from old build) | READ-ONLY until verified |
| WD Red 4TB #2 | dest | Plex media destination (ext4) → `/mnt/media`; had 3,343 UDMA CRC errors last build (usually cabling) | Verify checksums + recheck SMART |
| WD Black 2TB | — | Frigate recordings → `/mnt/frigate` | Needs format (ext4) |
| WD Blue 4TB (SMR) | — | TBD — caused ata4 SATA errors last build; candidate for retirement | Health-check, then decide |

**Post-verification note:** once checksums pass, source Red is freed up. Its next role (OneDrive/cloud backup target `/mnt/backup`) requires reformatting exFAT → ext4 — only after verification passes and only with explicit confirmation.

## Open Decisions

1. **Samsung 500GB spare space (~400GB).** Candidates: Immich originals (`/mnt/photos`), Immich Postgres + ML cache, nothing (keep OS drive clean). Decide before Phase 6. Tradeoff notes: photos on OS drive means OS reinstalls require care; but it's the only fast storage not otherwise allocated, and the old library fit in 500GB SATA.
2. **Old Immich data — CONFIRMED PRESERVED.** Crucial 500GB SATA SSD in ITX Windows PC was not reformatted; ext4 contents intact. Case stays closed — no exceptions. Copy method: **WSL2 `wsl --mount`** (first-party ext4 support in Win11): elevated PowerShell → `wsl --mount \\.\PHYSICALDRIVEn --partition 1 --type ext4` (find n via `Get-Disk`), then rsync/scp from inside WSL2 directly to server01 over network. `wsl --unmount` when done. Deferred until Phase 6 — PC not readily accessible. pg_dump: while mounted, check the SSD's old `/mnt/backup/migration` path (≤1 min); restore if found, else fresh DB with re-imported originals.
3. **WD Blue fate.** SMART long test when convenient. SMR + prior SATA errors = likely retire. No role assigned until it proves itself.
4. **Plex claim/identity.** Fresh Plex server identity vs. attempting to preserve old server ID — decide at Phase 4 (fresh is simpler; watch history is lost either way unless DB was backed up).

## Phases

Work top to bottom. Don't skip ahead past an unresolved blocker.

### Phase 0 — Baseline & safety
- [ ] Confirm SSH access, IP = 192.168.8.3, hostname set
- [ ] `apt update && apt full-upgrade`, reboot
- [ ] Inventory drives: `ls -l /dev/disk/by-id/`, `lsblk -o NAME,SIZE,FSTYPE,LABEL,MODEL,SERIAL` — record which by-id = which role in this doc
- [ ] SMART baseline on all drives (`smartctl -a`), note UDMA CRC counts (esp. dest Red)
- [ ] Confirm source Red is not mounted rw anywhere

### Phase 1 — Storage layout
- [ ] Create mount points: `/mnt/media`, `/mnt/frigate`, `/mnt/backup` (later), `/mnt/photos` (if decided)
- [ ] Format WD Black ext4
- [ ] fstab entries by-id with `nofail`; `systemctl daemon-reload`; `mount -a`; reboot test
- [ ] Inland 1TB: format ext4, mount, relocate `/var/lib/docker` to it (configure before or immediately after Docker install — daemon.json `data-root` or mount at `/var/lib/docker`)

### Phase 2 — Plex data verification
- [ ] Mount source Red read-only
- [ ] Checksum compare source vs dest Red (run in tmux; xxh128 or md5 manifest both sides)
- [ ] Recheck dest Red UDMA CRC count — if it grew since baseline, replace/reseat SATA cable before trusting the drive
- [ ] On pass: source Red released → future `/mnt/backup` (reformat only with explicit go-ahead)
- [ ] On fail: re-copy failed files from source, re-verify

### Phase 3 — Docker foundation
- [ ] Install NVIDIA driver (server/headless) + verify `nvidia-smi`
- [ ] Install Docker Engine (official repo) + compose plugin
- [ ] Confirm `data-root` on Inland 1TB
- [ ] Install NVIDIA Container Toolkit; test GPU in container (`docker run --rm --gpus all nvidia/cuda:*-base nvidia-smi`)
- [ ] Compose layout: one directory per stack (e.g. `/opt/stacks/plex`, `/opt/stacks/frigate`, …), each with its own `compose.yaml`

### Phase 4 — Plex (Docker)
- [ ] Compose stack with NVENC device access, `/mnt/media` bind mount (read-only into container), config volume on Inland NVMe
- [ ] Claim server, verify library scan, verify HW transcode
- [ ] Confirm remote/LAN access works (the old browser-timeout issue — diagnose UFW/port binding fresh if it recurs)

### Phase 5 — Frigate (Docker)
- [ ] Compose stack: TensorRT detector, NVENC, `/mnt/frigate` for recordings, shm sizing
- [ ] Reolink E1 Zoom config (HEVC 4K/20fps main, H.264 640×360/10fps sub, RTSP direct)
- [ ] MQTT to HA Green with dedicated `frigate` user
- [ ] Re-point HA Frigate integration to http://192.168.8.3:8971
- [ ] (Later) Amcrest floodlight cam — driveway; H264 main, reduced substream

### Phase 6 — Immich
- [ ] Resolve Open Decision 1 (photo storage location on server01)
- [ ] ITX: install WSL2 if not present; `wsl --mount` the Crucial SSD (see Decision 2); rsync originals → server01; check for pg_dump while mounted; `wsl --unmount`
- [ ] Compose stack; originals at decided location; DB on fast storage
- [ ] Restore pg_dump if found, else fresh library + re-import originals

### Phase 7 — Mealie + rclone
- [ ] Mealie compose stack
- [ ] rclone OneDrive sync to `/mnt/backup`: systemd `Type=oneshot`, no `Restart=`, `RequiresMountsFor=/mnt/backup`, exclude `/Personal Vault/**`, timer-driven

### Phase 8 — Monitoring & UPS
- [ ] node_exporter (:9100) + smartctl_exporter (:9633) for RPi5 Prometheus scrape
- [ ] NUT: RPi5 = server (USB cable), server01 = netclient with early shutdown; HA Green/Brume ride out battery
- [ ] MikroTik CSS610 config backup (outstanding loose end)

### Phase 9 — Cleanup & hardening
- [ ] WD Blue decision executed (retire or archive role)
- [ ] UFW rules reviewed and documented here
- [ ] Unattended-upgrades configured
- [ ] Document final fstab + compose tree in this file

## Session Log

Append one line per working session: date — what changed — next step.

- 2026-07-06 — Plan created. Ubuntu reinstalled fresh. Next: Phase 0 baseline.
