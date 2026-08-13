# Home Lab Finalization Plan

Status: active
Created: 2026-08-11
Target repo: HA repo (technical/config material — not the `home` repo)

---

## Purpose

Close out the partially-built state of the lab. The goal is not "more projects" — it is that every service already started is fully configured, fully functional, and actually being used.

## Definition of done

- All Dockerized services have their files fully migrated to final locations
- Backups exist, run on a schedule, and have been restore-tested
- Frigate delivers a family-usable experience, not just a working backend
- Monitoring dashboards running from the RPi5
- No half-configured services left running

## Fixed decisions (do not reopen)

| Decision | Resolution |
|---|---|
| server01 platform | Ryzen 2700X box is **final**. No Jonsbo/Z890 swap. RTX 2070 stays. |
| Camera stack | **Frigate**. Reolink NVR ruled out. |
| Buying break | Off **for blockers only** — purchase when something is genuinely gating work. |
| Storage capacity | All drives already installed in server01. No purchase needed to begin. |
| Repo | Current and authoritative. No documentation catch-up pass required. |
| Z-Wave dongle | Owned, not plugged in. Not cut — deferred until a need appears. |

## Constraints

- ~1–2 hours per evening
- Family-facing access must be zero-touch — no VPN client apps on other people's phones
- Hard rules still apply: everything in Docker, drives by `/dev/disk/by-id`, one command at a time, tmux for long runs, source drives never written before verification

---

## Phase 0 — Ground truth

Nothing gets built on assumptions. One or two evenings.

- [x] Verify whether the rclone OneDrive job actually exists on this box — confirmed live: `rclone-onedrive.timer` enabled, firing nightly ~03:00, every run for the past 7+ days exited `status=0/SUCCESS` (15–21 min each). See `maintenance.md` Log 2026-08-11.
- [x] Full drive inventory: device, by-id path, capacity, free space, filesystem, current mountpoint — done 2026-08-12, cross-checked against `parts-inventory.md` (already accurate, no drift). See `df -h` output in `maintenance.md` 2026-08-11 log and `lsblk`/by-id listing in the 2026-08-12 log.
- [ ] SMART check every drive before trusting anything to it — long self-tests started 2026-08-12 on the OS SSD, Docker NVMe, and the three spinning drives; results pending. WD Blue not tested (still physically pulled from the case).
- [ ] Inventory where music files currently live (Eversolo NVMe, PC, any other device)
- [ ] Write the target storage map into the repo before moving a single file

**Done when:** the storage map exists on paper and every drive's health is known.

---

## Phase 1 — Storage layout and backup

Highest risk item in the lab. No backups exist anywhere today, and Phase 2 will pile five categories of data onto a machine with no safety net. This comes first even though it isn't the most annoying thing.

**Classify data first:**

- *Irreplaceable* — photos, ripped CDs, documents. Must be backed up off-box and off-site.
- *Replaceable* — Plex media, Frigate recordings. Re-acquirable. Not worth paying to back up.

Tasks:

- [ ] Finalize mountpoints and fstab entries (by-id, `nofail`)
- [ ] Stand up or repair rclone → OneDrive for irreplaceable data only; copy-not-sync
- [ ] systemd oneshot + nightly timer; confirm it fires
- [x] **Restore test** — done 2026-08-11 via a read-only Samba share of `/mnt/backup` mapped from the Windows PC; opened several real files directly from the backup copy and confirmed they're intact.
- [ ] Immich: install the MX500 (bracket purchase = approved blocker), mount at `/mnt/photos`, then start the container, then restore the pg_dump from `/mnt/backup/migration`. In that order.

**Done when:** a nightly job runs unattended, failure is visible, and a restore has been performed successfully at least once.

---

## Phase 2 — File migration

- [ ] **Music** — canonical path decided: `/mnt/media/music` (already existed with 78GB from the original bring-up rsync, same drive Plex already uses). Now reachable read-write over the network via the `media` Samba share (see `decisions.md`). In progress: pulling the Eversolo DMP-A6 Gen 2's rips over via the PC as middleman (blocked as of 2026-08-12 on the A6's SMB login from Windows — see `maintenance.md` Gotchas → Eversolo). Remaining after that: consolidate every other scattered copy (PC, etc.), dedupe, confirm Plex/Plexamp already points at this exact folder.
- [ ] **Plex media** — move to final location, update library paths, verify transcoding still pins the RTX 2070
- [ ] **Documents/backups** — consolidate to a single tree
- [ ] **Frigate recordings** — define a dedicated path with an explicit retention policy (see Phase 3)

Verification before deleting any source: checksum manifests, exit code checked, source drives untouched until verification passes.

**Done when:** every category has exactly one authoritative location and no orphan copies remain on other devices.

---

## Phase 3 — Frigate family layer

The backend works. Cameras are live and viewable via the Reolink app. The gap is the experience lost when leaving Nest.

**Target parity list:**

1. Push alerts with a snapshot
2. Easy remote viewing away from home
3. Clip history and scrubbing back
4. Wife's phone works with zero setup

Two-way talk is explicitly not required.

**Open decision — remote access.** See "Remote access options" below. This gates items 2 and 4 and should be settled before the rest of this phase.

Tasks:

- [ ] Settle remote access approach
- [ ] Configure `record` + retention (separate retention for continuous vs. detection events)
- [ ] Detection tuning: motion masks, zones, object filters, thresholds — reduce false alerts before wiring notifications, or the notifications train everyone to ignore them
- [ ] HA notification automation with snapshot attachment; consider actionable notifications
- [ ] Build a single simple HA dashboard for family use — camera cards plus recent events, nothing else
- [ ] Physically mount the cameras (doorbell off the bench; E1 Zoom and E1 Pro to final positions)
- [ ] Only after the above is stable: retire the Reolink app for family use

**Done when:** an alert arrives with a snapshot, the clip can be opened from that alert away from home, and no setup steps were required on the recipient's phone.

---

## Phase 4 — Monitoring on the RPi5

Full dashboards, not just uptime alerts.

- [ ] Move RPi5 into Rack 2
- [ ] Stack on RPi5: Prometheus + Grafana (Docker)
- [ ] Exporters on server01: node_exporter, cAdvisor, smartctl_exporter
- [ ] Dashboards: host health, disk usage + SMART, container status, Plex sessions, Frigate detection/GPU stats
- [ ] Alerts that actually matter: backup job failed, disk above threshold, SMART pre-fail, container down

**Done when:** the dashboard is something worth opening, and a failed backup notifies without being looked for.

---

## Phase 5 — Power protection

- [ ] Source a second UPS for server01's new location (cheap/used is fine — it only needs to hold long enough for a clean shutdown)
- [ ] Confirm the unit has a USB data port and is NUT-compatible before buying
- [ ] Configure NUT for graceful shutdown; add the HA NUT integration for visibility

**Done when:** server01 shuts down cleanly on battery instead of losing power mid-write.

---

## Phase 6 — Long tail

Work through only after Phases 0–5 are closed. Roughly in value order.

**Home Assistant**

- [ ] Aqara W200 thermostat hub — HA integration (hardware is installed and working)
- [ ] Kitchen Hue dimmer migration to Z2M
- [ ] Nursery Hue Bloom re-pairing (hard power-cycle, re-interview within 5–10s, rename immediately)
- [ ] Basement dimmer project — four Kasa targets, Hue button cycles active set
- [ ] Bedroom Hue dimmer color layer
- [ ] Script/automation naming cleanup to the `area_subsystem_detail` standard
- [ ] Remaining vibration sensors: washer next; garage and sump still one sensor short
- [ ] AirLab WiFi join + MQTT path to Mosquitto
- [ ] Deck stair lighting — Zigbee smart transformer swap

**Network**

- [ ] Test the `brume-backup` Samba share from Windows
- [ ] Save Brume 3 config backup off-device
- [ ] Capture JBL Authentics 300 MAC, add IP reservation
- [ ] Tailscale + SMB validation for admin access

**ITX family PC**

- [ ] Display/TV session: resolution and refresh strategy, HDMI Deep Colour per input, Game Mode, VRR, HDR
- [ ] BIOS fan curve tuning; consider capping 265K power limits
- [ ] Media cabinet ventilation

**OTA TV**

- [ ] Run the KMGH RF7 confirmatory test — laptop direct to HDHomeRun, unplug the Cisco switch, watch the RF7 meter
- [ ] If confirmed: relocate HDHomeRun on quad-shield RG6 away from the Ethernet bundle, add ferrite chokes

**Outdoor hub** — blocked on the slope/erosion/regrading project. Not scheduled here.

---

## Remote access options

Needed for Frigate parity items 2 and 4. Three viable paths.

**Home Assistant Cloud (Nabu Casa)** — ~$6.50–8.50/mo
Zero configuration. Wife installs the HA app, signs in, done. Push notifications with image attachments work out of the box, which directly delivers the top parity item. Fits the stated preference for polished, certified solutions over DIY, and costs no ongoing maintenance time. Downsides: the only recurring subscription in an otherwise self-hosted lab, traffic relays through their infrastructure, and live video over the relay can be slower than local.

**Cloudflare Tunnel** — free, plus ~$10/yr for a domain
Real hostname with TLS, no ports opened. Can expose both HA and the Frigate UI. Downsides: needs Cloudflare Access or equivalent in front or it is public to the internet, which is more setup and more to maintain. Proxying camera video through the free tier is a gray area against their terms. Zero-touch for the family only after the auth layer is configured correctly.

**Tailscale** — already ruled out for family devices by the zero-touch constraint. Still the right answer for admin access.

**Important technical note:** Frigate runs in Docker on server01, not as an HA add-on on HA Green. That means Nabu Casa does **not** expose the Frigate web UI — remote access through it covers HA cards, the media browser, and notifications only. The Frigate UI itself is the best scrubbing experience, so the likely shape is a split:

- **Family** → Nabu Casa, HA app, notifications and simple camera cards
- **Admin** → Tailscale, full Frigate UI

Honest caveat on parity item 3: none of these fully reproduce the Nest timeline. Frigate's event browser is good, but clip review inside the HA app is clunkier than what was lost.

---

## Sequencing rationale

Backups and storage come before the annoyances. Frigate and music are what irritate daily, but a box holding photos, rips, and documents with no backup is the thing that can cause permanent loss. Phase 1 is short. It goes first.
