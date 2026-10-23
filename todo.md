# To-Do

Open action items across the home lab. Grouped by area. Check items off in place; when an item is done, either delete it or move a one-line summary into `maintenance.md`'s Log — don't leave stale checked boxes around.

See `decisions.md` for the "why" behind any approach referenced here, and `parts-inventory.md` for hardware status.

---

## Frigate (camera/NVR)

Live. All three cameras running — `reolink_e1_zoom`, `reolink_doorbell`, `reolink_e1_pro_wifi` (E1 Pro relocated to WiFi, see `maintenance.md` 2026-10-22). GPU inference, HA integration, and `person`-event alerting all confirmed working.

- [ ] Create a dedicated least-privilege Reolink user per camera (web UI → System → User Management) and swap `admin` out of configs/passwords.
- [ ] Decide audio-recording policy — some US states are two-party-consent for audio; all three cameras' main streams carry AAC audio that gets recorded by default.
- [ ] Per-camera zones + masks now that physical locations are settled — cuts false positives.
- [ ] Decide on birdseye view and whether to expose it.
- [ ] Tune retention per camera once real disk-usage/day is observed (currently 3-day continuous / 30-day events, a starting guess).
- [ ] (Optional, only if camera count outpaces one detector) add a second ONNX detector instance.
- [ ] Harden `nvidia-persistenced` against silent breakage — it's a systemd `static` unit with no auto-restart, so if it dies (as it did 2026-09-11) the GPU looks fine until something tries to use it, then every GPU container fails to start at creation. See `maintenance.md` 2026-10-22 and Gotchas.

## Immich

Live as of 2026-10-23 (`v3.3.5`, CPU-only ML — see `decisions.md`). Storage decision resolved: Postgres + ML cache on the OS SSD (`/opt/stacks/immich/`), native upload library on `/mnt/backup/immich-library/`, and the existing curated photos (`/mnt/backup/onedrive/Pictures/immich/` — `professional`/`wedding`/`years`, 39GB) added as a read-only External Library rather than duplicated.

- [ ] Confirm the External Library scan finished and the asset count looks right (39GB across the 3 curated folders).
- [ ] Set the External Library rescan schedule (Administration → Settings → External Library) to ~04:00 daily, safely after the rclone OneDrive pull (~03:00–03:30) finishes each night.
- [ ] **Build a reverse backup job for new phone uploads.** Nothing currently protects `/mnt/backup/immich-library/` off this one drive — the existing rclone job only pulls *from* OneDrive. Needs a second rclone job pushing that folder (or just its originals) up to a dedicated OneDrive folder, on its own timer, mirroring the existing pattern.
- [ ] Set up the Immich mobile app on your phone (and your wife's, later) for automatic photo backup — this is what will actually start filling `immich-library/`.
- [ ] Decide if any exclusion patterns are needed on the curated External Library (e.g. raw/sidecar files) — not yet assessed.

## Monitoring & UPS

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
- [ ] Unattended-upgrades.
- [ ] Media library cleanup: `(copy N)` duplicate DVR files + macOS cruft (`.DS_Store`, `AlbumArt_*.jpg`) on `/mnt/media` — cosmetic, dedupe at leisure once Plex's scan settles.
- [ ] Document the final fstab + compose tree.

## Process

- [ ] Training session owed: tmux fundamentals, the canonical verify workflow (hash both sides identically, `LC_ALL=C sort`, `comm -23 src dst | wc -l` → 0 = clean), iPad/Termius drills — so a repeat of the 2026-07-08 manifest-verify confusion doesn't happen again.
