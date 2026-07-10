# Frigate — Camera / NVR Solution (server01)

**Purpose:** Single source of truth for the Frigate NVR stack and all camera/DVR work. Companion to the server01 Master Plan (which keeps only a Phase 5 pointer). Update the camera inventory + session log as cameras and features are added.

**Last updated:** 2026-07-10

---

## Design Decisions (Frigate-specific)

1. **Detector = ONNX, NOT the "TensorRT detector" type.** As of Frigate 0.16+ the standalone `tensorrt` detector type was **removed for desktop Nvidia GPUs** (it now exists only for Jetson via the `-tensorrt-jp6` image). The current path is: run the **`-tensorrt` image** (ships the TensorRT libraries) with a **`type: onnx` detector**. On an Nvidia GPU the ONNX detector auto-detects and uses TensorRT for acceleration. This corrects the Master Plan's original "TensorRT detector" wording.

2. **Model = YOLOv9-s @ 320, exported manually.** For Nvidia/ONNX, Frigate does **not** auto-download YOLO models (auto-download exists only for Rockchip/RKNN and MemryX). We export the ONNX ourselves (one docker build), bind-mount it into `/config/model_cache/`, and Frigate compiles it into a GPU-specific `.trt` engine on first start (several minutes, one-time; re-runs if the GPU or driver changes). 320×320 is the recommended default — Frigate crops to motion regions before detection, so 320 is accurate and cheap. Bump to 640 later only if far-field detection needs it.

3. **go2rtc restream in front of every camera.** Frigate connects to `127.0.0.1:8554/<stream>`, not the camera directly. One connection per camera → less camera CPU, less bandwidth, stable RTSP. Reolinks in particular drop direct RTSP; go2rtc fixes that.

4. **Dual-stream per camera:** main (high-res) → `record` role; sub (low-res) → `detect` role. Never run detection on the 4K main — wasteful and slower.

5. **GPU is shared with Plex.** Both Plex (NVENC transcode) and Frigate (NVDEC decode + inference) use the one RTX 2070. Fine for now — YOLOv9-s engine is small and one camera's decode is trivial. Watch VRAM (8GB) as cameras are added; the future platform swap (265K Quick Sync / iGPU) is the pressure-release valve.

6. **Storage split:** config + `frigate.db` + `model_cache` live on the OS SSD under `/opt/stacks/frigate/config`. Recordings/snapshots go to the **WD Black 2TB at `/mnt/frigate`** → `/media/frigate` in-container. Recordings are stream-copied (no re-encode), so retention is a disk-space question, not a CPU one.

7. **Secrets:** MQTT + RTSP passwords in `.env` (gitignored, `chmod 600`), referenced in config as `{FRIGATE_MQTT_PASSWORD}` / `{FRIGATE_RTSP_PASSWORD}`. Real values in Proton Pass. Consistent with Master Plan Hard Rule 9.

8. **Auth / ports:** `8971` is the authenticated UI + API (this is what the HA integration points at). `5000` is the internal **unauthenticated** UI — exposed here for LAN convenience/first-boot troubleshooting only; drop it once things are stable. No external exposure; remote access is future-Tailscale, same as Plex.

---

## Hardware / Camera Inventory

Learning setup, indoor-only for now — a couple of simple Reolink cameras plus a doorbell coming soon. No fixed room assignments yet, so cameras are named by hostname, not location (see Session Log 2026-07-10).

| Camera | Model | Hostname | IP | Streams | Frigate name | Status |
|---|---|---|---|---|---|---|
| E1 Zoom | Reolink E1 Zoom (PT) | reolink-e1-zoom | 192.168.8.106 (ethernet) | main HEVC 3840×2160 + AAC (`h264Preview_01_main`); sub H.264 640×360 + AAC (`h264Preview_01_sub`) — verified via ffprobe 2026-07-09; login `admin` | `reolink_e1_zoom_eth` | previously named `living_room`; streams previously verified but config still needs updating to new name |
| E1 Pro | Reolink E1 Pro | reolink-e1-pro | 192.168.8.103 (ethernet) | TBD — not yet probed; don't assume it matches the Zoom's paths | `reolink_e1_pro_eth` | connected, streams not yet verified |
| Doorbell | Reolink doorbell | TBD | TBD | TBD | TBD | owned, hooking up soon |

**Detector host:** server01, RTX 2070 (driver 580-server, CUDA 13.0, Container Toolkit 1.19.1).
**MQTT broker:** Mosquitto on HA Green `192.168.8.2:1883`, service user `frigate` (defined in the Mosquitto add-on config, not an HA user account).

---

## Stack Files

- `/opt/stacks/frigate/compose.yaml` — the stack (image, GPU passthrough, volumes, ports, shm).
- `/opt/stacks/frigate/config/config.yml` — Frigate config (detector, model, go2rtc, cameras).
- `/opt/stacks/frigate/.env` — secrets (gitignored). Template: `.env.example`.
- `/opt/stacks/frigate/config/model_cache/yolov9-s-320.onnx` — exported model (see below). The `.trt` engine Frigate builds lands alongside it.

---

## Build Runbook (first bring-up)

Run on server01. One command at a time, review output before the next.

**0. MQTT user** — already exists in the Mosquitto add-on config (`frigate`), password in Proton Pass. Nothing to do unless the password was lost.

**1. Create the stack dir + drop the files in**
```
mkdir -p /opt/stacks/frigate/config/model_cache
```
Place `compose.yaml` in `/opt/stacks/frigate/`, `config.yml` in `/opt/stacks/frigate/config/`.
Create `.env` from `.env.example`, fill the two real passwords, then:
```
chmod 600 /opt/stacks/frigate/.env
```

**2. Export the YOLOv9-s ONNX model** (in tmux — pulls weights + builds, a few minutes)
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
Confirm `yolov9-s-320.onnx` now exists in `model_cache/`. (Matches the `model.path` in config.)

**3. First start** (in tmux — first boot compiles the .trt engine, several minutes)
```
cd /opt/stacks/frigate
docker compose up -d
docker compose logs -f
```
Watch for, in order:
- ONNX → TensorRT engine build ("... building engine ..." then load success
- go2rtc streams connecting to the Reolink
- detector process starting on the ONNX/GPU (not CPU)
- MQTT connected
- **the auto-generated admin password** for the `8971` UI is printed once in the logs — grab it and store in Proton Pass, then log in and set your own.

**4. Verify**
- UI at `http://192.168.8.8:8971` — live view of each configured camera.
- Walk in frame → a `person` event fires.
- System Metrics page → detector inference time (expect ~10–20 ms on the 2070) and that it's the GPU, not CPU.

**5. Re-point the HA integration** (daytime is fine)
- HA → Frigate integration → set URL to `http://192.168.8.8:8971`.
- Confirm entities repopulate and the doorbell/snapshot automations still resolve.

---

## shm sizing

Per-camera minimum for the detect pipeline:
```
(detect_width x detect_height x 1.5 x 20 + 270480) / 1048576  MB
```
Plus ~40 MB for logs. One 640×360 camera ≈ 47 MB total — the 128 MB default would cover it. We set **512 MB** deliberately for headroom as cameras are added; recompute and bump when you add several. Changing `shm_size` requires `docker compose down && up -d` (not just restart) — it's a container-creation setting.

---

## Gotchas / Notes (running list)

- **Explicit detect resolution required.** 0.17 changed camera-resolution auto-detection; some cameras make Frigate hang on startup if `detect.width/height` aren't set. We set them explicitly (640×360).
- **Model path must match the export.** If `model.path` and the actual `.onnx` filename disagree, Frigate errors at start. Re-exporting a different size/imgsz means updating both.
- **Re-build the engine on GPU/driver change.** The `.trt` engine is tuned to this exact GPU + CUDA/TensorRT. After a driver bump or the future GPU swap, delete the cached engine and let it rebuild.
- **H.265 live view.** The main stream is HEVC; browser WebRTC live view of H.265 can be flaky. Detection uses the H.264 sub, so detection is unaffected. If live view of main misbehaves, that's why — not a detection problem.
- **Reolink stream-name lies about the codec.** Both streams use the `h264Preview_` prefix — but the *main* actually encodes **HEVC/H.265** 4K (verified via ffprobe). There is no `h265Preview_` path on this camera; the `h264Preview_01_main` string carries H.265. Don't trust the path name for codec; probe it. This was the #1 near-miss on bring-up.
- **Both streams carry AAC audio.** Recording captures audio by default. Some US states are two-party-consent for audio recording — decide per camera whether to keep it (add `-an` handling / disable audio on the record role) before this becomes a legal question. Living-room main + sub both have audio.
- **RTSP creds / URL parsing.** A `401 Unauthorized` on ffprobe is often not a real auth failure — a stray space after `admin:` or unescaped special characters in the password break URL parsing and surface as 401. Percent-encode symbols in the password.
- **Login is `admin`.** Using the camera admin account directly (dedicated least-privilege user is a backlog hardening item — the E1 Zoom only exposes user creation via its web UI at `http://192.168.8.106`, not the mobile app).
- **YAML paste.** Create config with an editor (nano) or a clean ASCII heredoc; smart quotes / non-breaking spaces from rich-text paste corrupt YAML silently.
- **Retention.** Continuous set to 3 days, events (alerts/detections) 30 days, as a starting point. One 4K HEVC continuous stream is roughly 10–20 GB/day — fine on the 2TB Black for a few cameras; tune per camera as the count grows.

---

## Session Log

- 2026-07-09 (evening) — Doc created. Confirmed current Frigate = 0.17.2. Established the ONNX-on-tensorrt-image approach (corrects Master Plan's "TensorRT detector"). Authored `compose.yaml`, `config.yml`, `.env.example` for the first camera (Reolink E1 Zoom, `living_room`). MQTT `frigate` user confirmed present in Mosquitto add-on config. rclone OneDrive initial sync still running (unrelated, from Master Plan) — Frigate bring-up does not depend on it. **Verified E1 Zoom streams via ffprobe:** main = `h264Preview_01_main` (HEVC 4K + AAC), sub = `h264Preview_01_sub` (H.264 640×360 + AAC), login `admin`. Corrected config (main path was assumed `h265Preview`; the h264-prefix path carries HEVC). Next: run the build runbook steps 1–4 on server01. (Note: bring-up was never actually run — corrected 2026-07-10.)
- 2026-07-10 — Plan reset. The build runbook was never actually executed (no `docker compose up` yet — camera 1 bring-up had not started). Cameras have since been physically relocated; dropped location-based naming (`living_room`) in favor of hostname-style Frigate camera names since final placement isn't decided yet. Current plan: a simple indoor learning setup with two Reolink E1 cameras — **E1 Zoom** (`reolink_e1_zoom_eth`, 192.168.8.106, ethernet — same unit/IP as before, just renamed) and **E1 Pro** (`reolink_e1_pro_eth`, 192.168.8.103, ethernet — new, streams not yet probed) — plus a Reolink doorbell to be hooked up soon. The Amcrest floodlight driveway-cam plan is dropped. `figate.md` (typo'd duplicate of this doc from initial creation) removed — no content lost, `frigate.md` was already a strict superset; full history remains at commit `0c51da0`. Next: ffprobe the E1 Pro (192.168.8.103) to get its real stream paths (don't assume they match the Zoom's), then update `config.yml`/`compose.yaml` for both cameras and run the build runbook for the first time.

---

## Backlog

- [ ] Create dedicated least-privilege Reolink user (web UI, per-camera → System → User Management → type User) and swap `admin` out of the config. Hardening — currently using admin creds directly.
- [ ] Decide audio recording policy per camera (two-party-consent states) — currently AAC audio is captured on `reolink_e1_zoom_eth`.
- [ ] ffprobe the E1 Pro (192.168.8.103) to confirm its real RTSP stream paths — do not assume they match the Zoom's.
- [ ] Add `reolink_e1_pro_eth` camera block to `config.yml` once paths are verified — its own go2rtc streams + camera block.
- [ ] Hook up the Reolink doorbell when ready — new go2rtc streams + camera block, plus whatever doorbell-specific config it needs (button-press event, two-way audio, etc.).
- [ ] Per-camera zones + masks once locations are settled — cuts false positives.
- [ ] Decide birdseye view + whether to expose it.
- [ ] Revisit dropping port 5000 once auth + integration are settled.
- [ ] Tune retention per camera once real disk-usage/day is observed.
- [ ] (Optional) Second ONNX detector instance if camera count outpaces one detector.
