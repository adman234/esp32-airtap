# AirTap Auto Vent (Gen 2 Rev 2, XIAO ESP32-C6)

ESPHome firmware that turns an AC Infinity AirTap T-series register booster into
a self-deciding HVAC assist fan. The stock controller is replaced by the Gen 2
Rev 2 board from this repo (Seeed XIAO ESP32-C6). The original four panel
buttons and the SSD1306 OLED still work. An NTC probe reads duct temperature, and
the fan is driven over PWM. The device talks to Home Assistant over WiFi using
the ESPHome native API.

**The vent decides for itself when to run.** No HA helpers, template sensors or
automations are involved. HA only supplies inputs.

| File | Purpose |
|---|---|
| [`airtap-auto-vent.yaml`](airtap-auto-vent.yaml) | Shared body, pulled by every vent as a remote ESPHome package. Never flashed directly |
| [`example-device.yaml`](example-device.yaml) | Device file template: substitutions + `packages:`. Copy one per vent |
| [`secrets.yaml.example`](secrets.yaml.example) | Secrets the body expects |

Derived from [`../esphome-4btn-rev2.yaml`](../esphome-4btn-rev2.yaml) by
SiloCityLabs, licensed **CC BY-SA 4.0**. ShareAlike applies, so this derivative
is CC BY-SA 4.0 as well. Keep the attribution intact.

---

## How it's structured

Each vent's config lives in your ESPHome config directory (not in this repo) and
is only a few lines long:

```yaml
substitutions:
  name: "airtap-esp32-4btn-3"
  friendly_name: "AirTap Vent Bedroom"
  room_temp_entity: "sensor.bedroom_temperature"
  thermostat_entity: "climate.my_thermostat"

packages:
  airtap:
    url: https://github.com/adman234/esp32-airtap
    ref: main
    files: [Airtap-Tx/Gen-2/auto-vent/airtap-auto-vent.yaml]
    refresh: 0s
```

At compile time ESPHome clones this repo, loads `airtap-auto-vent.yaml`, and
applies the device's substitutions to it. Everything that makes a vent work
lives in one place, and every vent runs the same logic.

### Substitutions

| Key | Required | Default | What to put |
|---|---|---|---|
| `name` | yes | | Unique node name = hostname = HA device. Lowercase, `a-z 0-9 -`, at most **31** chars |
| `friendly_name` | yes | | For example `AirTap Vent Bedroom` |
| `room_temp_entity` | yes | | HA temperature sensor for that room. **Must natively report °F** |
| `thermostat_entity` | yes | | HA `climate.` entity. **Must be in `heat_cool` mode** |
| `default_speed` | | `4` | Resting AUTO speed, 1–10 (4 = 40 %) |
| `log_level` | | `INFO` | |
| `device_address` | | `${name}.local` | Where to reach the vent for OTA and logs. Only override during a rename (see below) |

`!secret` values come from the `secrets.yaml` in **your** ESPHome config
directory, the same way as for any other ESPHome device.

---

## Building a new vent

1. **Hardware:** fit the Gen 2 Rev 2 board to a 4-button AirTap. See the
   [main README](../../../Readme.md) and the brackets in `3D-Models/4btn-v3`.
2. **Secrets:** copy `secrets.yaml.example` to `secrets.yaml` in your ESPHome
   config directory and fill it in. Skip this if an earlier vent already did it.
3. **Device file:** copy `example-device.yaml` to `<name>.yaml` in your ESPHome
   config directory and fill in the placeholders. The file name should match
   `name`.
4. **First flash over USB:** `esphome run <name>.yaml`, then pick the serial port.
5. **Adopt in HA:** it is auto-discovered as `<name>`. Enter the API key from
   `secrets.yaml`.
6. **Check it:** after boot the OLED shows `AUTO`, and `Auto Fan Demand` in HA
   follows the room and thermostat.

Keep your own list of which node name belongs to which room. Don't commit
per-device files to this repo.

---

## Updating vents

1. Edit `airtap-auto-vent.yaml`, validate it (see [Validating](#validating)), and
   push to `main`.
2. Recompile and flash each vent over the air: `esphome run <name>.yaml`, or
   **Update All** in the HA ESPHome dashboard. `refresh: 0s` makes every compile
   fetch the latest `main`.

A vent only changes when it is reflashed. Until then it keeps running whatever
it was last built from.

**Pinning:** to freeze a vent on a known-good version, set `ref:` in its device
file to a tag or commit SHA instead of `main`. Tag a release with
`git tag auto-vent-v2.0.0 && git push --tags`. Test a change on one vent by
pointing only that vent at a branch.

**Compiling needs internet access**, because the package is fetched from GitHub
on every build. If GitHub is unreachable, set `refresh:` to something like `1d`
so a cached copy gets used.

---

## Node names and entity IDs

`name_add_mac_suffix` is **`false`**. The node name you give each vent is its
identity: its hostname (`<name>.local`), its HA device, and the prefix of every
entity ID it owns. That means:

- **Every vent needs a unique `name`.** Two vents with the same name will fight
  over the same hostname and HA device.
- **Renaming a deployed vent changes every entity ID it owns.** Dashboards and
  automations that reference them then break.

### Renaming a vent

This also applies to a unit that was previously flashed with
`name_add_mac_suffix: true`. That unit currently answers at
`<name>-<mac6>.local`, and its entity IDs contain the MAC suffix.

1. In the device file, set `device_address` to the address the vent answers on
   **right now** (the old one).
2. Flash it OTA. The vent reboots under its new name.
3. Remove the `device_address` override, so it falls back to `${name}.local`.
4. In HA, update any dashboards or automations that used the old entity IDs,
   then delete the stale entities if HA kept them.

---

## Control model

HA pushes these inputs over the native API (`sensor: platform: homeassistant`):

- room temperature
- `target_temp_high`
- `target_temp_low`
- `hvac_action`, as a text sensor

Everything else is computed on the device, so the vent keeps running on its
last-known inputs if HA restarts.

`binary_sensor.auto_fan_demand` is the single source of truth. Its `on_state` is
the **only** way into the `apply_auto` script. The input sensors deliberately
have no triggers, so demand is never evaluated from stale inputs.

```
cooling assist:  room > cool_sp   AND  vent < (room - auto_buffer)
heating assist:  room < heat_sp   AND  vent > (room + auto_buffer)
```

Hysteresis is on the **stop** side. Once the fan is running, the
room-vs-setpoint test relaxes by `room_deadband`, so the fan runs a little past
the target instead of stopping short of it:

```
cooling:  start at room > cool_sp,  stop at room <= cool_sp - deadband
heating:  start at room < heat_sp,  stop at room >= heat_sp + deadband
```

The deadband used to widen the *start* threshold instead. With a 75 °F setpoint
and a 0.5 °F deadband, the fan only started above 75.5 °F. A room that sat at
exactly 75.5 °F never triggered it, because the test is a strict `>`.

**All temperatures are °F.** The NTC calibration is in °C (correct physics) and
the output is converted. HA-imported values arrive with no unit conversion.

### Resting state

Every boot lands in **AUTO at `default_speed`** (40 % by default). `auto_mode`
and `auto_speed` are `restore_value: no` on purpose. Only three things switch the
vent to MANUAL:

- a Power, Up or Down press on the panel
- the Mode button
- someone turning the fan on or off from HA

The **Reset To Auto Default** button in HA returns the vent to its resting state.

### Tuning (HA `number`/`switch` entities, `restore_value: yes`)

| Setting | Default | Meaning |
|---|---|---|
| Auto Buffer | 1.0 °F | How much better the duct air must be than the room before running is worthwhile |
| Room Deadband | 0.5 °F | How far to run **past** the setpoint before stopping |
| Min Run Time | 30 s | Minimum time on, to prevent short-cycling |
| Min Off Time | 30 s | Minimum time off, to prevent short-cycling |
| Require HVAC Active | off | When on, demand also requires `hvac_action` to be `heating` or `cooling` |
| Disable Panel Buttons | off | Makes the physical buttons do nothing |
| OLED Brightness | 100 % | |
| Auto Fan Speed | `default_speed` | Not restored. Resets on every boot |

These are set per vent from HA, not in YAML.

---

## Known traps

1. **`restore_value: yes` ignores the flashed `initial_value`** once a value has
   been stored. Changing a default in the shared body may not take effect on a
   vent that is already deployed. Check the entity in HA and set the value there.
2. **The thermostat must stay in `heat_cool` mode.** `target_temp_high` and
   `target_temp_low` only exist in dual-setpoint mode. In Cool-only or Heat-only
   mode they disappear, the imported values become NaN, and demand stays false
   permanently. This fails silently and closed.
3. **The room sensor must report °F.** A °C sensor produces nonsense comparisons
   without any error.
4. **A push to `main` reaches every vent on its next flash.** Validate before
   pushing, or pin vents to a tag.
5. **Zigbee was evaluated and rejected.** ESPHome's Zigbee component only
   supports `light`, `switch`, `binary_sensor` and `sensor`. It has no equivalent
   of `sensor: platform: homeassistant`, which the whole control model depends
   on. It also has no `fan` or `number` platform, and no OTA. WiFi with the
   native API is strictly better here. (The upstream `zigbee-4btn-rev2/` folder
   is a separate ESP-IDF project.)
6. **Validation warnings** (ESPHome 2026.9). `web_server` auth defaults to
   `basic`, which will change to `digest` in 2027.1. The `web_server` OTA
   endpoint is not covered by OTA encryption. Both are harmless for now.
7. **Compiling on Windows:** run `esphome` from PowerShell or cmd, not Git
   Bash. ESP-IDF refuses to install under MSYS/MinGW.

### Validating

Validate against your local working copy before pushing. Point a scratch device
file's package at the local file instead of GitHub:

```yaml
packages:
  airtap: !include /path/to/esp32-airtap/Airtap-Tx/Gen-2/auto-vent/airtap-auto-vent.yaml
```

Then run `esphome config scratch.yaml`, followed by `esphome compile scratch.yaml`
for anything that touches a lambda.

Last verified with `esphome config` and a full `esphome compile` (no errors) on
ESPHome 2026.9.0 / ESP-IDF 5.5.5, 2026-09-22.

---

## Hardware reference

- Board `seeed_xiao_esp32c6`, framework `esp-idf` (required for the C6)
- **PWM:** `ledc` on **GPIO0** at 1000 Hz. Speed 1 → duty 0.38 (the minimum that
  starts the fan). Speed 2 and up → `(speed + 4) / 14`
- **NTC:** ADC on **GPIO2** (12 dB) with a 10 kΩ downstream divider. Calibration
  points: 3.389 kΩ → 0 °C, 10 kΩ → 25 °C, 27.219 kΩ → 50 °C
- **I²C OLED:** SDA **GPIO22**, SCL **GPIO23**, SSD1306 128×64 at `0x3C`
- **Buttons** (inverted, pull-up, 20 ms debounce): Power **GPIO20**, Up
  **GPIO17**, Down **GPIO19**, Mode **GPIO18**
- **No BLE.** `esp32_improv` was removed so the single 2.4 GHz radio serves WiFi
  only, which allows `power_save_mode: NONE`. `improv_serial` is kept because it
  runs over UART and costs nothing.

---

## Bug history: don't reintroduce these

**Reboots silently dropped the device into MANUAL.** The symptom was "the vents
keep going into manual mode and changing speed on their own", and it was long
blamed on the hardware. The cause:

1. `fan.restore_mode: RESTORE_DEFAULT_OFF` restores state inside `Fan::setup()`.
2. `FanRestoreState::apply()` calls `publish_state()`, which fires the callbacks
   that `FanTurnOnTrigger` listens on.
3. That trigger is edge-triggered, with `last_on_` initialised to `false`. A fan
   restoring to *on* therefore looks like a real off→on transition and fires
   `on_turn_on`.
4. The `on_turn_on` lambda cleared `auto_mode`, because the `auto_applying` guard
   is false during boot.
5. `apply_auto` is gated on `auto_mode`, so the fan held its restored speed
   forever.

The fix was `restore_mode: ALWAYS_OFF`, plus a `booting` global that guards both
fan triggers.

A second cause produced the same symptom: the panel buttons were not debounced.
Contact bounce on **Mode** toggled `auto_mode` more than once per press. All
buttons now have a 20 ms debounce.
