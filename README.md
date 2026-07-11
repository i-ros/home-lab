# home-lab

Planning, tracking, and (non-secret) config for my home lab — server01 (Plex, Frigate, Mealie, Immich, rclone backup) plus the surrounding network/Home Assistant setup.

- **`todo.md`** — open action items, grouped by area.
- **`maintenance.md`** — dated history log, known gotchas/traps, and recurring quick-reference commands.
- **`parts-inventory.md`** — owned hardware + planned/researching items. No purchase records, ever.
- **`decisions.md`** — the durable rules and the "why" behind how each service is built.
- **`private/`** — gitignored, local-only. Physical/location detail that shouldn't be on GitHub even in a private repo.

Per-stack config (compose + app config) lives at the repo root or in its own subdirectory, e.g. `compose.yml` / `config.yml` for Frigate. Real secrets are never committed — see `decisions.md` → Hard Rule 7.
