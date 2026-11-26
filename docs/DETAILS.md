# Design details

Why the flows are built the way they are: the plant they control, the decisions behind each
mechanism, and the alternatives that were measured and rejected. Numbers are measured on this
house in 2026 unless marked *documented* (vendor documentation), *simulated* (recorded data
replayed through the real PID node or a plant model) or *inferred*.

## The plant

### Boiler

- Vaillant ecoTEC Plus with OpenTherm, OTGW member id 24. Max CH output is set on the
  boiler panel (`d.00`), not over OpenTherm (Vaillant installation manuals; not checked
  against this unit's manual).
- Modulation sits at its 17 % floor for 86 % of firing time. The flame is on about 15 % of
  the time, so average demand is below the minimum output: the burner fires, overshoots,
  stops and refires, about 34 central heating (CH) ignitions per day. That cycling is set by
  the hardware; lowering flow temperature does not remove it.
- 99 % of CH burn time has return water below 54 C, so the boiler condenses. Domestic hot
  water (DHW) burns do not, which brings the all-burns figure to 93 %.
- DHW does not disturb the room loop: over 19 DHW episodes the PID integral moved by a
  mean of -0.003.
- The OTGW echoes a new control setpoint within about 2 min. Water reaching the setpoint
  takes a median 3 min on small steps and up to more than 60 min on a cold start.
- Commanding 10 to 20 C never produced heat: in every such minute the boiler water was
  already hotter than the command.

### Rooms and emitters

- The living room has a large radiator with no valve, acting as the system bypass, and a
  small one with a TRV. The zone follows its target mostly regardless of the TRV command,
  and the bypass is why closing every other valve causes no short-cycling.
- The workshop is part of the attic and runs warm by structure: over 14 days it was above
  its 14.2 C target 100 % of the time with heating off.
- Free-cooling time constants are 21.6 to 37.2 h per zone. The fastest sustained heating
  rate seen anywhere is 1.0 K/h.

### TRVs

| zone | device | protocol | reports |
|---|---|---|---|
| bedroom, nursery | SONOFF TRVZB | Zigbee | setpoint, temperature, running state |
| workshop | Eurotronic Spirit SPZB0001 | Zigbee | setpoint, temperature, running state, valve position |
| living room, drawing room | Danfoss LC-13 | Z-Wave | setpoint and battery only |

All are battery powered. The Z-Wave valves wake every 1800 s; a valve wakes, reports its
current value, accepts the queued command and reports the new value at the next wake, so the
report lags the command by one wake. The living-room LC-13 occasionally reports -0.0 or
2736.0 for a single sample (Z-Wave decode artefacts, 15 in 14 days).

TRV sensors disagree with room sensors depending on valve state. On the bedroom TRV a
calibration of -2.4 K was right with the radiator hot and left the sensor 3.25 K low with it
cold. No static offset is right in both states, and the Sonoff `external_temperature_input`
was tried and held stale values. So room temperature is never routed through the TRV.

### Room sensors

Workshop, nursery and drawing room use Bluetooth sensors bridged to MQTT by Tasmota,
publishing every 300 s. They sometimes stop, and HA keeps serving the last value. Bedroom and
living room use Zigbee sensors through Zigbee2MQTT. Healthy sensors hold one value for up to
7.7 h overnight.

## Boiler loop

### One actuator, one leader

Flow temperature is a single house-level actuator, so it follows the room with the largest
deficit; the per-room valves deal with the rest. Per-room disturbances (sun on one facade,
wind on another) belong to the valves.

Weather compensation was rejected: scalar irradiance or wind cannot say which facade is
affected, and the PID on room error already compensates load by feedback. A regression of
required water temperature on outdoor temperature looked predictive until it turned out to be
confounded by which zones were heating at which time of day. Irradiance feedforward for the
sunset cool-down was rejected too: the local loss of sun is a step when the sun goes behind a
neighbouring building, while the regional irradiance sensor declined smoothly and outdoor
temperature did not change.

### Off zones

A zone whose mode is off keeps a 16 C target as a frost floor. Above 17 C
(`off_lead_margin`) it cannot lead. Before that rule, an off zone at 17.8 C reported +1.8 K of
"too warm", which wound the integral to its clamp: over 14 days off zones led for 2.2 h, every
sample with P above +0.5.

Mode is read from the thermostat state, not `hvac_action`. A `generic_thermostat` reports
`hvac_action: off` briefly during a mode change; keyed on that, the living room was held at
16 C for 110 min while 1 K cold (07 Oct).

### PID node behaviour

From the source of node-red-contrib-pid, verified by running the module with a stubbed clock:

- power = -(P + I + D) / pb + 0.5, P = pv - setpoint, clamped to 0..1. Full output is at an
  error of pb/2, not pb.
- `integral_default` is the output at zero error after a start. The internal integral is
  inverted (higher means less output). `integral_default` 0 starts the integral at its
  clamp: zero output after every restart, about 4 h to recover from a 1 K deficit
  (*simulated*). 0.45 is used.
- Anti-windup: the integral updates only while |P + I| < pb/2. The lock clears without help,
  because P follows the room.
- `max_interval`: a longer gap between samples holds the integral, so an outage does not
  integrate a huge time step.
- `smooth_factor` has no effect at `td` 0.

### Proportional band

pb went 2.2, then 8, then 6. At 2.2 a 1.1 K deficit commanded 75 C, nearly on/off. At 8,
heating events started too low because the start command is set almost entirely by the
integral. Replaying 14 days of recorded rooms through the real node (*simulated*, open loop):

| pb | median integral | median first command | worst first command | worst lag to 25 C |
|---|---|---|---|---|
| 8 | 2.74 | 32.1 C | 24.1 C | 36 min |
| 6 | 1.74 | 36.1 C | 25.5 C | 0 |
| 4 | 0.78 | 44.1 C | 28.3 C | 0 |

A stronger P holds the operating point, so the integral stays small, and a small integral
both shortens the start and reduces overshoot from slow unwinding. pb 6 was the smallest
change that cleared the defect. Measured over 38 h after the change: median integral 2.60 to
1.53, every event started above 25 C with no lag, no hunting (0.3 error sign changes per
hour). Median event duration did not change (2.20 h against 2.27 h).

Derivative was rejected: room sensors report in 0.1 K steps and D turns the staircase into
spikes. At td 300 the worst-case command rose from 58.0 to 72.4 C (*simulated*). Shortening
`ti` was rejected as well: 5000 s is already short against 22 to 37 h cooling constants.

### Output curve

command = 20 + 55 * output^1.3, and 10 for output exactly 0.

- The 20 C floor: below it the boiler produces nothing (see the plant section).
- 10 C for output 0. OpenTherm gives no setpoint value a meaning: the control setpoint is 0
  to 100 C, and "no CH demand" is signalled by CHenable = 0 (*documented*, OpenTherm Protocol
  Specification v2.2, data-id 1). 10 C is the OTGW's own idle default, which "will not
  normally cause the boiler to start heating" (*documented*, OTGW stand-alone operation
  page). The flow sends no `CH` command, so it relies on 10 C being below the boiler's 20 C
  lower bound, where the burner does not fire (*observed*). Sending `CH=0` at idle would be
  the protocol-correct signal; it is not implemented.
- 0 is reserved for hand-over: CS=0 passes the thermostat's value through (*documented*, OTGW
  firmware command reference).
- The exponent: a linear map puts the zero-error point at the midpoint of the range, 47.5 C.
  Over steady hours the house actually receives 27 to 44 C mean water. A lower midpoint with
  a linear map would cost the 75 C ceiling. Radiator output goes roughly as deltaT^1.3
  (EN 442 convention, not measured here), so a convex command also makes equal output steps
  closer to equal heat steps.
- k was 1.4 (zero point 40.8 C) for four days, then 1.3 (42.3 C): at 1.4 a 0.8 K overnight
  deficit commanded only 34.6 C and recovered at 0.2 K/h.

| 14-day replay (*simulated*) | linear | k 1.4 |
|---|---|---|
| median command | 39.3 C | 32.7 C |
| hours above 55 C | 24.2 | 7.2 |
| morning recovery p90 | 59.2 C | 54.2 C |

Rejected:

- Capping output at 60 C on a wider span: the PID's anti-windup cannot see a cap outside
  the node, so the integral would wind against it.
- A 75 C boost when the loop "does not converge": the PID reaches 75 by itself at a 3 K
  deficit, and the boost let the integral wind while it held the output.
- An emergency override on large deficit: the largest deficit in 21 days was 2.4 K, and at
  2.0 K or more the PID already commanded a median 51.5 C.
- Bypassing the PID when no room wants heat: at steady state the error crosses zero
  constantly, and the bypass switched the boiler 301 times per day (*simulated*).

### Integral correction

The PID's error is the leader's target minus its temperature. It can change for reasons
unrelated to the heating: a schedule, a presence change, a manual edit, the lead passing to
another zone, or heat from a fireplace or the sun. The integral built up before then no longer
applies, and at `ti` 5000 s it takes hours to unwind. After a 00:30 setback the command decayed
from 59 to 44 C over three hours with no zone wanting heat.

The leader function sends `msg.integral` to the PID (node-red-contrib-pid 2.1.0) in two cases:

1. **The departing leader's own mode or target changed and the error stepped by 0.8 K or
   more.** Over 7 days, 175 configuration changes produced 23 downward steps, with no step
   between 0.60 and 0.80 K: manual edits of 1 K show up as 0.6 K or less because the lead
   usually passes to another zone. The test is on the departing leader's entry only. When it
   was on any zone, a second zone flickering into the lead while presence changed elsewhere
   stacked three corrections and took the integral from 0.28 to 3.27 (08 Nov). Both
   directions count: with the integral pinned at its clamp overnight, raising the bedroom
   target took two hours of unwinding before the command responded (17 Nov).
2. **A zone rose 2 K or more in 30 min while the leader is at or above target.** Heating alone
   never moved a zone more than 1.0 K in 30 min over 21 days. A replay of one afternoon fired on
   sun through the skylights (+2.00 K) and on the fireplace (+2.60 K). A latch gives one
   correction per episode. A slow fire stays under the threshold and is missed.

The correction cancels the step; it does not zero the integral. Zeroing means output 0.5,
which raised idle commanding from 8.4 to 11.1 h per day (*simulated*) and once took the
command from 27.9 to 53.9 C on a step that asked for less heat.

The first version keyed on the error step alone. 7 of its 8 fires in one day were the lead
passing between rooms, whose temperature gap exceeds 0.5 K in 40.6 % of samples, so no
threshold can separate the two cases.

This is not a derivative term: it fires once on a discrete event and changes stored state.

### Valve authority before water temperature

A proposed rule raised water temperature only when the leading zone's valve had no travel
left. Exported headroom never fell below 5.4 K (to 03 Nov), so the rule would never have
acted and was not built.

## TRV loop

### Control law

PI on the room sensor; the output is the TRV setpoint. One law for all five models, with no
per-model code. Each TRV keeps its own internal loop, and the integral absorbs its sensor bias.
Over 14 simulated days:

| law | writes per day | valve travel, % of stroke per day | rms error |
|---|---|---|---|
| previous, 5x amplified | 411 | 26314 | 0.099 K |
| binary 30/16 | 132 | 13143 | 0.136 K |
| PI, kp 2, ti 1800 s, step 1 K | 20 | 722 | 0.283 K |

Travel is what costs battery and wear. Binary control cycles constantly when one room is cold
and another warm.

- **Soft deadband.** The 0.2 K deadband is subtracted from the error. A hard deadband makes the
  command jump by kp times 0.2 at its edge, and every crossing costs a write.
- **Write step 1 K.** A 0.5 K step gave 634 writes per day (*simulated*).
- **Dedupe against the reported setpoint.** Comparing against the last value sent was not
  self-correcting: the living-room valve reverted from 16 to 18.5 six minutes after a write,
  and no rewrite followed for 9.9 h (21 Oct). HA is optimistic after a service call and
  corrects the entity when the device reports. A TRV with no state is always written.
- **Integral lower limit 0.** We have heating and no cooling, so a command below target is
  dead range. A room 0.4 to 0.7 K above target for 8 h wound the integral to -1.05, and the
  next morning's command of 19.55 against a 20.0 target could not open the valve. A -0.5
  limit still left a room 0.1 to 0.2 K short overnight.
- **Integral upper limit 7** (was 5). The TRV head reads high when its own radiator is hot,
  so the valve throttles itself while the room is short. Over 14 days only two zones reached 5
  (5.2 % and 3.1 % of samples); `cmd_raw` peaked at 27.9 C, inside the 30 C ceiling.
- **Decay while off.** A zone goes off for a scheduled setback or for a presence dropout of a
  still occupant, and the node cannot tell them apart. Zeroing on off threw away an integral
  of 4.37 over a 38 min radar dropout, and the room sat 0.3 K short until morning. Over 30 days
  bedroom had 34 off periods under 2 h; a 2 h time constant keeps 73 % over 38 min and almost
  nothing overnight.
- **Authority check next to the PI.** With the enable check four nodes upstream behind a
  queue, the PI ran 38 s after control was switched off and closed five valves that had just
  been opened (17 Nov).
- **Open all valves on disable.** A valve left at its last command is still controlled, by a
  stale command; for an off zone that means shut. Opening serves maintenance, pump circulation
  with heating off, and hand-over to the hardware thermostat.

### Accepted limitations

- The living-room TRV command chatters, but its radiator contributes little; no special
  case was added.
- The Z-Wave valves apply whatever command is queued at their 30 min wake while the PI
  recomputes every 2 min. Slowing the command rate for those zones is an open tuning item.
- The Spirit in `system_mode: auto` rewrote its own setpoint once (16 to 24 C, valve open
  for 99.5 h, with no write from Node-RED). The workshop zone is off, so it was left.

## Sensor health

- Over 30 days a 3 h no-change rule fired 19 times, all false positives, nearly all
  overnight. So no-change is a 12 h backstop.
- HA `last_updated` freezes together with the value and is not a liveness signal; HA
  availability showed 0 unavailable samples in 30 days before `expire_after` was set.
- `expire_after: 900` on the three Tasmota BLE sensors makes a silent sensor `unavailable` in 15 min, and
  the thermostat then reports `current_temperature` null, which the flow treats as stuck at
  once. Basis: in 30 days of ~25000 publishes there were no gaps between 330 and 930 s.

## Scheduling

- Schedules set mode, and two zones also set targets. A mode change is what produces a demand
  step, because an off zone's target becomes 16 C.
- Pre-heat crons assert heat repeatedly through their window; presence then keeps or releases
  it. A presence-driven off during the window would last only until the next tick: three such
  flaps in 31 min stacked the PID integral from 0.28 to 3.27 and dropped the command from 58
  to 33 C with the coldest room 1.5 K short (08 Nov). The off is now ignored until 15 min after
  the last pre-heat tick.
- The bedroom radar flips about 5 times as often as the nursery one (269 against 53 raw flips
  per week); the 15 min on / 10 min off filters reduce that to about 5 mode changes per day.

## Observability decisions

- `heating_demand` is exported for every eligible zone, not only the leader, so per-zone
  series do not freeze at their last leadership.
- `is_leader` is written every cycle for every zone. Deriving the leader from
  `heating_demand` in Grafana showed a stale zone for 5 min after it stopped being eligible,
  and drew two leaders on exact ties (14 in one 13 h window).
- Controller intent (`cmd_raw`, `p_term`, `integral`, `push`) is exported before rounding and
  dedupe, because the reported TRV setpoint moves only in 1 K steps and late.
- TRV writes are a counter, so `increase()` survives restarts.
- HA's prometheus integration exports no HA labels, so the dependency list is published by
  the flow and joined in PromQL.
- Every dashboard query aggregates: a 24 h check passed while a 7-day view showed 4 copies of
  each Node-RED series after pod restarts and 2 of each OTGW series after a rename.
- The PID panel draws P, I and P+I on one axis with background bands at the output levels.
  Stacking P and I mixed colours where they had opposite signs, and a filled P+I made 0 look
  like a switch point.

## Evaluating a change

Changes were judged by a comfort score, kept out of the dashboard because its exclusions are
judgement calls. Per zone, 5 min samples, mode heat only: mean squared (room - target), where
an overshoot counts only while the boiler fires for CH and the zone's own valve is open, and a
deficit does not count while both boiler and valve are saturated. House figure is the sum over
zones, workshop excluded. Baseline 27 Oct to 06 Nov: daily sum 0.66 to 2.31, mean 1.26. Report
gas per heating degree day separately, so a change that trades comfort for gas is visible.

## Open items

- Integral wound up during solar gain (I 2.13 to 2.52 over a sunny afternoon) takes about
  2.5 h to unwind after sunset. Candidates: an upper integral limit, or shedding integral on
  an error sign change. A limit at 2.0 would act far beyond sunny afternoons (I above 2.0 for
  27 % of minutes over 15 days).
- Command rate for the Z-Wave TRVs, see above.
