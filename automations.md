# Home Assistant — Automations

Companion doc to the master README. Covers naming conventions, the reusable
patterns behind remote-driven lighting, and a running log of what's been built.
This file is authoritative over any assistant memory.

---

## 1. Naming convention

**One rule: `area_subsystem_detail`, area first.**

Area-first groups every object for a feature together alphabetically in the HA
UI, so a feature's whole trio (helpers + script + automations) sits in one
block instead of scattering across the list.

| Object type        | Pattern                          | Example                                   |
|--------------------|----------------------------------|-------------------------------------------|
| Input helper       | `area_subsystem_facet`           | `backyard_lights_mode`, `backyard_lights_brightness` |
| Script             | `area_subsystem_verb`            | `backyard_lights_apply`                   |
| Automation         | `area_subsystem_event`           | `backyard_lights_sunset_on`, `backyard_lights_midnight_off`, `backyard_lights_remote` |
| Label (cross-area) | `lights_group`                   | `lights_primary`, `lights_nightlights`    |
| Physical device    | `type_color_location` (asset tag)| `remote_blue_backyard_lights`, `button_yellow_livingroom` |

Notes:
- **Devices are the exception** — they keep a stable physical asset tag
  (`type_color_location`) so the label on the hardware matches Z2M. Don't
  rename these to match the area-first rule; the color is how you find the
  right unit in a drawer.
- **Labels are cross-area groupings**, not scoped to one place, so they use
  `lights_group` rather than `area_...`.
- Lowercase, underscores, no spaces. The `states()` comparison in a shared
  script is case- and whitespace-sensitive — a stray capital silently drops
  every branch to the `else`.

### Known deviations (cleanup backlog)
These predate the convention and still work; migrate when convenient:
- `lights_backyard_on`  → `backyard_lights_sunset_on`
- `lights_backyard_off` → `backyard_lights_midnight_off`
- Bedtime scripts (`deep`, `wind_down`, lights-on) lack the `area_subsystem_`
  prefix — candidates for `bedtime_lights_deep` etc.

Renaming an entity ID is a first-class HA operation (entity page → gear →
Entity ID) and preserves state and history. Automations that reference the old
ID must be updated in the same pass.

---

## 2. Patterns

Two variants of "button drives lights." Pick by what the button is choosing.

### Pattern A — helpers + one shared apply script
Use when the button **cycles a state dimension** (mode, brightness) and you want
those dimensions composable and independent.

- One `input_select` helper per dimension (e.g. `_mode`, `_brightness`).
- One `_apply` script reads all helpers and drives the lights: turns the
  correct entities on at the stored brightness, turns the rest off.
- Button actions mutate a helper (`input_select.select_next` wraps
  automatically) then call the apply script.
- The schedule (sunset/off) sets the helpers to a known baseline and calls the
  same apply script — so the schedule and the remote can never disagree about
  state.

Independence falls out for free: change mode, brightness carries over; change
brightness, it applies to whatever mode is active.

**Reference implementation:** backyard (§3).

### Pattern B — labels + per-scene scripts
Use when the button **picks among fixed scenes** rather than cycling a dimension.

- A label per light group (`lights_primary`, `lights_nightlights`).
- One script per scene (`deep`, `wind_down`, on), each targeting label_ids.
- Button actions call the scene script directly.

**Reference implementation:** bedtime blue remote (§3).

### Shared conventions for both
- Remote automation triggers on the MQTT action topic
  (`zigbee2mqtt/<device>/action`), `mode: queued` so rapid presses serialize.
- THIRDREALITY Zigbee Smart Button emits `single` / `double` / `hold`
  (also `release` after hold, but preset-cycling on `hold` is preferred over
  ramp-while-held — a preset cycle can't leave a light stuck mid-ramp).
- Schedules stay as the safety net. An unconditional "all off" on a timer
  catches every manual/remote state so nothing burns till morning.
- Debugging: the script **Trace → first variables node → Changed Variables**
  shows exactly what each template rendered. `mode: unknown` there means the
  helper entity ID doesn't match what the script asked for.

---

## 3. Milestone log

### 2026-07 — Bedtime blue remote (shared living spaces)  ✅
Pattern B. Labels recreated clean as `lights_primary` (shared spaces, excludes
bedrooms) and `lights_nightlights` (kitchen entry, kitchen sink, main-level
landing); old slugs deleted; three bedtime scripts repointed to the new
label_ids.
- **single** → deep (1% orange)
- **double** → wind_down (readable nightlights)
- **hold** → lights on
- Midnight auto-deep. Device: THIRDREALITY Zigbee Smart Button.

### 2026-07-16 — Backyard outdoor remote  ✅
Pattern A. Two exterior Hue bulbs — `light.outdoor_basement` (walkout basement)
and `light.outdoor_deck` (deck off kitchen).
- Helpers: `input_select.backyard_lights_mode` (deck / basement / both),
  `input_select.backyard_lights_brightness` (17 / 50 / 100).
- Script: `backyard_lights_apply` — drives on/off per mode at stored brightness;
  `transition` field defaults to 1s, accepts 300 from the schedule.
- Schedule: `lights_backyard_on` resets helpers to both/17 and calls apply with
  a 300s fade at sunset; `lights_backyard_off` unchanged, midnight off with
  300s fade (catches manual use too).
- Remote (`remote_backyard_lights`, MQTT
  `zigbee2mqtt/remote_blue_backyard_lights/action`):
  - **single** → cycle mode (deck → basement → both → wrap)
  - **double** → all off / restore last mode
  - **hold** → cycle brightness (17 → 50 → 100 → wrap)
- **Root-cause note:** initial deck test lit the basement instead. Trace showed
  `mode: unknown` — the mode helper's entity ID hadn't followed its rename from
  `outdoor_` to `backyard_`. Renaming the helper's entity ID fixed it; script
  was correct. This is what prompted §1.

---

## 4. Open items
- **Naming cleanup** — migrate the deviations in §1 to the convention.
- **Backyard remote LQI** — sits on Z2M's Low LQI list (lives just inside a
  door). Watch as the outdoor Hue bulbs establish routing; revisit only if it
  stays low or misses presses.
