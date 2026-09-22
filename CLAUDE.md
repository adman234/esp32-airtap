# CLAUDE.md

Fork of [SiloCityLabs/esp32-airtap](https://github.com/SiloCityLabs/esp32-airtap)
(CC BY-SA 4.0: derivatives must stay CC BY-SA 4.0 with attribution, **not** MIT).

The fork's own work is the **AirTap Auto Vent** in `Airtap-Tx/Gen-2/auto-vent/`.
It is an ESPHome config for Gen 2 Rev 2 (XIAO ESP32-C6, 4 buttons) that decides
on the device itself when to run the fan, using room temperature and thermostat
setpoints imported from Home Assistant. Read
[`Airtap-Tx/Gen-2/auto-vent/README.md`](Airtap-Tx/Gen-2/auto-vent/README.md)
before touching it. It covers the structure, control model, traps, hardware pins
and bug history.

Everything else in the repo is upstream material. Leave it alone unless asked.

## Structure

- `airtap-auto-vent.yaml` is the shared body. Vents pull it from GitHub as a
  remote package (`packages:` with `url` / `ref` / `files`).
- `example-device.yaml` is the per-vent template: substitutions + `packages:`.
- Real device files live in the user's ESPHome config directory. **Never commit
  real device files, entity IDs, node names or MAC addresses here.** The repo is
  public.

## Working agreements

- Anything pushed to `main` reaches every vent on its next flash. Validate
  before committing. Point a scratch device file at the local body
  (`packages: airtap: !include <path>/airtap-auto-vent.yaml`) next to a dummy
  `secrets.yaml`, then run `esphome config`. Also run `esphome compile` when a
  lambda changes. On Windows, compile from PowerShell, not Git Bash.
- Required substitutions (`name`, `friendly_name`, `room_temp_entity`,
  `thermostat_entity`) have no default in the body. Optional ones do. Keep
  `example-device.yaml` and the README table in sync with the body.
- `name_add_mac_suffix` is `false`. The node `name` is the vent's identity.
  Anything that changes `name` or `name_add_mac_suffix` breaks HA entity IDs:
  flag it loudly and never do it incidentally.
- `restore_value: yes` globals ignore a changed `initial_value` on deployed
  units. Say so whenever you change a default.
- Do not reintroduce `fan.restore_mode: RESTORE_*`, and do not remove the
  `booting` / `auto_applying` guards or the button debounce.
- Zigbee was evaluated and rejected. Do not revisit it without new information.
- The user tests on real hardware. Suggest testing a change on one vent (via a
  branch `ref:`) before rolling it out.
