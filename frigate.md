# frigate.md

Companion doc for the Frigate NVR stack on server01. Phase 5 — first camera live.

**Status:** Working. One camera (E1 Zoom), GPU detection, recording to `/mnt/frigate`.
**Last updated:** 2026-07-11 (evening) — first camera bring-up completed.

---

## What's running

- **Frigate 0.17.2**, image pinned to `ghcr.io/blakeblackshear/frigate:0.17.2-tensorrt`
  (deliberate pin — no surprise major-version jumps on `docker compose pull`).
- **Detector:** ONNX (`type: onnx`) on the RTX 2070. Note the naming trap: on desktop
  Nvidia the *image* is `-tensorrt` but the *detector* is `onnx`. The standalone TensorRT
  detector was removed in 0.16+. 0.17 adds CUDA Graphs, which benefit the 2070.
- **Model:** YOLOv9-s @ 320×320, self-exported ONNX at
  `/opt/stacks/frigate/config/model_cache/yolov9-s-320.onnx` (28 MB), paired with
  `/labelmap/coco-80.txt`.
- **Camera:** `reolink_e1_zoom` @ 192.168.8.106
  - record: `h264Preview_01_main` — HEVC 3840×2160 @ 20fps, AAC (path name lies; it's H.265)
  - detect: `h264Preview_01_sub` — H.264 640×360 @ 10fps (Frigate detect fps set to 5)
  - direct RTSP, no go2rtc restream (E1 Zoom HEVC/go2rtc incompatibility)
- **Recordings:** `/mnt/frigate` (WD Black 2TB, sdc1) → `/media/frigate` in-container.
- **Retention (0.17 tiered):** continuous 3d, alerts 30d, detections 14d, snapshots 14d.
- **MQTT:** Mosquitto on HA Green 192.168.8.2:1883, user `frigate`.
- **GPU passthrough:** CDI (`devices: - nvidia.com/gpu=all`), matching Docker 29's
  advertised CDI devices — cleaner than the older `deploy`/`runtime: nvidia` syntax.
- **UI:** http://192.168.8.8:8971 (authenticated). Admin password auto-generated on first
  boot, saved to Proton Pass.

---

## Stack files

- `/opt/stacks/frigate/compose.yaml` — image, CDI GPU, volumes, ports, shm, tmpfs cache.
- `/opt/stacks/frigate/config/config.yml` — detector, model, camera, retention. `version: 0.17-0`.
- `/opt/stacks/frigate/.env` — secrets, `chmod 600`, gitignored. Template committed as
  `env.example` (no leading dot — iOS Files app reserves dot-prefixed names).

---

## Per-camera password convention

RTSP passwords are per-camera, named by MODEL (matches Frigate camera name + network
hostname). Named by model rather than location because cameras get moved around; revisit
to role-based names later once the setup is proven.

```
FRIGATE_MQTT_PASSWORD=...
FRIGATE_RTSP_PASSWORD_E1ZOOM=...
#FRIGATE_RTSP_PASSWORD_E1PRO=...      # uncomment when added
#FRIGATE_RTSP_PASSWORD_DOORBELL=...   # uncomment when added
```

Config references each camera's own variable:
`rtsp://admin:{FRIGATE_RTSP_PASSWORD_E1ZOOM}@...`

**Password gotcha (cost us real time tonight):** reserved characters (`@ : / # ? & %`) in
an RTSP password break URL parsing ("Port missing in uri" from ffprobe). Fix was setting
all camera passwords to **alphanumeric only, 16+ chars**. Do this for every camera.

---

## Model build (only needed once, or if the .onnx is deleted)

`YOLO_MODELS` env var does NOT build ONNX models — it's Jetson-only (`.trt`). The ONNX
detector needs a model you build yourself. Run in tmux from the model_cache dir; outputs
`yolov9-s-320.onnx` to the current directory. This is the verbatim command from Frigate docs.

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

---

## Bring-up / restart

```
cd /opt/stacks/frigate
docker compose up -d
docker compose logs -f          # Ctrl+C stops follow only, not the container
docker compose ps               # want: Up ... (healthy)
```

Healthy signal in logs: `ONNX: .../yolov9-s-320.onnx loaded` with NO traceback after,
and NO `frigate.watchdog INFO : Detection appears to have stopped`. The boot-order
`502 /api/version` nginx lines are normal noise.

---

## Next-session pickup — add E1 Pro + doorbell (second pass)

Stack is proven, so both can go in together. For each camera:
1. Install ffprobe if needed (`ffmpeg` package). Set the camera password alphanumeric.
2. Verify streams (don't trust path names):
   `read -rs RTSP_PW` (Enter once, paste password, Enter once), then
   `ffprobe -v error -rtsp_transport tcp -i "rtsp://admin:${RTSP_PW}@<ip>:554/<path>" -show_entries stream=codec_name,width,height,avg_frame_rate -of default=noprint_wrappers=1`
3. Uncomment + fill the `_E1PRO` / `_DOORBELL` line in `.env`.
4. Add a camera block in config.yml referencing that camera's password variable.
5. `docker compose up -d`, verify in UI.

Doorbell caveat: different Reolink model, WiFi — probe its actual codec before deciding
if it needs go2rtc; may want lower detect fps. E1 Pro is Ethernet, treat like the Zoom
but re-probe (different model = different paths/codecs possibly).

Backlog after cameras: Amcrest floodlight (Dahua RTSP, `cam/realmonitor?channel=1&subtype=0/1`,
not Reolink paths). Re-point HA Frigate integration to http://192.168.8.8:8971.
Watch RAM (server has ~15 GiB) as cameras are added.
