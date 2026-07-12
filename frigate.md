claude generated this from memory, so the last version is probably better. I gave it the copy of the current file, changed to fable, and told it to do the same thing, but not from memory. from the last version. then, anthropic throttled me, asking for more money. After I wasted countless tokens on claude code trying to f up the frigate setup. f anthropic. 
# frigate.md

Companion doc for the Frigate NVR stack on server01. Sanitized for a repo that may become
public — IPs, MACs, ports, and device specifics are placeholders. Real values live in the
private inventory / `.env`, not here.

**Status:** Working. Multiple cameras, GPU detection on the discrete Nvidia GPU, recording
to a dedicated data disk.
**Updated:** 2026-07-12.

---

## What's running

- **Frigate**, image pinned to a specific `<version>-tensorrt` tag (deliberate pin — no
  surprise major-version jumps on `docker compose pull`).
- **Detector:** ONNX (`type: onnx`) on the discrete Nvidia GPU. Naming trap: on desktop
  Nvidia the *image* is `-tensorrt` but the *detector* is `onnx`. The standalone TensorRT
  detector was removed in 0.16+. 0.17 adds CUDA Graphs, which benefit the GPU.
- **Model:** YOLOv9-s @ 320×320, self-exported ONNX in `config/model_cache/`, paired with
  `/labelmap/coco-80.txt`.
- **GPU passthrough:** CDI (`devices: - nvidia.com/gpu=all`), matching Docker's advertised
  CDI devices — cleaner than the older `deploy`/`runtime: nvidia` syntax.
- **Recordings:** dedicated disk mounted at `/mnt/frigate` → `/media/frigate` in-container.
- **Retention (0.17 tiered):** continuous 3d, alerts 30d, detections 14d, snapshots 14d.
- **MQTT:** Mosquitto on the HA host, service user `frigate`.
- **UI:** authenticated port `8971`. Admin password auto-generated first boot, stored in
  the password manager (not here).

---

## Stack files

- `compose.yaml` — image, CDI GPU, volumes, ports, shm, tmpfs cache.
- `config/config.yml` — detector, model, cameras, retention. `version: 0.17-0`.
- `.env` — secrets, `chmod 600`, gitignored. Template committed as `env.example`
  (no leading dot — iOS Files app reserves dot-prefixed names).

---

## Per-camera password convention

RTSP passwords are per-camera, named by MODEL/ROLE, referenced in config as
`{FRIGATE_RTSP_PASSWORD_<NAME>}`. Real values only in `.env`.

```
FRIGATE_MQTT_PASSWORD=...
FRIGATE_RTSP_PASSWORD_<CAM_A>=...
FRIGATE_RTSP_PASSWORD_<CAM_B>=...
```

**Password gotcha (cost real time):** reserved characters (`@ : / # ? & %`) in an RTSP
password break URL parsing ("Port missing in uri" from ffprobe). Fix: set all camera
passwords **alphanumeric only, 16+ chars**. Do this for every camera.

---

## Model build (once, or if the .onnx is deleted)

`YOLO_MODELS` env var does NOT build ONNX models — it's Jetson-only (`.trt`). The ONNX
detector needs a model you build yourself; there is no default. Run in tmux from the
model_cache dir; outputs `yolov9-s-320.onnx` to the current directory. Verbatim from
Frigate docs (object_detectors → "YOLOv9 for other detectors").

```
cd config/model_cache
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

---

## Adding a camera (repeatable method)

1. Set an alphanumeric RTSP password in the camera app. Reserve its IP on the DHCP server
   (MAC-bound). **Reboot/renew the camera** so it actually moves to the reserved IP — a
   reservation does not force a currently-leased device off its old address (this produced
   a "No route to host" until the camera was power-cycled).
2. Confirm RTSP is enabled in the camera app. ONVIF is only needed for PTZ autotracking or
   doorbell button-press events — not for plain streaming.
3. Probe both streams with a per-camera shell variable (keeps the password out of history):
   ```
   read -rs RTSP_PW          # Enter once, paste password, Enter once
   echo ${#RTSP_PW}          # sanity-check length
   ffprobe -v error -rtsp_transport tcp -i "rtsp://admin:${RTSP_PW}@<CAM_IP>:554/<PATH>" \
     -show_entries stream=codec_name,width,height,avg_frame_rate -of default=noprint_wrappers=1
   ```
   **Never assume paths or codecs — each model differs.** Reolink path names can lie (an
   `h264`-named path may carry HEVC). Detect resolutions vary per model, so every detect
   block gets its own probed dimensions; never copy blocks between cameras.
4. Add the password to `.env` (own variable). Add a camera block to `config.yml` referencing
   that variable, with the probed record/detect dimensions.
5. `cp config.yml config.yml.bak` first, then `docker compose up -d`, watch logs, verify UI.

---

## Bring-up / restart & health check

```
docker compose up -d
docker compose logs -f     # Ctrl+C stops the follow only, not the container
docker compose ps          # want: Up ... (healthy)
docker compose logs 2>&1 | grep -iE 'No frames|failed|error|watchdog' | grep -ivE '502|api/version|/ws' | tail -20
```

Healthy signal: `ONNX: .../yolov9-s-320.onnx loaded` with NO traceback after, camera
processor + capture process started for each camera, and NO `watchdog: Detection appears to
have stopped`. Empty filtered-log output = no camera problems. The `502 /api/version` and
`/ws` lines at boot are nginx answering before the backend is up — normal noise.

---

## ONVIF notes

RTSP is all Frigate needs for video on every camera. ONVIF is only consumed by:
- PTZ cameras — enables autotracking (needs separate config)
- Doorbells — enables button-press events (needs separate config)
Fixed non-PTZ cameras don't need it; Frigate does its own motion detection on the stream.

---

## Backlog (optional)

- Doorbell button-press events (ONVIF path; verify against current docs — model-specific).
- Re-point the HA Frigate integration to pick up all cameras.
- Amcrest floodlight uses Dahua RTSP paths (`cam/realmonitor?channel=1&subtype=0/1`),
  not Reolink paths.
- Watch host RAM as cameras are added (2K+ record streams add up).
- PTZ autotracking on the PTZ camera, if wanted.
