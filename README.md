# ebusd configuration for the Atag Energion (all-electric)

[ebusd](https://github.com/john30/ebusd) configuration for an **Atag Energion M Plus all-electric heat pump**
(11 kW outdoor unit, Atag energy manager, Atag system interface), with Home Assistant MQTT discovery.

Atag is part of the Ariston group and uses the same BridgeNet eBUS protocol as the Ariston Nimbus / Elco Aerotop
heat pumps. This configuration is a fork of the excellent
[wrongisthenewright/ebusd-configuration-ariston-bridgenet](https://github.com/wrongisthenewright/ebusd-configuration-ariston-bridgenet),
cleaned up and verified for an all-electric Atag installation.

> **Use at your own risk.** Reading is harmless; writing changes heat pump settings. Read the
> [Writing settings](#writing-settings-safely) section before enabling any write.

## Tested setup

| Item | Value |
|---|---|
| Heat pump | Atag Energion M Plus AllElectric, 11 kW outdoor unit (installer manual 420010942900) |
| Bus devices | system interface (master 0x10), energy manager (0x18, master 0x13), heat pump (0x1e) |
| Heating | 1 zone, weather compensation (thermoregulation type 3 = outdoor only), floor heating |
| DHW | Atag tank with sensor on the TNK input, heated by the heat pump |
| ebusd | 26.1, eBUS adapter in enhanced mode over TCP, Home Assistant ebusd app + MQTT |

Other Energion variants (Light/Lightbox, hybrid, cooling, multiple zones) share most registers, but lines that
are not relevant for the tested setup are disabled. See also
[hesselonline/atag-ebusd](https://github.com/hesselonline/atag-ebusd) (Energion Light).

## Files

| File | Purpose |
|---|---|
| `ebus/ariston.csv` | Message definitions (kept under the upstream name so diffs against upstream stay readable) |
| `ebus/_templates.csv` | Field templates (from upstream) |
| `ebus/broadcast.csv`, `ebus/memory.csv` | Standard ebusd definitions (memory circuit is filtered out of HA) |
| `mqtt-hassio.cfg` | Home Assistant MQTT discovery configuration |
| `homeassistant/scripts.yaml` | Bounded HA scripts to change the pump PWM and the flow temperature correction safely |

Copy the `ebus/` folder to your ebusd config path and point ebusd's `--mqttint` to `mqtt-hassio.cfg`.

## Changes compared to the Ariston configuration

**Disabled (not present on an all-electric Atag)** – every disabled line keeps its original definition behind an
`#off[reason]` prefix, so it is easy to re-enable for other setups:
- gas boiler circuit (0x3c), gateway (0x84) and Sensys devices that do not respond;
- hybrid-only cost/efficiency registers (they read `0x8000` = not set);
- zones 2 and 3 (filtered out in `mqtt-hassio.cfg`).

**Fixed**
- Thermoregulation type labels 2/3 were swapped: per Atag manual 4.2.1, 2 = indoor only, 3 = outdoor only.
- Six heat pump reads lacked the leading `IGN:1` header byte (e.g. `compressor_state` was stuck "on").
- `hp_delta_t_setpoint` in °C like `em_delta_t` (unit K is interpreted as kelvin by Home Assistant).
- PV Delta T DHW write register: `762d` (the old line wrote `6271` = zone 1 day temperature!). Kept disabled.
- `z1_heat_temp_range` is read-only on purpose: "high" forces the radiator curve (slope ≥ 1.0, max 60 °C) onto
  floor heating.

**Added**
- Registers decoded from energy manager ↔ heat pump traffic: flow setpoint sent to the heat pump, requested mode,
  modulation limit, delta-T setpoint, energy manager state, zone 1 flow setpoint, system return temperature,
  electric energy counters (heating / DHW / cooling).
- Electric backup heater state and hour counters (manual 17.9.x).
- Pump PWM min/max (manual 17.3.5/17.3.6) with the system limits (`2001` query returns value + min + max).
- Read-only: quiet mode window, DHW thermal cleanse settings, max HP charging time, and three energy manager
  registers whose names come from [claytonn73/ariston-nimbus-ebusd](https://github.com/claytonn73/ariston-nimbus-ebusd)
  (`pump_prerun`, `retry_time`, `pump_overrun`; not in the Atag manual, unconfirmed).
- Write definitions: pump PWM min/max and flow temperature correction (manual 17.2.3).

**Home Assistant (`mqtt-hassio.cfg`)**
- Writable messages exposed (`filter-direction = r|u|^w`), bus reset and auto function never exposed.
- Read-only time fields (VTM/HTM) get a `type_part`, otherwise they were never published.

## Writing settings safely

Lessons learned the hard way:

1. **ebusd cannot enforce a value range on writes.** There is no min/max in the CSV syntax; the `min`/`max` that
   `mqtt-hassio.cfg` sends to HA come from the raw data type (e.g. −3276.7…3276.7) and only shape the HA slider.
   Use the scripts in `homeassistant/scripts.yaml` (hard bounds + read-back) and disable the raw number entity.
2. **The value of a write must be in the master part.** A direct write (e.g. to the heat pump `1e`) with the value
   field in part `s` makes ebusd send the register **without a value**; the heat pump then set its own values
   (min PWM 76 %, max 100 %). Use part `m` (or leave it empty). Always test a new write definition with the
   *current* value first and check that the ebusd log shows `sent write ...` (not `unable to parse write`).
3. **Pump PWM must be written directly to the heat pump (`1e`)**; a broadcast write (`fe`) is ignored.
4. **Message names are matched as substrings by `mqtt-hassio.cfg`.** A name containing `temp` becomes a temperature
   sensor – `new_attempt_time` (at-*temp*-t) was rejected by HA because of its unit `min`, hence `retry_time`.

## Notes and open questions

- `heatpump_mode` value 15 (`Soft_fail`) is not listed in the Atag manual (17.11.0 ends at 14 = Pump Down), but the
  Nimbus configuration lists the same sequence (12 system fail, 13 hard system fail, 14 pump down, 15 soft fail).
  On the tested system it appears at compressor stops with low water flow (~12-13 l/min), without an error code.
- The energy counters use divider 352 (derived from integrated power); claytonn73 uses 360 for the same registers.
  Not verified against an independent meter.
- The *produced energy* counters that the Ariston Nimbus provides (`46d9`…`51d9`, read as low/high word pairs) all
  read 0 on this Atag, so a COP cannot be taken from the heat pump itself. It can be estimated in Home Assistant
  from flow × (LWT − EWT) × 4.186 and the electrical power.
- `hydraulic_scheme` reads 3 (= Flex on this installation), which is not in the inherited value list.
- `filter-seen = 5` auto-polls every exposed message. Fine on this Atag; on an Ariston Cares S gas boiler this was
  reported to trigger bus address scans and boiler reboots (see
  [Syax89/Ariston-Cares-S-Ebusd](https://github.com/Syax89/Ariston-Cares-S-Ebusd)).
- New entities only appear in HA after ebusd republishes its discovery messages (e.g. after an HA restart or an
  MQTT `homeassistant/status` = `online` message).

## Credits

- [wrongisthenewright/ebusd-configuration-ariston-bridgenet](https://github.com/wrongisthenewright/ebusd-configuration-ariston-bridgenet) – the base of this configuration
- [ysard/ebusd_configuration_chaffoteaux_bridgenet](https://github.com/ysard/ebusd_configuration_chaffoteaux_bridgenet) – protocol and error codes
- [claytonn73/ariston-nimbus-ebusd](https://github.com/claytonn73/ariston-nimbus-ebusd) – Nimbus registers, `2001` limits, write behaviour
- [hesselonline/atag-ebusd](https://github.com/hesselonline/atag-ebusd) – Atag Energion Light configuration

## License

GPL-3.0, like the upstream configuration it is derived from. See [LICENSE](LICENSE).
