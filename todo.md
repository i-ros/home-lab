# To-Do

Open action items across the home lab. Grouped by area. Check items off in place; when an item is done, either delete it or move a one-line summary into `maintenance.md`'s Log — don't leave stale checked boxes around.

See `decisions.md` for the "why" behind any approach referenced here, and `parts-inventory.md` for hardware status.

---

## Frigate (camera/NVR)

- [ ] **Update `config.yml` camera name** — still uses `living_room`; rename to `reolink_e1_zoom_eth` (cameras were physically relocated, dropped location-based naming — see `decisions.md`) before first bring-up.
- [ ] **Run the Frigate bring-up runbook** (never yet executed — do this on server01, one command at a time, in tmux):
  1. `mkdir -p /opt/stacks/frigate/config/model_cache`; place `compose.yml` at `/opt/stacks/frigate/`, `config.yml` at `/opt/stacks/frigate/config/`
  2. Create `.env` from `env.example`, fill real MQTT/RTSP passwords, `chmod 600 /opt/stacks/frigate/.env`
  3. Export YOLOv9-s ONNX model (docker build command in `decisions.md` under Frigate) — confirm `yolov9-s-320.onnx` lands in `model_cache/`
  4. `docker compose up -d`, `docker compose logs -f` — watch for TensorRT engine build, go2rtc connecting, detector on GPU (not CPU), MQTT connected, and the one-time auto-generated admin password (save it)
  5. Verify at `http://192.168.8.8:8971` — live view, walk-in triggers a `person` event, System Metrics shows GPU inference ~10-20ms
- [ ] **Re-point HA Frigate integration** to `http://192.168.8.8:8971` once bring-up is verified.
- [ ] ffprobe the E1 Pro (192.168.8.103) to get its real RTSP stream paths — do not assume they match the E1 Zoom's.
- [ ] Add `reolink_e1_pro_eth` camera block to `config.yml` once its paths are verified (own go2rtc streams + camera block).
- [ ] Hook up the Reolink doorbell — new go2rtc streams + camera block, plus doorbell-specific config (button-press event, two-way audio).
- [ ] Create a dedicated least-privilege Reolink user per camera (web UI → System → User Management) and swap `admin` out of configs/passwords.
- [ ] Decide audio-recording policy per camera — some US states are two-party-consent for audio; both E1 Zoom streams currently carry AAC audio that gets recorded by default.
- [ ] Per-camera zones + masks once physical locations are finalized — cuts false positives.
- [ ] Decide on birdseye view and whether to expose it.
- [ ] Revisit dropping port 5000 (unauthenticated UI) once auth + HA integration are settled.
- [ ] Tune retention per camera once real disk-usage/day is observed (currently 3-day continuous / 30-day events, a starting guess).
- [ ] (Optional, only if camera count outpaces one detector) add a second ONNX detector instance.

## Immich — on hold

Blocked on the Samsung 500GB spare-space decision (see `decisions.md` → Open Decisions).

- [ ] Resolve the photo-storage-location decision.
- [ ] From the Windows ITX PC: WSL2 `wsl --mount` the old Crucial SSD, rsync originals → server01, check for a `pg_dump` in its old `/mnt/backup/migration`, `wsl --unmount` after.
- [ ] Reconcile old-SSD originals with the OneDrive `immich` folder already being pulled to `/mnt/backup` by rclone — same photos may exist in both places.
- [ ] Stand up the compose stack once storage location is decided; DB on fast storage.
- [ ] Restore the `pg_dump` if one was found, else fresh library + re-import.

## Monitoring & UPS

- [ ] `node_exporter` (:9100) + `smartctl_exporter` (:9633) on server01 for the RPi5 Prometheus scrape (target = 192.168.8.8).
- [ ] Wire up NUT: RPi5 = server (USB cable from the CyberPower CP1500PFCLCD), server01 = netclient for early shutdown. USB cable not yet connected.
- [ ] Back up the MikroTik CSS610 switch config (loose end, low priority).

## Networking / File Sharing (SMB + Tailscale)

**Goal:** LAN landing spot for iOS-shot video (fast access for editing from other devices) and general native file access for iPad/other devices. Working area only — nothing under it is backed up; keepers go through the OneDrive/Immich pipeline, finished edits move to Plex.

- [ ] Per-user SMB credentials (no guest access) — store in password manager. Currently a single shared `ian` account across all shares; revisit if finer-grained access is ever needed.
- [x] iPad connects to the `media` share and streams music from it — confirmed 2026-08-11.
- [ ] iOS: Settings → Photos → "Transfer to Mac or PC: Keep Originals" to avoid a silent HEVC→H.264 transcode when dropping video into `editing/inbox`.
- [ ] Test the share from the Windows ITX PC.
- [ ] Once the stack feels valuable enough to want verified remote reach: add the iPad to the existing tailnet, confirm MagicDNS resolves `server01`, test the SMB share over Tailscale (not the LAN IP), and confirm the same path works for Plex remote access (external Plex access is currently OFF by design — Tailscale is the intended future path, no port forwarding).

Note: this replaces the dead Proton Drive Files integration — Proton dropped iOS Files support in their current SDK build (per their dev response; re-add is "future version," no ETA).

## Cleanup & Hardening

- [ ] **Rotate the Mealie Postgres password** — the current value was committed to git history in an earlier version of the old planning doc. Private repo + low-value LAN password = not urgent, but rotation (not just deletion) is the correct fix since git history is permanent. Generate a new password, update both the `mealie` and `postgres` services in `/opt/stacks/mealie/compose.yaml`, then either recreate the DB or run `ALTER USER mealie PASSWORD` inside the container (sequence TBD at execution time) before `docker compose down && up -d`. Update the password manager entry after.
- [ ] Execute the WD Blue decision once the SMART long test is run (see `decisions.md` → Open Decisions) — SMR + prior SATA errors make retirement likely.
- [ ] UFW baseline (SSH + service ports) — nothing is firewalled on server01 currently.
- [ ] Unattended-upgrades.
- [ ] Media library cleanup: `(copy N)` duplicate DVR files + macOS cruft (`.DS_Store`, `AlbumArt_*.jpg`) on `/mnt/media` — cosmetic, dedupe at leisure once Plex's scan settles.
- [ ] Document the final fstab + compose tree.

## Process

- [ ] Training session owed: tmux fundamentals, the canonical verify workflow (hash both sides identically, `LC_ALL=C sort`, `comm -23 src dst | wc -l` → 0 = clean), iPad/Termius drills — so a repeat of the 2026-07-08 manifest-verify confusion doesn't happen again.
