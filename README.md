# heating-control

Node-RED flows that control a gas boiler and per-room thermostatic radiator valves (TRVs)
through Home Assistant (HA). The boiler is driven over OpenTherm by an OpenTherm Gateway
(OTGW). Each room has a virtual thermostat in HA. The flows decide the boiler flow
temperature and every TRV setpoint, and export their internal state to Prometheus for a
Grafana dashboard and alerts.

It has run a five-room house since October 2026. All design values were tuned against
measurements from that house; [docs/DETAILS.md](docs/DETAILS.md) has the numbers.

## Why

- A boiler thermostat measures one room. The other rooms are guessed.
- A TRV regulates on its own sensor, which is next to a hot radiator and often behind a
  curtain. Here the error between TRV sensor and room sensor changed by about 2.4 K with
  valve state, so no static calibration offset is right in both states.
- Rooms are used on different schedules, and one room's demand should not overheat the
  others.

## How it works

- **Boiler loop, every 15 s.** The room with the largest deficit (target minus room
  temperature) is the leader. Its error feeds a PID controller
  ([node-red-contrib-pid](https://github.com/colinl/node-red-contrib-pid)). The 0..1 output
  is mapped to a flow temperature of 20 to 75 C on a convex curve and written to the boiler
  as the OpenTherm control setpoint. Output 0 sends 10 C, the OTGW's idle default, which is
  below the boiler's lower bound.
- **TRV loop, each zone every 2 min.** A PI controller per room reads the room sensor and
  writes the TRV setpoint. The integral absorbs the TRV's own sensor bias. A write is sent
  only when the command differs from what the TRV reports by 1 K, because the TRVs are
  battery powered.
- **Integral correction.** When the leading room's mode or target changes, or a room
  warms by 2 K in 30 min from a heat source the loop did not command (sun, fireplace), the
  PID integral is corrected by the error step so the loop does not spend hours unwinding
  stale history.
- **Sensor health.** A room whose sensor is unavailable, or unchanged for 12 h, uses the mean
  of the healthy rooms.
- **Scheduling.** A second flow sets each room's HA thermostat mode and target from cron
  schedules, radar presence and a door contact.
- **Hand-over.** Two HA switches enable boiler and TRV control. Turning boiler control off
  sends control setpoint 0, which returns the boiler to its own thermostat. Turning TRV
  control off opens every valve.

## Requirements

Tested with these versions:

| component | version |
|---|---|
| Node-RED | 3.1.6 |
| node-red-contrib-home-assistant-websocket | 0.76.0 |
| node-red-contrib-pid | 2.1.0 (minimum: the flow sets `msg.integral`) |
| node-red-contrib-cron-plus | 1.5.7 |
| node-red-contrib-prometheus-exporter | 1.0.5 |
| Home Assistant | 2024.9.3, with the `prometheus` integration |
| OTGW | NodoShop OTGW with OTGW-firmware, MQTT topics as in `ha/boiler.yaml` |
| Grafana | 11.5.1 |

The HA side needs, per zone, `climate.<zone>_thermostat` (a `generic_thermostat`) and
`climate.trv_<zone>`, plus `climate.boiler_direct` and two `input_boolean` switches.
Setup, every tunable value, and the known pitfalls are in
[docs/USER_GUIDE.md](docs/USER_GUIDE.md).

## Repository contents

| path | content |
|---|---|
| `flows/heating.json` | Heating v2 tab: boiler loop, TRV loop, metrics |
| `flows/heating-controls.json` | Heating Controls tab: per-zone mode and target schedules |
| `grafana/heating-operation.json` | Grafana dashboard, imported with a Prometheus datasource input |
| `ha/` | Example HA configuration: boiler climate entity, OTGW keepalive, thermostats |
| `prometheus/heating.rules.yaml` | Alert on an unavailable HA entity the flow depends on |
| `docs/USER_GUIDE.md` | Installation, tunables, observability, pitfalls |
| `docs/DETAILS.md` | Design decisions, measurements, rejected alternatives |

## Credits

- [node-red-contrib-pid](https://github.com/colinl/node-red-contrib-pid) by Colin Law. The
  boiler loop is built on it. Version 2.1.0 added runtime integral setting (`msg.integral`),
  which the integral correction depends on; thanks to Colin for reviewing and merging it.
- [OpenTherm Gateway](https://otgw.tclcode.com/) by Schelte Bron: the gateway hardware design
  and PIC firmware that make OpenTherm control from outside the thermostat possible.
- [OTGW-firmware](https://github.com/rvdbreemen/OTGW-firmware) by Robert van den Breemen: the
  ESP firmware for the NodoShop OTGW that provides the MQTT interface used here.
- [node-red-contrib-home-assistant-websocket](https://github.com/zachowj/node-red-contrib-home-assistant-websocket)
  by zachowj, [node-red-contrib-cron-plus](https://github.com/Steve-Mcl/node-red-contrib-cron-plus)
  by Steve-Mcl, and [node-red-contrib-prometheus-exporter](https://github.com/Docoyo/node-red-contrib-prometheus-exporter)
  by docoyo.

## Licence

[WTFPL](LICENSE). The dependencies above have their own licences.
