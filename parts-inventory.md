# Parts & Hardware Inventory

Owned hardware and planned/researching items. **No purchase records here, ever** — no prices paid, no order numbers, no receipts. Status and location only. When something moves from "planned" to "owned," update its entry in place rather than duplicating it.

---

## server01 (24/7 home server)

**CPU/board:** Ryzen 7 2700X on MSI X470 (stable BIOS flashed). Subjectively snappier than the retired ITX/265K build it replaced — not raw silicon (265K is faster on paper) but the plumbing: media on direct SATA instead of a USB-C enclosure, Docker root on Gen4 NVMe, fresh minimal install, less service contention.

**GPU:** RTX 2070 — driver 580-server (580.159.03), CUDA 13.0, Container Toolkit 1.19.1. Shared between Plex (NVENC transcode, HW-confirmed live) and Frigate (NVDEC decode + ONNX/TensorRT inference). 8GB VRAM — watch usage as camera count grows.

**Network:** `eno1` (MAC `30:9c:23:d5:68:f8`) → `192.168.8.8`, ethernet, primary, reserved on the Brume 3 router. `192.168.8.7` = WiFi, disabled, break-glass only. DHCP + MAC-match via netplan.

**UPS:** CyberPower CP1500PFCLCD. USB data cable will run to the RPi5 (as NUT server; server01 will be the netclient) — cable not yet connected, low priority.

### Drives

| Drive | by-id | Role | Status |
|---|---|---|---|
| Samsung 970 EVO 500GB NVMe | `nvme-Samsung_SSD_970_EVO_500GB_S466NX0KA14327N` | Ubuntu OS. ~400GB spare, allocation undecided | OS installed |
| Inland 1TB NVMe | `nvme-Inland_QN450_NVMe_SSD_IB26AC1000P03985` | `/var/lib/docker` (ext4 "docker") | Live, boot-verified ×2 |
| WD Red 4TB | `ata-WDC_WD40EFZX-68AWUN0_WD-WX32D124UYHS` | `/mnt/backup` — rclone OneDrive target (ext4 "backup") | Live, fstab by-UUID+nofail, boot-verified |
| WD Red 4TB | `ata-WDC_WD40EFZX-68AWUN0_WD-WX62D12CCFRX` | `/mnt/media` — Plex media (ext4) | Live under Plex, holds verified library |
| WD Black 2TB | `ata-WDC_WD2003FZEX-00SRLA0_WD-WMC6N0P1DV7X` | `/mnt/frigate` — Frigate recordings (ext4 "frigate") | Live, boot-verified ×2 |
| WD Blue 4TB (SMR) | — | Unassigned | Physically pulled from the case — prior SATA errors, retirement candidate pending a SMART long test |

**Two of the WD Reds are visually identical** (same model string, different serial) — always resolve by serial before any destructive operation. See `maintenance.md` → Gotchas.

## Cameras (Frigate)

| Camera | Model | Hostname | IP | Status |
|---|---|---|---|---|
| E1 Zoom | Reolink E1 Zoom (PT) | `reolink-e1-zoom` | 192.168.8.106 (ethernet) | Owned, streams ffprobe-verified, config pending rename |
| E1 Pro | Reolink E1 Pro | `reolink-e1-pro` | 192.168.8.103 (ethernet) | Owned, connected, streams not yet probed |
| Doorbell | Reolink doorbell | TBD | TBD | Owned, not yet hooked up |

Indoor-only learning setup for now — no fixed room assignments yet, so cameras are named by hostname rather than location.

## Other owned devices

- **RPi5** — separate box, planned role: Prometheus/Grafana + NUT server for server01's UPS. Out of scope for server01 detail except its exporters/clients running on server01 (see `todo.md` → Monitoring & UPS).
- **MikroTik CSS610** — network switch. Config backup still outstanding (see `todo.md` → Monitoring & UPS).
- **ITX PC (desktop)** — Windows 11, rebuild complete and back in daily use. Drives: Samsung 970 EVO 300GB NVMe (OS), Crucial 500GB SATA SSD (confirmed empty — the old Immich library is not on it, see `decisions.md`), TeamGroup AX2 1TB SATA SSD (new, replacing the removed WD 2.5" HDD), and its original 128GB NVMe retained in place (pulling the board to swap in the spare 512GB was more effort than it was worth) — earmarked for video editing scratch once a tool is chosen. Noctua cooler retained. External drive enclosure no longer used — internal drives only. Also now handles CD-ripping duties, previously the Eversolo's job. Powered 7-port Pluggable USB-A hub added (USB-A into the ITX), feeding the SMSL DDC for desktop audio — see Desktop audio setup below.
- **Eversolo DMP-A6 Gen 2** — network media streamer/DAC (see `maintenance.md` → Gotchas for its SMB behavior). Moved off the desk into the main audio setup (see below), no longer paired with the ITX. Internal NVMe pulled entirely — see TeamGroup T-Force G50 note below.
- **TeamGroup T-Force G50 512GB NVMe** — pulled from the Eversolo, now in an external enclosure as a general file-transfer/scratch drive. Not part of any build.
- **Main audio setup** — Eversolo DMP-A6 Gen 2 (DAC/streamer); Waxwing phono preamp (optical into the Eversolo); SMSL CD player (coaxial into the Eversolo); Rega Exon 3s (amps); Dynaudio Audience 60 towers. Everything else previously in rotation is on the shelf, pending a decision on what to sell.
- **Desktop audio setup (ITX)** — KEF LSX LT II speakers; RSL 10" Speedwoofer subwoofer (the higher-end model); SMSL DDC feeding the KEFs via optical, fed in turn by the Pluggable USB-A hub off the ITX; a work dock also feeds the DDC over USB; subwoofer cable from Blue Jeans Cables.

## Live Services (server01)

| Service | URL | Stack | Status |
|---|---|---|---|
| Mealie | http://192.168.8.8:9925 | `/opt/stacks/mealie/` — mealie v3.20.1 + postgres:17, bind mounts, `ALLOW_SIGNUP=false` | Live |
| Plex | http://192.168.8.8:32400/web | `/opt/stacks/plex/` — lscr.io/linuxserver/plex, host net, NVENC pinned to RTX 2070 (HW-confirmed), `/mnt/media` ro, tmpfs `/transcode` | Live, fresh identity claimed. External access OFF by design (see `decisions.md`) |
| Frigate | http://192.168.8.8:8971 (once up) | `/opt/stacks/frigate/` — `0.17.2-tensorrt` image, ONNX detector, go2rtc restream, recordings → `/mnt/frigate` | Bring-up pending — see `todo.md` → Frigate |
| rclone (host service, not a container) | — | Remote `onedrive` → `/mnt/backup`, config at `~/.config/rclone/rclone.conf` (never committed — holds the refresh token) | Initial `copy` sync in progress; systemd timer to follow — see `todo.md` → Backup |

## Planned / researching

- **Future CPU/GPU platform swap** — candidate: Intel 265K for Quick Sync (iGPU), as the pressure-release valve once the RTX 2070 is under real VRAM pressure from Plex+Frigate sharing it. Not decided, not urgent.
- **Samba** — to be deployed as a Docker container on server01 for the video-ingest SMB share (see `todo.md` → Networking / File Sharing). No specific hardware need, just the container.
