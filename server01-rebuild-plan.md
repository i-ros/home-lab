# server01 Rebuild — Master Plan

**Purpose:** Single source of truth for the server01 rebuild. Upload/paste this into any Claude session (any model tier) to restore full context. Update the Status column and Session Log as work progresses.

**Last updated:** 2026-07-08

---

## End State

server01 = 24/7 home server. Ubuntu Server 26.04, headless, SSH via Termius. All services in Docker containers via docker compose. Monitored by RPi5 (Prometheus/Grafana/NUT — separate box, not in scope here except exporters/clients on server01).

Services (each its own compose stack):
- **Plex** (NVENC transcode via RTX 2070)
- **Frigate** (TensorRT detection + NVENC on RTX 2070; Reolink E1 Zoom at 192.168.8.106; MQTT → Mosquitto on HA Green 192.168.8.2:1883)
- **Immich** (fresh start + import old originals — see Open Decisions)
- **Mealie** — LIVE
- Future stacks as needed

## Hard Rules

1. **Everything runs in Docker.** No native service installs. No exceptions without a documented reason in this file.
2. **The Plex source Red (exFAT) is READ-ONLY until checksum verification passes.** Mount `ro` or leave unmounted. Never format, never write.
3. **Immich does not start until photo storage location is decided and mounted** (see Open Decisions) and the old-data question is resolved.
4. **All fstab entries use `/dev/disk/by-id/`** with `nofail` on data drives. Never `/dev/sdX` — letters shuffle between boots. (PROVEN 2026-07-08: NVMe letters swapped across a reboot; by-id held mounts to the correct physical drives.)
5. **Architectural decisions before execution.** If a step forces an undecided choice, stop and decide first.
6. **If a step goes wrong, cleanup/undo instructions come first** before any new approach.
7. **Compose convention:** one dir per stack at `/opt/stacks/<service>/`, config via BIND MOUNTS (not named volumes) so a plain rsync of `/opt/stacks` backs up every service. PUID/PGID 1000, TZ America/Denver, image versions pinned (no `latest` on databases).
8. **Working practice:** one command at a time, review output before proceeding, assume only the stated command ran. When a pattern emerges that could be standardized or made safer, proactively suggest an optimization rather than just executing — the by-id fstab rule and the /opt/stacks convention both came from this and prevented real mishaps.
9. **GitHub / secrets discipline.** This repo is private and holds the plan. Two rules, and they're different:
   - **Never commit credentials** — passwords, tokens, API keys. In committed docs, reference them generically (e.g. `POSTGRES_PASSWORD=<in password manager>`) without naming the manager or the entry title — where credentials live is itself sensitive. If compose files go in the repo, real values live in a `.gitignore`'d `.env`. Git history is permanent — a committed secret is compromised even after deletion, so the fix for a leaked secret is always to ROTATE it, not just delete the line.
   - **Local network details (private-range IPs like 192.168.x.x, MACs, hostnames) ARE fine to commit** and should be — they're meaningless outside the LAN and make the doc actually usable. Don't over-redact these; scrubbing them costs clarity for zero security gain.
   - Default posture: be specific where it's generic/local, be strict where it's a credential. When unsure whether something is repo-safe, ask — don't assume.

## Hardware

- **CPU/board:** Ryzen 7 2700X on MSI X470 (stable BIOS flashed)
- **GPU:** RTX 2070 — driver 580-server (580.159.03, CUDA 13.0), Container Toolkit 1.19.1. Verified GPU-in-container working.
- **Network:** eno1 (MAC 30:9c:23:d5:68:f8) → 192.168.8.8 (ethernet, primary) reserved on Brume 3. 192.168.8.7 = wifi (disabled, break-glass only). DHCP + MAC-match in netplan.
- **UPS:** CyberPower CP1500PFCLCD. USB data cable will run to **RPi5** (NUT server); server01 = NUT netclient. Cable not yet connected — low priority.

### Drives

| Drive | by-id | Role | Status |
|---|---|---|---|
| Samsung 970 EVO 500GB NVMe | nvme-Samsung_SSD_970_EVO_500GB_S466NX0KA14327N | Ubuntu OS. Spare ~400GB use = **undecided** | OS installed |
| Inland 1TB NVMe | nvme-Inland_QN450_NVMe_SSD_IB26AC1000P03985 | `/var/lib/docker` (ext4 "docker", UUID 4cea74ce-20a4-4969-a3be-fff04e7ac4dd) | DONE, mounted, boot-verified |
| WD Red 4TB (source) | ata-WDC_WD40EFZX-68AWUN0_WD-WX32D124UYHS | Plex media SOURCE (exFAT "plex") | READ-ONLY until verified. NOT in fstab (manual ro mount only) |
| WD Red 4TB (dest) | ata-WDC_WD40EFZX-68AWUN0_WD-WX62D12CCFRX | Plex media → `/mnt/media` (ext4) | Mounted. Holds dvr+music+videos. CRC baseline 3343 (stable) |
| WD Black 2TB | ata-WDC_WD2003FZEX-00SRLA0_WD-WMC6N0P1DV7X | Frigate recordings → `/mnt/frigate` (ext4 "frigate") | DONE, mounted, boot-verified |
| WD Blue 4TB (SMR) | — | TBD — prior ata4 errors; retirement candidate | PHYSICALLY PULLED — diagnose later |

**Post-verification note:** once checksums pass, source Red is freed up. Its next role (OneDrive/cloud backup target `/mnt/backup`) requires reformatting exFAT → ext4 — only after verification passes and only with explicit confirmation.

## Open Decisions

1. **Samsung 500GB spare space (~400GB).** Candidates: Immich originals (`/mnt/photos`), Immich Postgres + ML cache, nothing (keep OS drive clean). Decide before Phase 6. Photos on OS drive means OS reinstalls need care, but it's the only fast storage unallocated and the old library fit in 500GB.
2. **Old Immich data — CONFIRMED PRESERVED.** Crucial 500GB SATA SSD in ITX Windows PC not reformatted; ext4 intact. Case stays closed. Copy method: **WSL2 `wsl --mount`** (elevated PowerShell → `wsl --mount \\.\PHYSICALDRIVEn --partition 1 --type ext4`, find n via `Get-Disk`), then rsync from WSL2 to server01. `wsl --unmount` after. Deferred to Phase 6 — PC not readily accessible. Check SSD's old `/mnt/backup/migration` for pg_dump while mounted; restore if found else fresh DB.
3. **WD Blue fate.** SMART long test when convenient. SMR + prior SATA errors = likely retire. Currently pulled from case.
4. **Plex claim/identity.** Fresh identity vs. preserve old server ID — decide at Phase 4 (fresh is simpler; watch history lost either way unless DB backed up).

## Phases

### Phase 0 — Baseline & safety — COMPLETE
- [x] SSH access, IP 192.168.8.8, hostname server01
- [x] apt full-upgrade + autoremove, reboot
- [x] Drive inventory + by-id mapping (recorded in Drives table)
- [x] SMART baselines — all PASSED, zero reallocated/pending. dest Red CRC = 3343
- [x] WiFi + Bluetooth soft-blocked via rfkill (persists reboots)

### Phase 1 — Storage layout — COMPLETE
- [x] Mount points /mnt/media, /mnt/frigate created
- [x] WD Black formatted ext4 "frigate" → /mnt/frigate
- [x] dest Red → /mnt/media (ext4)
- [x] Inland → /var/lib/docker (ext4), fstab by-id nofail
- [x] Docker mount-guard: docker.service.d/require-mount.conf RequiresMountsFor=/var/lib/docker
- [x] fstab all by-id + nofail; boot-tested (all mounts survived, docker active)
- [ ] /mnt/backup — deferred until source Red freed after Phase 2

### Phase 2 — Plex data verification — IN PROGRESS (open loop)
- [x] Source Red mounted read-only
- [x] music + videos rsynced source→dest (322GB, verified transfer complete)
- [ ] **RUNNING: xxh128 manifest compare (music+videos+dvr), source vs dest**
- [ ] Recheck dest Red CRC after — expect 3343 (bad-cable theory confirmed if stable)
- [ ] `sudo chown -R ian:ian /mnt/media` after verify (dvr is root-owned from old copy)
- [ ] On pass: source Red released → future /mnt/backup (reformat only with explicit go-ahead)

### Phase 3 — Docker foundation — COMPLETE
- [x] NVIDIA driver 580-server, nvidia-smi verified
- [x] Docker Engine 29.6.1 (official repo), compose plugin, data-root on Inland
- [x] Container Toolkit 1.19.1, runtime configured, GPU-in-container smoke test passed
- [x] Compose layout /opt/stacks/<service>/ established
- [x] ian added to docker group

### Phase 4 — Plex (Docker) — STAGED, not launched
- [x] compose.yaml written at /opt/stacks/plex/ (lscr.io/linuxserver/plex, host net, NVENC caps video,compute,utility, /mnt/media ro, tmpfs transcode, config bind mount)
- [ ] Waiting on: Phase 2 verify done + PLEX_CLAIM token from plex.tv/claim (4-min expiry) at launch
- [ ] Claim server, verify library scan, verify HW transcode
- [ ] Confirm LAN access (old browser-timeout issue — diagnose fresh if recurs)

### Phase 5 — Frigate (Docker)
- [ ] Compose stack: TensorRT detector, NVENC, /mnt/frigate recordings, shm sizing
- [ ] Reolink E1 Zoom (HEVC 4K/20fps main, H.264 640×360/10fps sub, RTSP direct)
- [ ] MQTT to HA Green with dedicated `frigate` user
- [ ] Re-point HA Frigate integration to http://192.168.8.8:8971
- [ ] (Later) Amcrest floodlight cam — driveway; H264 main, reduced substream

### Phase 6 — Immich
- [ ] Resolve Decision 1 (photo storage location)
- [ ] ITX: WSL2 `wsl --mount` Crucial SSD, rsync originals → server01, check pg_dump, unmount
- [ ] Compose stack; originals at decided location; DB on fast storage
- [ ] Restore pg_dump if found, else fresh library + re-import

### Phase 7 — rclone (Mealie already done)
- [x] Mealie compose stack LIVE (see Services / Session Log 07-08)
- [ ] rclone OneDrive → /mnt/backup: systemd Type=oneshot, no Restart=, RequiresMountsFor=/mnt/backup, exclude /Personal Vault/**, timer-driven

### Phase 8 — Monitoring & UPS
- [ ] node_exporter (:9100) + smartctl_exporter (:9633) for RPi5 Prometheus scrape (targets = 192.168.8.8)
- [ ] NUT: RPi5 = server (USB cable), server01 = netclient early shutdown
- [ ] MikroTik CSS610 config backup (loose end)

### Phase 9 — Cleanup & hardening
- [ ] **ROTATE Mealie Postgres password** — the current value was committed to git history in an earlier version of this doc. Private repo + low-value LAN password = not urgent, but history is permanent so the clean fix is rotation, not deletion. Generate a new alphanumeric password, update BOTH the `mealie` and `postgres` services in /opt/stacks/mealie/compose.yaml, `docker compose down && up -d` (Postgres keeps the old password baked into pgdata, so this also requires either recreating the DB or `ALTER USER mealie PASSWORD` inside the container — sequence to be worked out at the time). Update the password manager entry. After rotating, the old committed value is dead and harmless.
- [ ] WD Blue decision executed
- [ ] UFW baseline (SSH + service ports) — NOT YET configured; nothing firewalled currently
- [ ] Unattended-upgrades
- [ ] Media library cleanup: (copy N) duplicate DVR files + macOS cruft (.DS_Store, AlbumArt_*.jpg) — cosmetic, dedupe before/after Plex scan
- [ ] Document final fstab + compose tree here

## Live Services

- **Mealie** — http://192.168.8.8:9925 — /opt/stacks/mealie/ — mealie v3.20.1 + postgres:17, bind mounts (data + pgdata), ALLOW_SIGNUP=false. Credentials stored in password manager (DB + web admin, separate entries).

## Session Log

- 2026-07-06 — Plan created. Ubuntu reinstalled fresh.
- 2026-07-07 — Confirmed Crucial SSD ext4 intact (WSL2 copy method set). veracrypt/pers_container (100GiB) copied to microSD + xxh128 VERIFIED (7cdf603e827ae26b7a6dbe4775ba820d), card pulled/labeled. Black wiped → ext4 /mnt/frigate. dest Red → /mnt/media. Inland → /var/lib/docker. All fstab by-id+nofail, boot-tested (NVMe letters swapped, by-id held). Docker 29.6.1 + NVIDIA 580 + Container Toolkit, GPU-in-container verified. Mealie LIVE. Plex stack staged (not launched). music+videos rsynced to /mnt/media.
- 2026-07-08 — **OPEN LOOP: media xxh128 verify still running** (source read slow; src.manifest 0 bytes, load ~1.0 = still grinding). NEXT: confirm manifests match → recheck sdb CRC (expect 3343) → chown /mnt/media to ian → then launch Plex (grab claim token) or build Frigate. Added GitHub/secrets discipline (Hard Rule 9) + Mealie password rotation task (Phase 9) after realizing the Postgres password was committed to git history in an earlier doc version. Built gear-inventory skill (separate deliverable, staged for repo review pending an org strategy).

### Next-session quick refs
- Check verify status: `ls -la /tmp/*.manifest; uptime` (non-zero manifest sizes + load ~0 = done)
- Compare: `diff /tmp/src.manifest /tmp/dst.manifest && echo MATCH || echo MISMATCH`
- CRC recheck: `sudo smartctl -a /dev/sdb | grep CRC` (expect 3343)
- Ownership fix: `sudo chown -R ian:ian /mnt/media`
- Plex launch: paste fresh token into /opt/stacks/plex/compose.yaml PLEX_CLAIM, `cd /opt/stacks/plex && docker compose up -d`
