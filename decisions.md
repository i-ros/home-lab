# Decisions

The "why" behind how the home lab is built — durable rules, per-service design calls, and decisions still open. Update in place when something changes; don't leave contradictory entries.

---

## Hard Rules (apply lab-wide)

1. **Everything runs in Docker.** No native service installs, no exceptions without a documented reason here. **Documented exception: rclone** — it runs as a host-level systemd oneshot + timer, not a container, because it needs direct host filesystem access to `/mnt/backup` and a persistent auth config; containerizing a scheduled pull-down buys nothing. Same call made on a prior retired box.
2. **All fstab entries use stable IDs** (`/dev/disk/by-id/` or `UUID=`) with `nofail` on data drives — never `/dev/sdX`/`/dev/nvmeXnY`, letters shuffle between boots. Proven twice on server01 (see `maintenance.md` → Gotchas); by-id/UUID held every mount both times.
3. **Architectural decisions before execution.** If a step forces an undecided choice, stop and decide first (that's what this file is for).
4. **If a step goes wrong, cleanup/undo instructions come first**, before trying a new approach.
5. **Compose convention:** one directory per stack at `/opt/stacks/<service>/`, config via bind mounts (not named volumes) so a plain rsync of `/opt/stacks` backs up every service. PUID/PGID 1000, TZ `America/Denver`, image versions pinned (no `latest` on databases).
6. **Working practice:** one command at a time, review output before proceeding, assume only the stated command ran. When a pattern emerges that could be standardized or made safer, call it out proactively rather than just executing — the by-id fstab rule and the `/opt/stacks` convention both came from this habit and prevented real mishaps.
7. **GitHub / secrets discipline** — two separate rules:
   - **Never commit credentials** — passwords, tokens, API keys. Reference them generically in docs (e.g. "in password manager") without naming the manager or entry title. Real values live in a gitignored `.env`. Git history is permanent, so the fix for a leaked secret is always to **rotate** it, not just delete the line.
   - **Local network details are fine to commit and should be** — private-range IPs (192.168.x.x), MACs, hostnames. They're meaningless outside the LAN and make the docs actually usable. Don't over-redact these.
   - Default posture: specific where it's generic/local, strict where it's a credential. Ask when unsure.
8. **Long-running jobs go in tmux, always** — stderr captured to a file, output paths on persistent storage (not `/tmp`) if they need to survive a reboot.
9. **Two identical WD Red 4TB drives exist** — resolve by serial before any destructive drive operation (see `parts-inventory.md`).

## Frigate design decisions

1. **Detector = ONNX, not the old "TensorRT detector" type.** As of Frigate 0.16+, the standalone `tensorrt` detector type was removed for desktop Nvidia GPUs (it now only exists for Jetson via the `-tensorrt-jp6` image). Current path: run the `-tensorrt` image (ships the TensorRT libraries) with a `type: onnx` detector — on an Nvidia GPU, ONNX auto-detects and uses TensorRT for acceleration.
2. **Model = YOLOv9-s @ 320, exported manually.** Nvidia/ONNX doesn't auto-download YOLO models (auto-download only exists for Rockchip/RKNN and MemryX). Export ONNX via a one-time docker build, bind-mount into `/config/model_cache/`; Frigate compiles it into a GPU-specific `.trt` engine on first start (several minutes, one-time; re-runs if the GPU or driver changes). 320×320 is the recommended default since Frigate crops to motion regions before detection — bump to 640 later only if far-field detection needs it.

   Export command (run in tmux, pulls weights + builds):
   ```
   cd /opt/stacks/frigate/config/model_cache
   docker build . --build-arg MODEL_SIZE=s --build-arg IMG_SIZE=320 --output . -f- <<'EOF'
   FROM python:3.11 AS build
   RUN apt-get update && apt-get install --no-install-recommends -y cmake libgl1 && rm -rf /var/lib/apt/lists/*
   COPY --from=ghcr.io/astral-sh/uv:0.10.4 /uv /bin/
   WORKDIR /yolov9
   ADD https://github.com/WongKinYiu/yolov9.git .
   RUN uv pip install --system -r requirements.txt
   RUN uv pip install --system onnx==1.18.0 onnxruntime onnx-simplifier==0.4.* onnxscript
   ARG MODEL_SIZE
   ARG IMG_SIZE
   ADD https://github.com/WongKinYiu/yolov9/releases/download/v0.1/yolov9-${MODEL_SIZE}-converted.pt yolov9-${MODEL_SIZE}.pt
   RUN sed -i "s/ckpt = torch.load(attempt_download(w), map_location='cpu')/ckpt = torch.load(attempt_download(w), map_location='cpu', weights_only=False)/g" models/experimental.py
   RUN python3 export.py --weights ./yolov9-${MODEL_SIZE}.pt --imgsz ${IMG_SIZE} --simplify --include onnx
   FROM scratch
   ARG MODEL_SIZE
   ARG IMG_SIZE
   COPY --from=build /yolov9/yolov9-${MODEL_SIZE}.onnx /yolov9-${MODEL_SIZE}-${IMG_SIZE}.onnx
   EOF
   ```
3. **go2rtc restream in front of every camera.** Frigate connects to `127.0.0.1:8554/<stream>`, never the camera directly — one connection per camera means less camera CPU/bandwidth and stable RTSP. Reolinks in particular drop direct RTSP connections; go2rtc fixes that.
4. **Dual-stream per camera:** main (high-res) → `record` role, sub (low-res) → `detect` role. Never run detection on a 4K main stream — wasteful and slower for no accuracy benefit (Frigate crops to motion regions anyway).
5. **GPU is shared with Plex.** Both Plex (NVENC transcode) and Frigate (NVDEC decode + inference) run on the one RTX 2070. Fine for now — the YOLOv9-s engine is small and one camera's decode is trivial. Watch VRAM (8GB) as cameras are added; a future platform swap (265K Quick Sync/iGPU) is the pressure-release valve if it becomes a problem.
6. **Storage split:** config + `frigate.db` + `model_cache` live on the OS SSD under `/opt/stacks/frigate/config` (fast, small). Recordings/snapshots go to the WD Black 2TB at `/mnt/frigate` → `/media/frigate` in-container. Recordings are stream-copied (no re-encode), so retention is a disk-space question, not a CPU one.
7. **Secrets:** MQTT + RTSP passwords live in `.env` (gitignored, `chmod 600`), referenced in config as `{FRIGATE_MQTT_PASSWORD}` / `{FRIGATE_RTSP_PASSWORD}`. Real values in the password manager. Consistent with Hard Rule 7.
8. **Auth / ports:** `8971` is the authenticated UI + API — this is what the HA integration points at. `5000` is the internal *unauthenticated* UI, exposed for LAN convenience/first-boot troubleshooting only; drop it once things are stable (tracked in `todo.md`). No external exposure — remote access is future-Tailscale, same plan as Plex.
9. **shm sizing.** Per-camera minimum for the detect pipeline: `(detect_width × detect_height × 1.5 × 20 + 270480) / 1048576` MB, plus ~40MB for logs. One 640×360 camera ≈ 47MB total — the 128MB default would cover it, but `shm_size` is set to **512MB** deliberately for headroom as cameras are added. Recompute and bump when adding several at once. Changing `shm_size` requires `docker compose down && up -d` (not just restart) — it's a container-creation setting, not a runtime one.

## Open decisions

- **Samsung 500GB spare space (~400GB on the OS drive) — still open.** Candidates: Immich originals (`/mnt/photos`), Immich Postgres + ML cache, or leave it alone to keep the OS drive clean. Photos on the OS drive means OS reinstalls need more care, but it's the only fast unallocated storage and the old photo library fit in 500GB. Blocks the Immich phase (`todo.md`).
- **WD Blue (SMR) fate — still open.** SMR + prior SATA errors on this drive make retirement likely; a SMART long test is needed to confirm before deciding. Currently physically pulled from the case.

## Resolved

- **Old Immich data — preserved, not lost.** The Crucial 500GB SATA SSD from the retired ITX Windows PC was never reformatted; its ext4 partition is intact. Planned copy method: WSL2 `wsl --mount` (elevated PowerShell → `wsl --mount \\.\PHYSICALDRIVEn --partition 1 --type ext4`, find `n` via `Get-Disk`), then rsync from WSL2 to server01, `wsl --unmount` after. Check the SSD's old `/mnt/backup/migration` path for a `pg_dump` while mounted — restore it if found, otherwise start a fresh DB. Note: the OneDrive `immich` folder is also being pulled to `/mnt/backup` by the ongoing rclone sync, so cloud originals will be staged locally regardless of this decision — the two sources need reconciling at Immich bring-up time.
- **Plex identity — fresh identity, claimed (2026-07-08).** Old watch history was not migrated.
- **rclone `copy` vs `sync` — `copy`.** A pull-down backup must never propagate a cloud-side deletion (accidental, ransomware, sync glitch) to the local copy. Trade-off accepted: the local copy retains files removed from OneDrive (drift over time), which is the safe direction for a backup to drift in.
- **Plex external/remote access — disabled by design.** Tailscale-based remote access (no port forwarding) is the intended future path; revisit once the SMB/Tailscale work in `todo.md` happens.
