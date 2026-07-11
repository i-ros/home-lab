# Maintenance Log

Running history of what happened, plus the trap list of gotchas learned along the way. New log entries go at the top (reverse chronological). New gotchas get appended to their section as they're discovered — don't let this list go stale, it's the whole point of keeping it.

---

## Log

- **2026-07-10** — Plan reset on Frigate naming. The bring-up runbook has still never actually been executed (no `docker compose up` yet). Cameras were physically relocated, so location-based naming (`living_room`) was dropped in favor of hostname-style names since final placement isn't decided yet: **E1 Zoom** → `reolink_e1_zoom_eth` (192.168.8.106, same unit, just renamed) and **E1 Pro** → `reolink_e1_pro_eth` (192.168.8.103, new, streams not yet probed). A Reolink doorbell is owned and will be hooked up soon. The Amcrest floodlight driveway-cam plan was dropped entirely. Repo cleanup: removed a typo'd duplicate doc (`figate.md`) — no content lost, full history remains in git at commit `0c51da0`.
- **2026-07-09 (evening)** — Frigate bring-up started. Confirmed current Frigate version = 0.17.2. Corrected the detector approach: ONNX detector on the `-tensorrt` image, not the standalone `tensorrt` detector type (removed for desktop Nvidia GPUs in 0.16+). Model is a manually-exported YOLOv9-s ONNX (Nvidia doesn't auto-download models, unlike Rockchip/MemryX). Authored `compose.yml`, `config.yml`, `env.example` for the first camera. MQTT `frigate` user confirmed present in the Mosquitto add-on config. **ffprobe-verified the E1 Zoom streams before launch:** main = `h264Preview_01_main` (actually HEVC 3840×2160 + AAC — the path name lies about the codec, see Gotchas), sub = `h264Preview_01_sub` (H.264 640×360 + AAC), login `admin`. Config was corrected accordingly (had assumed a `h265Preview_` path that doesn't exist). Stopped before running the bring-up runbook. rclone's OneDrive initial sync was running in the background, unrelated to Frigate.
- **2026-07-09 (early AM)** — Backup drive went live and OneDrive backup was stood up. Plex hardware transcode confirmed ("(hw)" in dashboard) — Plex phase fully closed, initial library scan finished. Reformatted the former source Red (serial `WX32D124UYHS`) from exFAT to ext4 → `/mnt/backup`, UUID-based fstab entry with `nofail`, owned `ian:ian`. A preventative reboot to validate the new fstab line under real conditions passed — and NVMe drive letters swapped a second time (Docker root moved `nvme1n1`→`nvme0n1`); by-id/UUID mounts held everything correctly both times this has happened. Installed rclone v1.74.4 via the official install script (deliberately not snap, not apt's stale 1.60 build). Authed the OneDrive remote (`onedrive`) after a couple of false starts, via the token-paste method. Chose `copy` over `sync` for the backup direction — a pull-down backup must never propagate a cloud-side deletion to the local copy. Launched the initial full-drive `copy` in a `tmux` session named `onedrive`, excluding `/Personal Vault/**`, logging to `~/rclone-onedrive-initial.log`. Stopped for the night pending that sync finishing.
- **2026-07-08 (evening)** — Plex phase closed, LIVE. Definitive verify: fresh md5 re-hash of the full source (with the `./` prefix stripped, `LC_ALL=C`) against the destination manifest came back with zero mismatches. The destination Red's CRC (3343) stayed stable through the rsync and two full drive re-hashes, confirming the earlier bad-cable theory (drive itself is healthy). `chown -R ian:ian /mnt/media`. Plex launched, claim token worked first try, fresh identity (no old watch history migrated), three libraries scanning, LAN access clean.
- **2026-07-08 (day)** — Manifest-verify detour. `dst.manifest` completed (10,230 files) but the initial diff attempts returned confusing all-mismatch results. Root causes, found one at a time: (1) the source manifest was stale — its hashes disagreed with the live source drive; (2) a second full rsync ran during diagnosis (exact sequence never fully reconstructed); (3) `comm` needs `LC_ALL=C` on both inputs or sort order mismatches silently break the diff; (4) a fresh source re-hash omitted the `./` prefix strip, causing a false failure. Spot checks throughout confirmed live source and live destination were actually byte-for-byte identical with zero missing paths — the tooling was lying, not the data.
- **2026-07-07** — Confirmed the old Crucial SSD (from the retired ITX Windows PC) still has its ext4 partition intact. The VeraCrypt personal container (100GiB) was copied to a microSD and xxh128-verified (`7cdf603e827ae26b7a6dbe4775ba820d`), then the card was pulled and labeled. WD Black wiped → ext4 → `/mnt/frigate`. Dest Red → `/mnt/media`. Inland NVMe → `/var/lib/docker`. All fstab entries switched to by-id + `nofail` and boot-tested — NVMe drive letters swapped across the reboot, and the by-id mounts held (first of two times this has now happened). Docker 29.6.1 + NVIDIA driver 580 + Container Toolkit installed, GPU-in-container verified. Mealie went live. Plex stack staged (not yet launched). Music + videos rsynced to `/mnt/media`.
- **2026-07-06** — Plan created. Ubuntu Server reinstalled fresh on server01.

---

## Gotchas / Known Issues

**Frigate / cameras**
- **Reolink stream paths lie about codec.** Both E1 Zoom streams use the `h264Preview_` prefix, but the *main* stream actually encodes HEVC/H.265 4K — verified via ffprobe. There is no `h265Preview_` path on this camera. Never trust the path name for codec; always probe it. This was the #1 near-miss on the Frigate bring-up.
- **RTSP `401 Unauthorized` is often not a real auth failure.** A stray space after `admin:` or an unescaped special character in the password breaks URL parsing and surfaces as a 401. Percent-encode symbols in RTSP passwords before debugging further.
- **Explicit `detect.width`/`detect.height` required.** Frigate 0.17 changed camera-resolution auto-detection; some cameras make Frigate hang on startup if these aren't set explicitly.
- **Model path must match the actual export filename exactly**, or Frigate errors at start. Re-exporting a different model size/imgsz means updating both the export command and `model.path` in config.
- **The `.trt` engine is GPU/driver-specific.** After a driver bump or a future GPU swap, delete the cached TensorRT engine and let Frigate rebuild it on next start.
- **H.265 live view over WebRTC can be flaky in-browser.** Detection runs on the H.264 sub-stream so detection itself is unaffected — if only the *main* live view misbehaves, that's why, not a detection problem.
- **YAML pasted from a rich-text source can silently corrupt config.** Smart quotes and non-breaking spaces from copy-paste break YAML parsing without an obvious error. Create/edit config with a plain editor (nano) or a clean ASCII heredoc.
- **Both current camera streams carry AAC audio**, and recording captures audio by default — relevant if operating in a two-party-consent state (see `todo.md` → Frigate).

**Storage / drives**
- **NVMe drive letters are not stable across reboots** — proven twice on this box (Docker's data drive moved from `nvme1n1` to `nvme0n1` on one reboot). Only `/dev/disk/by-id/` or `UUID=` fstab entries survive this; never reference `/dev/sdX`/`/dev/nvmeXnY` directly.
- **Two visually-identical WD Red 4TB drives exist in this box** — same model string `WDC_WD40EFZX-68AWUN0`, only the serial differs. `WX32D124UYHS` = `/mnt/backup`. `WX62D12CCFRX` = `/mnt/media` (holds the verified Plex library — never reformat without resolving by serial first).

**Verify workflow** (from the 2026-07-08 manifest-verify detour above — the process this prevents)
- Strip the same `./` path prefix identically on both manifests being compared, or every line will false-mismatch.
- Both inputs to `comm`/`sort` need `LC_ALL=C`, or locale-dependent sort order breaks the diff.
- `comm` compares whole lines — hash *and* path both have to match, so a manifest format drift between two runs looks like total data loss even when the data is fine.
- Re-hash from the live source, not a manifest that might be stale relative to it, whenever verification results look suspicious.

---

## Quick Reference

- **Canonical file-verify pattern** (any future bulk copy): hash both sides with the same tool and the same `./` strip, `LC_ALL=C sort` both manifests, `comm -23 src dst | wc -l` → 0 means clean. Always run in tmux with stderr captured to a file.
- **Frigate RTSP stream probe** (any camera): `docker run --rm --entrypoint ffprobe linuxserver/ffmpeg -v error -show_entries stream=codec_name,width,height -of default=noprint_wrappers=1 "rtsp://USER:PASS@IP:554/PATH"`.
- **OneDrive sync check:** `tmux attach -t onedrive` (Ctrl-b then d to detach without stopping); log at `~/rclone-onedrive-initial.log`; sanity check with `rclone lsd onedrive:`.
- **rclone re-auth** if the refresh token ever lapses (~90 days unused): `rclone config reconnect onedrive:` plus one browser round-trip.
- **Plex HW-transcode check:** play a video, force a lower quality, dashboard should show "(hw)".
