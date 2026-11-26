# User guide

How to install the flows, what each tunable value does, what the system exports for
monitoring, and the failure modes found in operation. The reasons behind the values are in
[DETAILS.md](DETAILS.md).

## Installation

1. Install the Node-RED modules listed in the [README](../README.md#requirements).
2. In HA, create per zone a `generic_thermostat` whose entity id is
   `climate.<zone>_thermostat`, and make sure the zone's TRV is `climate.trv_<zone>`.
   `ha/thermostats.yaml` is the example.
3. In HA, create the boiler entity, the OTGW keepalive automation and the two switches from
   `ha/boiler.yaml`. Replace `otgw-XXXXXXXXXXXX` with your gateway's MQTT unique id.
   Any `climate` entity whose `set_temperature` writes the OpenTherm control setpoint
   works; the flow only calls `climate.set_temperature` on `climate.boiler_direct`.
4. Import `flows/heating.json` and `flows/heating-controls.json`. Both contain the same
   `server` config node: on the second import, choose to keep the existing one. Open that
   node and set your HA connection.
5. Edit the zone list (see [Zones](#zones)) and the schedules on the Heating Controls tab.
6. Configure Prometheus to scrape the Node-RED exporter endpoint (see the exporter's README)
   under `job="node-red"`, and load `prometheus/heating.rules.yaml`.
7. Import `grafana/heating-operation.json` and select the Prometheus datasource when asked.
   The dashboard uses this site's OTGW entity ids (`*_opentherm_gateway_otgwnew_otgwnew_*`,
   from the HA `opentherm_gw` integration) and `sensor.temperature` for outdoor; replace them.
8. Deploy with both switches off. Check node status texts and the dashboard, then turn on
   `input_boolean.trv_control_enabled` and `input_boolean.boileroverrideenable`.

## Zones

A zone name `z` maps to `climate.z_thermostat` and `climate.trv_z`. Adding or removing a zone
needs edits in four places:

| where | what |
|---|---|
| `init data`, On Start | `zones` array |
| `format entity dependencies` | `zone_sensors` map: the thermostat's `target_sensor` entities |
| Heating Controls tab | the zone's cron, presence and veto nodes |
| dashboard | `$zone` variable, and the zone regexes in the Zone mode, TRV push and Valve state panels |

The leader function also adds a fixed `backup` zone with target `thermostat_off_temp` and a
room temperature 3 K above it. It can only lead when nothing else has a larger demand.

## Switches

| entity | on | turned off |
|---|---|---|
| `input_boolean.boileroverrideenable` | the PID runs and its output is written every 15 s; while off, the PID receives nothing and its integral holds | `Set 0 to surrender boiler control` sends control setpoint 0; the boiler follows its own thermostat. The HA keepalive stops re-sending CS. |
| `input_boolean.trv_control_enabled` | the PI writes TRV setpoints | every TRV is set to `trv_max_setpoint` (fully open) and the PI state is cleared |

## Tunables

All in the On Start block of `init data` on the Heating v2 tab, unless noted.

### Boiler

| name | value | meaning |
|---|---|---|
| PID node `pb` | 6 | proportional band, K. Full output at a 3 K deficit (pb/2) |
| PID node `ti` | 5000 | integral time, s |
| PID node `td` | 0 | derivative off; `smooth_factor` has no effect at td 0 |
| PID node `integral_default` | 0.45 | output at zero error after a restart |
| PID node `max_interval` | 600 | s. Longer gaps between samples do not integrate |
| `pid_pb` | 6 | must equal the PID node's `pb`; a function node cannot read the PID node's config |
| `flow_idle` | 10 | C, sent when PID output is exactly 0; the OTGW's default setpoint, below the boiler's lower bound |
| `flow_min` | 20 | C, the flow temperature for the smallest positive output |
| `flow_span` | 55 | C, so output 1.0 gives 75 C |
| `flow_exp` | 1.3 | command = flow_min + flow_span * output^flow_exp |
| `thermostat_off_temp` | 16 | target used for a zone whose mode is off (frost floor) |
| `off_lead_margin` | 1.0 | K. An off zone warmer than 17 C cannot lead |
| `pid_step_thresh` | 0.8 | K. Error step that counts as a configuration change |
| `pid_rise_k` | 2 | K. Rise that counts as an external heat source |
| `pid_rise_window_s` | 1800 | s, window for `pid_rise_k` |

The PID node reports its integral inverted: a more positive value lowers the output.
Output = 0.5 - (P + I) / pb, clamped to 0..1, with P = room - target. The resulting flow
command at the current values:

| P + I | output | flow command |
|---|---|---|
| -3 | 1.0 | 75 C |
| -1.5 | 0.75 | 57.8 C |
| 0 | 0.5 | 42.3 C |
| +1.5 | 0.25 | 29.1 C |
| +3 | 0 | 10 C (idle) |

### TRV

| name | value | meaning |
|---|---|---|
| `trv_kp` | 2.0 | K of setpoint per K of room error |
| `trv_ti_s` | 1800 | integral time, s |
| `trv_integral_clamp` | 7 | K, upper limit of the integral |
| `trv_integral_min` | 0 | K, lower limit of the integral |
| `trv_off_decay_s` | 7200 | s, integral decay time constant while the zone is off |
| `trv_deadband_k` | 0.2 | K, subtracted from the error (soft deadband) |
| `trv_write_step` | 1.0 | K, minimum difference to the TRV's reported setpoint before a write |
| `trv_off_temp` | 16 | C, commanded while the zone is off |
| `trv_min_setpoint` | 16 | C |
| `trv_max_setpoint` | 30 | C, also the value used to open all valves |
| delay node `ratelim topic 1msg/2 min` | 2 min | evaluation interval per zone |

Command = target + kp * (deadbanded error) + integral, rounded to 0.5 C. The command sets the
TRV's own control target; it is not a valve position.

### Sensors

| name | value | meaning |
|---|---|---|
| `stuck_ms` | 12 h | no change in a zone's reading for this long marks it stuck |

The immediate check is `current_temperature` null, which a `generic_thermostat` reports only
when its source sensor is `unavailable`. MQTT sensors become unavailable only with
`expire_after` set (documented, HA MQTT sensor integration); 900 s is used here.

### Scheduling

The Heating Controls tab sets HA thermostat mode (`climate.set_hvac_mode`) and target
(`climate.set_temperature`). Its tab description lists the current schedule. Points to keep
when editing:

- Mode crons repeat through their window: every 5 min for the bedroom and nursery pre-heat,
  every minute for living room and drawing room. A presence-driven off is ignored until
  15 min after the zone's last pre-heat tick (`presence -> mode`).
- The 20:00 off for bedroom and nursery is delivered only if the radar reads unoccupied.
  Presence-on fires once per transition, so a wrongly delivered off would not be undone.
- The backyard door is a veto: open for 5 min forces living room off; closing sends
  nothing and the next cron tick restores heat.
- Crons are 6 or 7 field (seconds first), parsed by cron-plus.

## Observability

### Metrics

| metric | type | labels | content |
|---|---|---|---|
| `pid_component` | gauge | `component` | PID `setpoint`, `pv`, `proportional`, `integral`, `derivative`, and `payload`, which is the flow command after scaling. Written only while boiler control is on |
| `heating_demand` | gauge | `zone` | target minus room, every 15 s, for each zone allowed to lead (an off zone above `thermostat_off_temp + off_lead_margin` is skipped) |
| `heating_flags` | gauge | `zone`, `flag` | `is_leader` (incl. `backup`), `sensor_stuck`; per TRV evaluation: `cmd_raw`, `p_term`, `integral`, `push` (command minus target), `cmd_headroom`, and `cmd_sent` on a write |
| `heating_trv_writes` | counter | `zone` | TRV writes including re-sends |
| `heating_errors` | counter | `message`, `node_name`, `node_type`, `node_id` | errors caught on the Heating v2 tab |
| `heating_entity_dependency` | gauge | `entity`, `role`, `zone` | value 1 for every HA entity the tab uses, for the alert join |

The per-TRV flags are written only while TRV control is enabled.

### Dashboard

`grafana/heating-operation.json` has five rows: current state, which zone drives the boiler,
the boiler (flow command against flow and return water, PID terms on the output scale,
modulation), per-zone detail (target, room, TRV command and sensor, demand, TRV push, writes
per day, valve state), and health (dependency availability, stuck flags, errors, TRV
headroom).

### Alerts

`HeatingDependencyUnavailable` fires when an entity in `heating_entity_dependency` is not
available in HA for 10 min. `HeatingDependencyListMissing` fires when the list itself is not
exported, which would make the first alert silent.

### Logs

The leader function logs every integral correction with its cause (`config change in`,
`external heat in`) and every error step it ignored because the lead passed between zones.

## Pitfalls

- **Restart and redeploy reset controller state.** Flow context is in memory unless a
  persistent `contextStorage` is configured in `settings.js`. A deploy that restarts
  `init data` also resets it: TRV integrals restart at 0 and the PID starts from
  `integral_default`.
- **`pid_pb` must equal the PID node's `pb`.**
- **Adding an output to a function node** was not picked up by a modified-nodes deploy
  here; the new output worked only after a Node-RED restart (observed).
- **The Prometheus exporter cannot delete one series.** To remove a dependency from the
  alert, edit `format entity dependencies` and also change its metric config node (for
  example its help text), which clears the whole metric; it is refilled within 60 s.
- **Prometheus series identity changes** with a Node-RED pod restart (`pod`, `instance`) and
  with an HA `friendly_name` change. Every dashboard query aggregates (`max by (...)`);
  keep that when adding panels, or a 7-day view shows duplicates.
- **`node.error(text)` without the msg argument does not reach a Catch node**, so it never
  counts in `heating_errors`. The `node.error` calls in these flows pass `msg`.
- **The OTGW CS override expires** if not repeated within 60 s (documented, OTGW firmware
  command reference). The HA keepalive repeats it every 15 s.
- **Sleepy TRVs.** All TRVs here are battery powered. The Z-Wave ones wake every 30 min and
  report a setpoint one wake late, so the write dedupe re-sends until the report catches up
  (220 writes in one day observed on one valve). The Z-Wave controller delivers only the
  latest queued command, so re-sends cost no valve battery.
- **Danfoss LC-13 (Z-Wave) reports only setpoint and battery**: no valve state, no
  temperature. Their dashboard panels for TRV sensor and valve state are empty.
- **Eurotronic Spirit in `system_mode: auto`** rewrote its own setpoint (observed once,
  undocumented in zigbee2mqtt). The only documented way to take full control is
  `trv_mode: 1` with direct `valve_position`, which disables the device buttons.
- **A `generic_thermostat` stays available when its source sensor dies.** That is why
  `zone_sensors` lists source sensors by hand. A renamed source sensor is not detected.
- **External heat detection** uses 2 K in 30 min. Heating alone never exceeded 1.0 K in
  30 min here, but the boiler was never saturated in that period. A false positive appears
  in the log as `external heat in <zone>` during a cold-start ramp; raise `pid_rise_k`.
