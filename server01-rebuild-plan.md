# server01 Rebuild — Master Plan

**Purpose:** Single source of truth for the server01 rebuild. Upload/paste this into any Claude session (any model tier) to restore full context. Update the Status column and Session Log as work progresses.

**Companion docs:** Per-service detail lives in its own doc in this repo (compose + config also committed per stack). Current companions: `frigate.md` (camera/NVR solution). Plex/Mealie companion docs to be retrofitted.

**Last updated:** 2026-07-09 (evening)

---

## End State

server01 = 24/7 home server. Ubuntu Server 26.04, headless, SSH via Termius. All services in Docker containers via docker compose (rclone is the one documented host-level exception — see Hard Rule 1). Monitored by RPi5 (Prometheus/Grafana/NUT — separate box, not in scope here except exporters/clients on server01).

Services (each its own compose stack):
- **Plex** — LIVE (NVENC transcode via RTX 2070, HW confirmed)
- **Frigate** — IN PROGRESS (ONNX detector, TensorRT-accelerated, on RTX 2070; Reolink E1 Zoom at 192.168.8.106; MQTT → Mosquitto on HA Green 192.168.8.2:1883). Full detail in `frigate.md`.
- **Immich** (fresh start + import old originals — see Open Decisions)
- **Mealie** — LIVE
- **rclone** — OneDrive backup (host service, not a container — see Hard Rule 1). Remote authed, initial sync running.
- Future stacks as needed

## Hard Rules

1. **Everything runs in Docker.** No native service installs. No exceptions without a documented reason in this file. **Documented exception: rclone.** It runs as a host-level systemd oneshot + timer, not a container, because it needs direct host filesystem access to `/mnt/backup` and a persistent auth config; containerizing a scheduled pull-down buys nothing. Same call as on the retired ubunrian box.
2. ~~The Plex source Red (exFAT) is READ-ONLY until checksum verification passes.~~ **FULLY RETIRED 2026-07-09.** Verification passed 07-08; source Red reformatted ext4 → `/mnt/backup` on 07-09. The drive no longer exists as a "source." Rule is dead.
3. **Immich does not start until photo storage location is decided and mounted** (see Open Decisions) and the old-data question is resolved. **Immich is fully on hold** — not tonight's work, no dependency blocking it except Decision 1.
4. **All fstab entries use stable IDs** (`/dev/disk/by-id/` or `UUID=`) with `nofail` on data drives. Never `/dev/sdX` — letters shuffle between boots. **PROVEN TWICE:** NVMe letters swapped across the 07-07 reboot (by-id held); swapped AGAIN across the 07-09 reboot — `/var/lib/docker` moved from `nvme1n1` to `nvme0n1`, OS root landed on the other — and UUID/by-id held every mount to the correct physical drive both times.
5. **Architectural decisions before execution.** If a step forces an undecided choice, stop and decide first.
6. **If a step goes wrong, cleanup/undo instructions come first** before any new approach.
7. **Compose convention:** one dir per stack at `/opt/stacks/<service>/`, config via BIND MOUNTS (not named volumes) so a plain rsync of `/opt/stacks` backs up every service. PUID/PGID 1000, TZ America/Denver, image versions pinned (no `latest` on databases).
8. **Working practice:** one command at a time, review output before proceeding, assume only the stated command ran. When a pattern emerges that could be standardized or made safer, proactively suggest an optimization rather than just executing — the by-id fstab rule and the /opt/stacks convention both came from this and prevented real mishaps.
9. **GitHub / secrets discipline.** This repo is private and holds the plan. Two rules, and they're different:
   - **Never commit credentials** — passwords, tokens, API keys. In committed docs, reference them generically (e.g. `POSTGRES_PASSWORD=<in password manager>`) without naming the manager or the entry title — where credentials live is itself sensitive. If compose files go in the repo, real values live in a `.gitignore`'d `.env`. Git history is permanent — a committed secret is compromised even after deletion, so the fix for a leaked secret is always to ROTATE it, not just delete the line. (Note: the rclone config at `~/.config/rclone/rclone.conf` holds the live OneDrive refresh token — it is NOT committed and must never be. Not in the repo tree.)
   - **Local network details (private-range IPs like 192.168.x.x, MACs, hostnames) ARE fine to commit** and should be — they're meaningless outside the LAN and make the doc actually usable. Don't over-redact these; scrubbing them costs clarity for zero security gain.
   - Default posture: be specific where it's generic/local, be strict where it's a credential. When unsure whether something is repo-safe, ask — don't assume.
10. **Long-running jobs go in tmux, always** — with stderr captured to a file and output paths on persistent storage (not /tmp) if they must survive a reboot. Verify-workflow gotchas learned 2026-07-08: `./` prefix must be stripped identically on both manifests; both `comm`/`sort` inputs need `LC_ALL=C`; `comm` compares whole lines, so hash+path both must match.

## Hardware

- **CPU/board:** Ryzen 7 2700X on MSI X470 (stable BIOS flashed). Note: subjectively snappier than the retired ITX/265K build — not the silicon (265K is faster on paper) but the plumbing: media now on direct SATA instead of USB-C enclosure, Docker root on Gen4 NVMe, fresh minimal install, less service contention. The design decisions bought this.
- **GPU:** RTX 2070 — driver 580-server (580.159.03, CUDA 13.0), Container Toolkit 1.19.1. GPU-in-container verified; Plex sees the 2070 by name in transcoder dropdown; **HW transcode confirmed live ("(hw)" in dashboard) 07-09.** Shared between Plex (NVENC transcode) and Frigate (NVDEC decode + ONNX inference) — fine for now; watch 8GB VRAM as cameras grow.
- **Network:** eno1 (MAC 30:9c:23:d5:68:f8) → 192.168.8.8 (ethernet, primary) reserved on Brume 3. 192.168.8.7 = wifi (disabled, break-glass only). DHCP + MAC-match in netplan.
- **UPS:** CyberPower CP1500PFCLCD. USB data cable will run to **RPi5** (NUT server); server01 = NUT netclient. Cable not yet connected — low priority.

### Drives

| Drive | by-id | Role | Status |
|---|---|---|---|
| Samsung 970 EVO 500GB NVMe | nvme-Samsung_SSD_970_EVO_500GB_S466NX0KA14327N | Ubuntu OS. Spare ~400GB use = **undecided** (Decision 1) | OS installed |
| Inland 1TB NVMe | nvme-Inland_QN450_NVMe_SSD_IB26AC1000P03985 | `/var/lib/docker` (ext4 "docker", UUID 4cea74ce-20a4-4969-a3be-fff04e7ac4dd) | DONE, mounted, boot-verified ×2 |
| WD Red 4TB (was source) | ata-WDC_WD40EFZX-68AWUN0_WD-WX32D124UYHS | **`/mnt/backup`** (ext4 "backup", UUID 43a16c87-453d-4508-a7e6-d60ae7ce43f9) — rclone OneDrive target | **LIVE, reformatted 07-09, fstab by-UUID+nofail, boot-verified, owned ian:ian** |
| WD Red 4TB (dest) | ata-WDC_WD40EFZX-68AWUN0_WD-WX62D12CCFRX | Plex media → `/mnt/media` (ext4) | LIVE under Plex. dvr+music+videos, owned ian:ian. CRC 3343 confirmed stable |
| WD Black 2TB | ata-WDC_WD2003FZEX-00SRLA0_WD-WMC6N0P1DV7X | Frigate recordings → `/mnt/frigate` (ext4 "frigate") | DONE, mounted, boot-verified ×2 |
| WD Blue 4TB (SMR) | — | TBD — prior ata4 errors; retirement candidate | PHYSICALLY PULLED — diagnose later |

**Two identical WD Red 4TB drives — do not confuse them.** They share model string `WDC_WD40EFZX-68AWUN0`; only the serial differs. **WX32D124UYHS = `/mnt/backup`** (the former source, now reformatted). **WX62D12CCFRX = `/mnt/media`** (Plex, holds the verified media — never reformat without extreme care). Always resolve by serial before any destructive drive op.

## Open Decisions

1. **Samsung 500GB spare space (~400GB).** STILL OPEN. Candidates: Immich originals (`/mnt/photos`), Immich Postgres + ML cache, nothing (keep OS drive clean). Decide before Phase 6. Photos on OS drive means OS reinstalls need care, but it's the only fast storage unallocated and the old library fit in 500GB.
2. **Old Immich data — CONFIRMED PRESERVED.** Crucial 500GB SATA SSD in ITX Windows PC not reformatted; ext4 intact. Case stays closed. Copy method: **WSL2 `wsl --mount`** (elevated PowerShell → `wsl --mount \\.\PHYSICALDRIVEn --partition 1 --type ext4`, find n via `Get-Disk`), then rsync from WSL2 to server01. `wsl --unmount` after. Deferred to Phase 6. Check SSD's old `/mnt/backup/migration` for pg_dump while mounted; restore if found else fresh DB. NOTE: the OneDrive `immich` folder is also being pulled to `/mnt/backup` by the current rclone sync, so cloud originals will be staged locally regardless — reconcile the two sources at Phase 6.
3. **WD Blue fate.** SMART long test when convenient. SMR + prior SATA errors = likely retire. Currently pulled from case.
4. **Plex identity — RESOLVED 2026-07-08: fresh identity, claimed.** Old watch history not migrated.

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
- [x] **/mnt/backup — DONE 07-09.** Source Red (WX32D124UYHS) reformatted exFAT→ext4 "backup", UUID 43a16c87-453d-4508-a7e6-d60ae7ce43f9, fstab by-UUID+nofail, `mount -a` + `daemon-reload`, chown ian:ian, boot-verified.

### Phase 2 — Plex data verification — COMPLETE (2026-07-08)
- [x] Source Red mounted read-only
- [x] music + videos rsynced source→dest (322GB)
- [x] Manifest compare PASSED — md5 (not xxh128 as originally planned): full fresh source re-hash vs dst.manifest via comm, LC_ALL=C, zero mismatches
- [x] Dest Red CRC rechecked — 3343, unchanged through rsync + two full drive re-hashes. Bad-cable theory CONFIRMED; drive healthy
- [x] chown -R ian:ian /mnt/media
- [x] Source Red released, then reformatted to /mnt/backup 07-09 (Phase 1)

### Phase 3 — Docker foundation — COMPLETE
- [x] NVIDIA driver 580-server, nvidia-smi verified
- [x] Docker Engine 29.6.1 (official repo), compose plugin, data-root on Inland
- [x] Container Toolkit 1.19.1, runtime configured, GPU-in-container smoke test passed
- [x] Compose layout /opt/stacks/<service>/ established
- [x] ian added to docker group

### Phase 4 — Plex (Docker) — COMPLETE / LIVE (2026-07-09)
- [x] compose.yaml at /opt/stacks/plex/ (lscr.io/linuxserver/plex, host net, NVENC caps video,compute,utility, /mnt/media ro, tmpfs transcode, config bind mount)
- [x] Launched + claimed successfully (fresh identity — Decision 4 resolved)
- [x] Libraries added: /media/music, /media/videos, /media/dvr — **initial scan COMPLETE**
- [x] LAN access confirmed via http://192.168.8.8:32400/web — no timeout recurrence
- [x] Transcoder temp dir = /transcode (tmpfs); HW accel on; transcoding device pinned to RTX 2070 (explicit over Automatic — GPU failure should be loud, not silent CPU fallback)
- [x] External access DISABLED — deliberate. Remote Plex via Tailscale is the future path; revisit at a later date
- [x] **HW transcode VERIFIED — "(hw)" confirmed in dashboard 07-09.** Phase fully closed.

### Phase 5 — Frigate (Docker) — IN PROGRESS (full detail in companion doc `frigate.md`)

**Detector correction:** the detector is **ONNX (TensorRT-accelerated)**, NOT the old `tensorrt` detector type — that standalone type was **removed for desktop Nvidia GPUs in 0.16+**. Current path = image `0.17.2-tensorrt` (ships TensorRT libs) + `type: onnx` detector + YOLOv9-s @ 320 model (manual ONNX export — Nvidia does NOT auto-download models). This supersedes the original "TensorRT detector" plan wording.

- [x] Stack files authored + pushed to repo: `compose.yaml`, `config/config.yml`, `env.example` (template; real `.env` is local-only, gitignored)
- [x] **E1 Zoom streams ffprobe-VERIFIED (2026-07-09):** main `h264Preview_01_main` = **HEVC 3840×2160 + AAC**; sub `h264Preview_01_sub` = **H.264 640×360 + AAC**; login `admin`. (Gotcha: the `h264Preview`-prefixed main path actually carries H.265 — Reolink's stream name lies about the codec. There is no `h265Preview_` path.)
- [x] MQTT `frigate` user confirmed present in Mosquitto add-on config (password in password manager)
- [ ] Run the bring-up runbook (`frigate.md`): create `/opt/stacks/frigate` + `config/model_cache` → create `.env` from `env.example` (chmod 600) → export YOLOv9-s ONNX (docker build, in tmux) → `docker compose up -d` → watch logs for TensorRT engine build + go2rtc connect + MQTT + the one-time admin password → verify a person event
- [ ] Re-point HA Frigate integration to http://192.168.8.8:8971
- [ ] (Later) Amcrest floodlight cam — driveway. NOTE: Dahua-lineage RTSP (`/cam/realmonitor?channel=1&subtype=0` main, `subtype=1` sub) — NOT Reolink path syntax.

### Phase 6 — Immich — ON HOLD (Decision 1 gates it)
- [ ] Resolve Decision 1 (photo storage location)
- [ ] ITX: WSL2 `wsl --mount` Crucial SSD, rsync originals → server01, check pg_dump, unmount
- [ ] Reconcile with OneDrive-staged originals now landing in /mnt/backup (see Decision 2)
- [ ] Compose stack; originals at decided location; DB on fast storage
- [ ] Restore pg_dump if found, else fresh library + re-import

### Phase 7 — rclone (Mealie already done)
- [x] Mealie compose stack LIVE (see Services / Session Log)
- [x] **rclone v1.74.4 installed** via official install script (`curl https://rclone.org/install.sh | sudo bash`) — deliberately NOT snap and NOT apt's stale 1.60. Static binary in /usr/bin.
- [x] **OneDrive remote authed** — remote name `onedrive`, config at `~/.config/rclone/rclone.conf`, drive = OneDrive personal `2A77BE67D195BC46`. Token-paste method (headless box → `rclone authorize "onedrive"` on Windows PC → paste blob back). Refresh token persists; access token is 1h and auto-renews; refresh token only dies after ~90 days unused (nightly timer will prevent that).
- [x] **Initial full-drive copy RUNNING** in tmux session `onedrive`: `rclone copy onedrive: /mnt/backup --exclude "/Personal Vault/**" --progress --log-file=/home/ian/rclone-onedrive-initial.log --log-level INFO`
  - **`copy` not `sync`** — a pull-down backup must never propagate a cloud-side deletion (accidental, ransomware, sync glitch) to the local copy. Trade-off accepted: local retains files removed from OneDrive (drift over time), which is the safe direction for a backup.
  - `--exclude "/Personal Vault/**"` always — Vault is server-side locked; exclusion avoids harmless-but-noisy errors in logs.
- [ ] **After initial sync completes:** wrap as systemd Type=oneshot (no `Restart=`), `RequiresMountsFor=/mnt/backup`, timer-driven (nightly). Reuse the `copy` + Personal Vault exclude semantics above.

### Phase 8 — Monitoring & UPS
- [ ] node_exporter (:9100) + smartctl_exporter (:9633) for RPi5 Prometheus scrape (targets = 192.168.8.8)
- [ ] NUT: RPi5 = server (USB cable), server01 = netclient early shutdown
- [ ] MikroTik CSS610 config backup (loose end)

### Phase 9 — Cleanup & hardening
- [ ] **ROTATE Mealie Postgres password** — the current value was committed to git history in an earlier version of this doc. Private repo + low-value LAN password = not urgent, but history is permanent so the clean fix is rotation, not deletion. Generate a new alphanumeric password, update BOTH the `mealie` and `postgres` services in /opt/stacks/mealie/compose.yaml, `docker compose down && up -d` (Postgres keeps the old password baked into pgdata, so this also requires either recreating the DB or `ALTER USER mealie PASSWORD` inside the container — sequence to be worked out at the time). Update the password manager entry. After rotating, the old committed value is dead and harmless.
- [ ] WD Blue decision executed
- [ ] UFW baseline (SSH + service ports) — NOT YET configured; nothing firewalled currently
- [ ] Unattended-upgrades
- [ ] Media library cleanup: (copy N) duplicate DVR files + macOS cruft (.DS_Store, AlbumArt_*.jpg) — cosmetic. Verify confirmed dest carries benign extras (dupe-hash art files, "* 2.mp3" suffixes, (copy N) DVR) — dedupe at leisure, after Plex scan settles
- [ ] Plex external/remote access — currently DISABLED by design. Future: Tailscale-based remote access, no port forwarding. Design at a future session
- [ ] Document final fstab + compose tree here

## Live Services

- **Mealie** — http://192.168.8.8:9925 — /opt/stacks/mealie/ — mealie v3.20.1 + postgres:17, bind mounts (data + pgdata), ALLOW_SIGNUP=false. Credentials stored in password manager (DB + web admin, separate entries).
- **Plex** — http://192.168.8.8:32400/web — /opt/stacks/plex/ — lscr.io/linuxserver/plex, host net, NVENC (RTX 2070, pinned, HW confirmed), /mnt/media ro, tmpfs /transcode. Claimed, fresh identity. External access OFF.
- **Frigate** — (bring-up pending) — /opt/stacks/frigate/ — 0.17.2-tensorrt image, ONNX detector, go2rtc restream, recordings → /mnt/frigate. UI will be at http://192.168.8.8:8971 (authenticated). Full detail + runbook in `frigate.md`.
- **rclone (host service, not a container)** — remote `onedrive` → `/mnt/backup`. Config `~/.config/rclone/rclone.conf` (NOT committed — holds refresh token). Drive = OneDrive personal `2A77BE67D195BC46`. Initial `copy` sync in progress; systemd timer to follow.

## Session Log

- 2026-07-06 — Plan created. Ubuntu reinstalled fresh.
- 2026-07-07 — Confirmed Crucial SSD ext4 intact (WSL2 copy method set). veracrypt/pers_container (100GiB) copied to microSD + xxh128 VERIFIED (7cdf603e827ae26b7a6dbe4775ba820d), card pulled/labeled. Black wiped → ext4 /mnt/frigate. dest Red → /mnt/media. Inland → /var/lib/docker. All fstab by-id+nofail, boot-tested (NVMe letters swapped, by-id held). Docker 29.6.1 + NVIDIA 580 + Container Toolkit, GPU-in-container verified. Mealie LIVE. Plex stack staged (not launched). music+videos rsynced to /mnt/media.
- 2026-07-08 (day) — Manifest verify detour: dst.manifest completed (10,230 files) but initial diff/comm attempts returned confusing all-mismatch results. Root causes found one at a time: (1) src.manifest was STALE — its hashes disagreed with the live source drive itself; (2) a second full rsync ran during diagnosis (exact history murky — see training item); (3) comm needs LC_ALL=C on both inputs; (4) fresh source re-hash omitted the ./ prefix strip, false-failing again. Spot checks along the way proved live src == live dst byte-for-byte and zero missing paths.
- 2026-07-08 (evening) — **Phase 2 CLOSED, Plex LIVE.** Definitive verify: fresh md5 re-hash of full source (./ stripped, LC_ALL=C) vs dst.manifest = ZERO mismatches. CRC 3343 stable through all of it → bad-cable confirmed, drive healthy. chown /mnt/media → ian. Plex launched, claim token worked first try, fresh identity, 3 libraries scanning, LAN access clean. Transcoder: temp dir → /transcode tmpfs, device pinned to 2070, external access OFF (deliberate; Tailscale later). Hard Rule 10 added (tmux + verify gotchas).
- 2026-07-09 (early AM) — **Backup drive live + OneDrive backup stood up.** HW transcode "(hw)" confirmed → Phase 4 fully closed; Plex initial scan finished. Gave go-ahead to reformat source Red (WX32D124UYHS): exFAT→ext4 "backup", UUID 43a16c87..., /mnt/backup, fstab by-UUID+nofail, chown ian. Hard Rule 2 fully retired. Preventative reboot to validate the new fstab line under real boot — passed; **NVMe letters swapped a 2nd time (docker root nvme1n1→nvme0n1), UUID/by-id held everything** (Hard Rule 4 reinforced). rclone v1.74.4 installed via official script (rejected snap + apt 1.60). OneDrive remote authed after a couple of false starts (wizard/authorize run on wrong machine first; clean redo with token-paste worked). Chose `copy` over `sync` (documented in Phase 7 + Hard Rule 1 rclone exception). Initial full-drive `copy` launched in tmux `onedrive`, Personal Vault excluded, logging to ~/rclone-onedrive-initial.log. **STOPPED FOR THE NIGHT — done until OneDrive initial sync completes.**
- 2026-07-09 (evening) — **Frigate bring-up started; split into companion doc `frigate.md`.** Confirmed current Frigate = 0.17.2. Corrected the detector approach: ONNX detector on the `-tensorrt` image (standalone TensorRT detector removed for desktop Nvidia in 0.16+); model is a manually-exported YOLOv9-s ONNX (Nvidia doesn't auto-download). Authored + pushed `compose.yaml`, `config.yml`, `env.example`. **ffprobe-verified E1 Zoom streams before launch:** main `h264Preview_01_main` = HEVC 4K + AAC, sub `h264Preview_01_sub` = H.264 640×360 + AAC, login `admin` — corrected config (main path was assumed `h265Preview`; the h264-prefixed path actually carries HEVC). Adopted per-service companion-doc convention (compose + config in repo, secrets in `.env`, data on disk); Plex/Mealie docs to retrofit later. Stopped before running the bring-up runbook. OneDrive initial sync still running (expected done shortly after midnight).

### Next-session pickup (START HERE)
1. **Check the OneDrive initial sync.** Reattach: `tmux attach -t onedrive`. If the prompt is back (job done), review the tail of `~/rclone-onedrive-initial.log` for errors/skips. If still running, let it finish.
2. **Once sync is clean:** build the rclone systemd oneshot + nightly timer (Phase 7 last box) — `copy`, Personal Vault exclude, `RequiresMountsFor=/mnt/backup`, no `Restart=`.
3. **Frigate bring-up (Phase 5)** — camera verified, files in repo. Follow the runbook in `frigate.md`: create `/opt/stacks/frigate` + `config/model_cache`, create `.env` from `env.example` (chmod 600), export YOLOv9-s ONNX (docker build, tmux), `docker compose up -d`, watch logs for engine build + go2rtc + MQTT + one-time admin password, verify a person event. Then re-point the HA integration.
- Owed: **training session** — tmux fundamentals, canonical verify workflow, iPad/Termius drills — so the 07-08 verify confusion never recurs.

### Quick refs
- OneDrive sync check: `tmux attach -t onedrive` (Ctrl-b then d to detach without stopping it); log at `~/rclone-onedrive-initial.log`; sanity: `rclone lsd onedrive:` lists cloud folders.
- rclone re-auth if refresh token ever lapses (~90 days unused): `rclone config reconnect onedrive:` + one browser dance.
- Canonical verify pattern (for any future copy): hash both sides identically (same ./ strip, same tool), `LC_ALL=C sort` both, `comm -23 src dst | wc -l` → 0 = clean. Always in tmux, stderr to a file.
- Plex HW check (already done): play video, force lower quality, dashboard shows "(hw)".
- Two identical WD Reds: WX32D124UYHS = /mnt/backup, WX62D12CCFRX = /mnt/media. Resolve by serial before any drive op.
- Frigate RTSP probe (any camera): `docker run --rm --entrypoint ffprobe linuxserver/ffmpeg -v error -show_entries stream=codec_name,width,height -of default=noprint_wrappers=1 "rtsp://USER:PASS@IP:554/PATH"`. A 401 is often a stray space or unescaped password symbol, not real auth failure.
- E1 Zoom streams: main `h264Preview_01_main` (HEVC 4K), sub `h264Preview_01_sub` (H.264 640×360), login `admin`. Path name says h264 but main is H.265 — probe, don't trust the name.

## To-Do: Video Ingest Share (SMB on server01)

**Goal:** LAN landing spot for iOS-shot video — fast access for editing from other devices.

- [ ] Deploy Samba as Docker container (container on Inland NVMe like everything else)
- [ ] Create bind-mount target on OS drive (970 EVO): `/srv/video-inbox`
  - Subdirs: `inbox/` (raw phone dumps), `projects/<name>/` (active edits), `outbox/` (finished, awaiting move)
- [ ] Create SMB user/credentials → store in Proton Pass
- [ ] iOS: connect via Files app → `smb://192.168.8.8`
- [ ] iOS: Settings → Photos → "Transfer to Mac or PC: Keep Originals" (avoid silent HEVC→H.264 transcode)
- [ ] Test share from Windows ITX PC (bundle with pending brume-backup Samba test)

**Rules:**
- Working area, NOT a library — nothing here is backed up
- Keepers → OneDrive/Immich pipeline; finished edits → Plex
- Footage sitting in `inbox/` = at-risk/unarchived by definition

**Timing:** No dependency on Immich/Phase 6.
