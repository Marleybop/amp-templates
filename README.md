# Project CARS 2 Dedicated Server — AMP template

A CubeCoders **AMP** "Generic" module template for the Project CARS 2 dedicated server.
All gameplay/network settings are configured in the AMP web panel and are written into a
real `server.cfg` every time the server starts.

## Files

| File | Role |
|------|------|
| `project-cars-2-server.kvp` | Master module config (`Meta.*`, `App.*`, `Console.*`). |
| `project-cars-2-serverconfig.json` | **ConfigManifest** — the settings shown on the *Configuration* tab. |
| `project-cars-2-servermetaconfig.json` | **MetaConfigManifest** — tells AMP to generate `server.cfg`. |
| `project-cars-2-serverports.json` | Ports AMP reserves / opens / monitors. |
| `project-cars-2-serverupdates.json` | Install steps (SteamCMD, chmod, `server.cfg` template fetch). |
| `project-cars-2-server-template.cfg` | The `server.cfg` **template** with `{{tokens}}` AMP fills in. |
| `manifest.json` | Repo descriptor so AMP can import this as a template repository. |

## How the config actually reaches the server (the important part)

1. You set values on the **Configuration** tab → AMP stores them.
2. On every **Start**, AMP reads `project-cars-2-servermetaconfig.json`, opens the
   `project-cars-2-server-template.cfg` template, replaces each `{{token}}` with the
   matching setting (`ParamFieldName`), and writes the result to
   `…/project-cars-2-server/413770/server.cfg`.
3. `DedicatedServerCmd.elf` runs with that folder as its working directory and reads
   `server.cfg` from there. **No more "server makes up its own config".**

### Ports (why they now show "Listening")

Port numbers live in **one place**: `project-cars-2-serverports.json` (Refs `GamePort`,
`QueryPort`, `SteamPort`, `HttpApiPort`). Each is bridged into `server.cfg` by a hidden
setting whose `FieldName` is `$<Ref>` (e.g. `$GamePort` → writes `hostPort`). So the
server binds **exactly** the ports AMP reserved and monitors → the Network tab shows them
green. Change a port on the **Network/Ports** tab, not in the config text.

> Port-ref tokens like `{{$GamePort}}` only resolve on the command line / in a setting's
> `DefaultValue` — **never inside a generated config file**. That is why the bridge
> settings exist instead of putting `{{$GamePort}}` directly in the template.

Forward these for players: **UDP 27015, 27016, 8766**. Keep **TCP 9000 (HTTP API)** private.

## Installing the server files

The PC2 dedicated server (Steam app **413770**) requires a Steam account that **owns
Project CARS 2** — it cannot be downloaded anonymously. Two options:

* **SteamCMD (automatic):** in the instance's *Updates* settings enter Steam credentials
  for an account that owns PC2, then press **Update**. (The SteamCMD stage is marked
  `SkipOnFailure` so manual installs aren't blocked by it.)
* **Manual:** drop the server files into `…/project-cars-2-server/413770/` yourself.
  If you are **not** using the GitHub fetch, also copy `project-cars-2-server-template.cfg`
  to `…/project-cars-2-server/AMP_server.cfg` (that path is what `metaconfig.json`
  references). On Linux make sure `DedicatedServerCmd.elf` is executable (`chmod +x`).

> Native Linux binary is `DedicatedServerCmd.elf` and Windows is `DedicatedServerCmd.exe`.
> **No Wine is required** — the previous template's `WINEPREFIX`/`WINEARCH` env vars were
> removed.

## Using it in AMP

* **As a template repo:** AMP → *Configuration* → add repository
  `https://github.com/Marleybop/amp-templates` (the fixed `manifest.json` has
  `"prefix": ""` and `"repotype": "AppTemplates"` so AMP finds the files). Then create a
  new instance of "Project CARS 2 Dedicated Server".
* **Manually:** copy all `project-cars-2-server*` files into the AMP datastore template
  directory.

## Verify after first start

1. Open `…/project-cars-2-server/413770/server.cfg` and confirm it shows **your** values
   (name, ports, race setup) — not stock defaults.
2. Network tab: GamePort / QueryPort / SteamPort show **Listening**.
3. The server appears in the in-game browser at your public IP.

## `flags` bitfield (sessionAttributes Flags)

The *Session Flags* setting is a number; OR these together (default **656106**):

| Value | Flag | | Value | Flag |
|------:|------|-|------:|------|
| 2 | FORCE_IDENTICAL_VEHICLES | | 131072 | FILL_SESSION_WITH_AI |
| 8 | ALLOW_CUSTOM_VEHICLE_SETUP | | 262144 | MECHANICAL_FAILURES |
| 16 | FORCE_REALISTIC_DRIVING_AIDS | | 524288 | AUTO_START_ENGINE |
| 32 | ABS_ALLOWED | | 1048576 | TIMED_RACE |
| 64 | SC_ALLOWED | | 2097152 | GHOST_GRIEFERS |
| 128 | TCS_ALLOWED | | 4194304 | PASSWORD_PROTECTED |
| 256 | FORCE_MANUAL | | 8388608 | ONLINE_REPUTATION_ENABLED |
| 512 | FORCE_SAME_VEHICLE_CLASS | | 134217728 | PIT_SPEED_LIMITER |
| 1024 | FORCE_MULTI_VEHICLE_CLASS | | 1073741824 | ANTI_GRIEFING_COLLISIONS |

Default 656106 = AUTO_START_ENGINE + FILL_SESSION_WITH_AI + FORCE_SAME_VEHICLE_CLASS +
TCS_ALLOWED + SC_ALLOWED + ABS_ALLOWED + ALLOW_CUSTOM_VEHICLE_SETUP + FORCE_IDENTICAL_VEHICLES.

Track/vehicle/weather IDs: run with the HTTP API on and open
`http://<ip>:9000/api/list/tracks` (also `/vehicles`, `/vehicle_classes`,
`/enums/weather`).

## Optional addons

`sms_stats` is loaded by default. To add **track rotation** (`sms_rotate`) or a **MOTD**
(`sms_motd`), add them via the *Additional server.cfg lines* box, e.g.:

```
luaApiAddons : [ "sms_base", "sms_stats", "sms_rotate", "sms_motd" ]
```

…then edit `lua_config/sms_rotate_config.json` / `sms_motd_config.json` in the file
manager. Note: when `sms_rotate` controls the setup, do **not** also hand-set the
`ServerControls*` options — `sms_rotate` manages them.

## Tuning the "ready" indicator (optional)

`ApplicationReadyMode` is `Immediate`, so AMP marks the server *Running* as soon as it
launches (port monitoring still updates independently). PC2 has no officially documented
"ready" log line. If you want AMP to wait for true readiness, watch the console on first
start, copy the line printed once it is up, and set in the `.kvp`:
`App.ApplicationReadyMode=RegexMatch` and `Console.AppReadyRegex=^<that line>$`.
