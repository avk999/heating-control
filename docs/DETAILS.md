# Design details

Why the flows are built the way they are: the plant they control, the decisions behind each
mechanism, and the alternatives that were measured and rejected. Numbers are measured on this
house between August and November 2026, each with its window length, unless marked
*documented* (vendor documentation), *simulated* (recorded data replayed through the real
node or a plant model) or *inferred*. Signs follow the table in
[USER_GUIDE.md](USER_GUIDE.md#sign-conventions); "PID integral" and "TRV integral" are kept
apart throughout.

## The plant

### Boiler

- Vaillant ecoTEC Plus, controlled over OpenTherm through an OTGW wired between the physical
  room thermostat and the boiler. The CH setpoint bounds reported through the OTGW are
  20 and 75 C.
- Over 7 days, modulation sat at its 17 % floor for 86 % of firing time, and the flame was on
  about 15 % of the time, so average demand is below the minimum output: the burner fires,
  overshoots, stops and refires, about 34 central heating (CH) ignitions per day. That cycling
  is set by the hardware; lowering flow temperature does not remove it.
- Over the same 7 days, 99 % of CH burn time had return water below 54 C, so the boiler
  condenses. Domestic hot water (DHW) burns do not, which brings the all-burns figure to 93 %.
- DHW does not disturb the room loop: over 19 DHW episodes the PID integral moved by a mean of
  -0.003.
- A new control setpoint appeared in HA's reported value within about 2 min, measured at the
  1 min Prometheus resolution, so the figure includes MQTT, HA and scrape delay. Water
  reaching the setpoint took a median 3 min on small steps and up to more than 60 min on a
  cold start.
- Over 7 days, every minute spent commanding 10 to 20 C had boiler water already hotter than
  the command, so none of it produced heat. Idle boiler water sits at a median 25 C.

### Rooms and emitters

- The living room has a large radiator with no valve, acting as the system bypass, and a
  small one with a TRV. Its temperature is set mainly by the flow temperature through the
  unvalved radiator, not by its TRV (*inferred*). The bypass is also why closing every other
  valve causes no short-cycling.
- The workshop is part of the attic and runs warm by structure: over 14 days it was above its
  HA target of 14.2 C 100 % of the time, with its mode off throughout (the flow substitutes
  16 C for an off zone, so it never asked for heat either way).
- Free-cooling time constants are 21.6 to 37.2 h per zone.

### TRVs

Five valves of three models:

| zone | device | protocol | reports |
|---|---|---|---|
| bedroom, nursery | SONOFF TRVZB | Zigbee | setpoint, temperature, running state |
| workshop | Eurotronic Spirit SPZB0001 | Zigbee | setpoint, temperature, running state, valve position |
| living room, drawing room | Danfoss LC-13 | Z-Wave | setpoint and battery only |

All are battery powered. The Z-Wave valves wake every 1800 s; a valve wakes, reports its
current value, accepts the queued command and reports the new value at the next wake, so the
report lags the command by one wake. The living-room LC-13 occasionally reports -0.0 or 2736.0
for a single sample (Z-Wave decode artefacts, 15 in 14 days); the write dedupe treats such a
sample as a mismatch and costs at most one extra write (*inferred*).

TRV sensors disagree with room sensors depending on valve state. The bedroom TRV had a
calibration of -2.4 K, set so it read correctly with the radiator hot; over a mostly idle
week it then read 3.25 K below the room sensor. The error therefore swings by about 3 K
between states, and no static offset is right in both. The Sonoff `external_temperature_input`
was tried and held stale values. So room temperature is never routed through the TRV.

### Room sensors

Workshop, nursery and drawing room use Bluetooth sensors bridged to MQTT by Tasmota,
publishing every 300 s. They sometimes stop, and HA keeps serving the last value. Bedroom and
living room use Zigbee sensors through Zigbee2MQTT; the living-room thermostat reads a
`min_max` sensor built on the Zigbee one, which is why `zone_sensors` lists two entities for
it. Over 30 days, healthy sensors held one value for up to 7.7 h overnight.

## Boiler loop

### One actuator, one leader

Flow temperature is a single house-level actuator, so it follows the room with the largest
demand (the leader); the per-room valves deal with the rest. Per-room disturbances (sun on one
facade, wind on another) belong to the valves.

Weather compensation was rejected: scalar irradiance or wind cannot say which facade is
affected, and the PID on room error already compensates load by feedback. A regression of
required water temperature on outdoor temperature looked predictive until it turned out to be
confounded by which zones were heating at which time of day. Irradiance feedforward for the
sunset cool-down was rejected too: the local loss of sun is a step when the sun goes behind a
neighbouring building, while the regional irradiance sensor declined smoothly and outdoor
temperature did not change.

### Off zones

A zone whose mode is off is given a 16 C target as a frost floor. The leader is simply the
zone with the largest demand, so at night, with every heating zone well above its target, an
off zone at 17.8 C (demand -1.8) could still be the largest and lead. Its P of +1.8 then drove
the PID integral up to its +3 limit (less heat), and the next real demand started from that
wound-up state. Over 14 days off zones led for 2.2 h, every sample with P above +0.5. Since
then an off zone at 17 C or warmer (`off_lead_margin`) cannot lead; below that it still can,
so frost protection is kept.

Mode is read from the thermostat state, not `hvac_action`. A `generic_thermostat` reports
`hvac_action: off` briefly during a mode change. The earlier code latched the off setpoint on
that transient, and the living room was held at 16 C for 110 min while 1 K cold (07 Oct).

### PID node behaviour

From the source of node-red-contrib-pid, verified by running the module with a stubbed clock:

- power = -(P + I + D) / pb + 0.5, P = room - target, clamped to 0..1. Full output is at an
  error of pb/2, not pb.
- `integral_default` is the output at zero error after a start. `integral_default` 0 starts
  the PID integral at +pb/2: zero output after every restart, about 4 h to recover from a 1 K
  deficit (*simulated*). 0.45 is used.
- Anti-windup: the PID integral changes only while |P + I| < pb/2 and is limited to +-pb/2.
  A frozen integral resumes changing as soon as the room moves P back inside that band.
- `max_interval`: a longer gap between samples holds the PID integral, so an outage does not
  integrate a huge time step.
- `smooth_factor` has no effect at `td` 0.

### Proportional band

Definitions used below: a heating event is a leader deficit above 0.3 K lasting more than
15 min; the first command is the flow command at the start of the event; the lag is the time
until the command reaches 25 C, the median idle water temperature, below which a command
produces no heat.

pb went 2.2, then 8, then 6. At 2.2 a 1.1 K deficit commanded 75 C, nearly on/off. At 8 some
events started with a command below 25 C, because the first command is set almost entirely by
the PID integral. Replaying the 14 days to 20 Oct through the real node (*simulated*, open
loop):

| pb | median PID integral | median first command | worst first command | worst lag to 25 C |
|---|---|---|---|---|
| 8 | 2.74 | 32.1 C | 24.1 C | 36 min |
| 6 | 1.74 | 36.1 C | 25.5 C | 0 |
| 4 | 0.78 | 44.1 C | 28.3 C | 0 |

A stronger P holds the operating point, so the PID integral stays small, and a small PID
integral both raises the first command and reduces overshoot from slow unwinding. pb 6 was the
smallest value at which every replayed event started above 25 C. Measured in the 38 h after
the change against the period before it: median PID integral 2.60 before, 1.53 after; every
event started above 25 C with no lag; no hunting (0.3 error sign changes per hour). Median
event duration did not change (2.27 h before, 2.20 h after).

Derivative was rejected: room sensors report in 0.1 K steps and D turns the staircase into
spikes. At td 300 the worst-case command rose from 58.0 to 72.4 C (*simulated*). Shortening
`ti` was rejected as well: 5000 s is already short against 22 to 37 h cooling constants.

### Output curve

command = 20 + 55 * output^`flow_exp`, and 10 for output exactly 0.

- The 20 C floor is the boiler's reported lower bound, and commands below it produced no heat
  (see the plant section).
- 10 C for output 0. OpenTherm gives no setpoint value a meaning: the control setpoint is 0
  to 100 C, and "no CH demand" is signalled by CHenable = 0 (*documented*, OpenTherm Protocol
  Specification v2.2, data-id 1). 10 C is the OTGW's own idle default, which "will not
  normally cause the boiler to start heating" (*documented*, OTGW stand-alone operation
  page). The flow sends no `CH` command, so it relies on 10 C being below the boiler's 20 C
  lower bound, where the burner does not fire (*observed*). Sending `CH=0` at idle would be
  the protocol-correct signal; it is not implemented.
- 0 is reserved for hand-over: CS=0 passes the physical thermostat's value through
  (*documented*, OTGW firmware command reference).
- The exponent: a linear map puts the zero-error point at the midpoint of the range, 47.5 C.
  Mean flow water temperature, burner-off time included, over hours in which rooms held their
  targets was 27 to 44 C. A lower midpoint with a linear map would cost the 75 C ceiling.
  Radiator output goes roughly as deltaT^1.3 (EN 442 convention, not measured here), so a
  convex command also makes equal output steps closer to equal heat steps.
- `flow_exp` was 1.4 (zero point 40.8 C) for four days, then 1.3 (42.3 C): at 1.4 a 0.8 K
  overnight deficit commanded only 34.6 C and recovered at 0.2 K/h.

The replay below was run before 1.3 was chosen, so 1.3 is not in it:

| 14-day replay (*simulated*) | linear | `flow_exp` 1.4 |
|---|---|---|
| median command | 39.3 C | 32.7 C |
| hours above 55 C | 24.2 | 7.2 |
| morning recovery p90 | 59.2 C | 54.2 C |

Rejected or removed:

- A cap at 60 C on an earlier 10 to 75 C range (span 65): the PID's anti-windup cannot see a
  cap outside the node, so the PID integral would wind against it.
- An earlier boost that forced 75 C when the loop "did not converge": the PID reaches 75 by
  itself at a 3 K deficit, and the PID integral kept winding while the boost held the output.
  Removed.
- An emergency override on large deficit: the largest deficit in 21 days was 2.4 K, and at
  2.0 K or more the PID already commanded a median 51.5 C.
- Skipping the PID while no room wants heat: at steady state the error crosses zero
  constantly, and the command switched between about 53 C and 10 C 301 times per day
  (*simulated*).

### PID integral correction

The PID's error is the leader's target minus its temperature. It can change for reasons
unrelated to the heating: a schedule, a presence change, a manual edit, the lead passing to
another zone, or heat from a fireplace or the sun. The PID integral built up before then no
longer applies, and at `ti` 5000 s it takes hours to unwind. After a scheduled night setback
(3 to 4 Nov) the command decayed from 59 to 44 C over three hours while no zone was below
target.

The leader function sends `msg.integral` to the PID (node-red-contrib-pid 2.1.0) in two cases:

1. **Configuration change of the previous leader.** Configuration is each zone's mode and
   target. The trigger needs both: the zone that led on the previous 15 s cycle changed its
   own configuration, and the leader's error (target - room) changed by 0.8 K or more between
   the two cycles. The correction subtracts that step from the PID integral, in either
   direction. Over 7 days, 175 configuration changes gave 23 error drops, and no drop fell
   between 0.60 and 0.80 K, so the threshold sits in that gap. A manual 1 K edit shows up as
   0.6 K or less: lowering a target usually passes the lead to another zone, and the step is
   then the difference between the two zones' errors. Rises count too: with the PID integral
   at its +3 limit overnight, raising the bedroom target took two hours of unwinding before
   the command responded (17 Nov). Why the test is on the previous leader's own change: see
   the 08 Nov incident under [Scheduling](#scheduling).
2. **External heat.** Any zone rose 2 K or more over the last 30 min while the leader is above
   its target. The rise is added to the PID integral (less heat). Over 21 days, heating alone
   never moved a zone more than 1.0 K in 30 min. A replay of one afternoon fired on sun
   through the skylights (+2.00 K) and on the fireplace (+2.60 K). A latch gives one
   correction per episode and clears when the zone's rise falls below 1 K. A slow fire stays
   under the threshold and is missed.

The correction cancels the step; it does not zero the PID integral. Zeroing means output 0.5,
which raised the hours spent commanding heat while no zone was below target from 8.4 to
11.1 per day (*simulated*), and once took the command from 27.9 to 53.9 C on a step that
asked for less heat.

An earlier version of this correction triggered on any error step of 0.5 K, without the
configuration test. 7 of its 8 fires in one day were the lead passing between rooms: the
difference between the leader's and the next zone's error exceeds 0.5 K in 40.6 % of samples,
so no threshold on the step alone can separate the two cases.

This is not a derivative term: it fires once on a discrete event and changes stored state.

### Valve authority before water temperature

A proposed rule raised water temperature only when the leading zone's TRV had no command
headroom left (`cmd_headroom`, see the user guide). On 03 Nov headroom read 5.4 to 8.0 K in
every zone, an earlier 7-day check of the commands found a minimum of 2 to 5 K, and the largest
leader deficit to that date was 1.6 K. The rule would never have acted and was not built.

## TRV loop

### Control law

PI on the room sensor; the output is the TRV setpoint. One law for all five valves, with no
per-model code. Each TRV keeps its own internal loop, and the TRV integral absorbs its sensor
bias. Over 14 days on a simulated plant with disturbances:

| law | writes per day | valve travel, % of stroke per day | rms error |
|---|---|---|---|
| winter 2025-26 law, room error amplified 5 times | 411 | 26314 | 0.099 K |
| binary 30/16 | 132 | 13143 | 0.136 K |
| PI, kp 2, ti 1800 s, step 1 K | 20 | 722 | 0.283 K |

Travel is what costs battery and wear. The PI accepts 0.18 K more rms error for 36 times less
travel. Binary control slams each valve fully open or shut; while a cold zone keeps the water
hot, a warm zone's valve overshoots each time it opens and cycles constantly (*inferred*
mechanism). The simulation did not model the Z-Wave re-sends described below, and whether its
write counts are per valve or for the house was not recorded.

- **Soft deadband.** The 0.2 K deadband is subtracted from the error. A hard deadband makes the
  command jump by kp times 0.2 at its edge, and every crossing costs a write.
- **Write step 1 K.** A 0.5 K step gave 634 writes per day (*simulated*).
- **Dedupe against the reported setpoint.** Comparing against the last value sent was not
  self-correcting: the living-room valve reverted from 16 to 18.5 six minutes after a write,
  and no rewrite followed for 9.9 h (21 Oct). HA is optimistic after a service call and
  corrects the entity when the device reports. A TRV whose entity reports no setpoint
  (unavailable) is always written.
- **TRV integral lower limit 0.** We have heating and no cooling, so a command below target
  is dead range. A room 0.4 to 0.7 K above target for 8 h wound the TRV integral to -1.05, and
  the next morning's command of 19.55 against a 20.0 target could not open the valve. A -0.5
  limit still left a room 0.1 to 0.2 K short overnight.
- **TRV integral upper limit 7** (was 5). The TRV head reads high when its own radiator is hot,
  so the valve throttles itself while the room is short. Over 14 days only bedroom (5.2 % of
  samples) and living room (3.1 %) sat at the old limit of 5; `cmd_raw` peaked at 27.9 C,
  inside the 30 C ceiling.
- **Decay while off.** A zone goes off for a scheduled setback or for a presence dropout of a
  still occupant, and the node cannot tell them apart. Zeroing on off threw away a TRV
  integral of 4.37 over a 38 min radar dropout, and the room sat 0.3 K short until morning.
  Over 30 days bedroom had 34 off periods under 2 h; a 2 h time constant keeps 73 % over
  38 min and almost nothing overnight.
- **Enable check next to the PI.** With the enable check four nodes upstream behind a queue,
  the PI ran 38 s after control was switched off and closed five valves that had just been
  opened (17 Nov).
- **Open all valves on disable.** A valve left at its last command is still controlled, by a
  stale command; for an off zone that means shut. Opening serves maintenance, pump circulation
  with heating off, and hand-over to the physical thermostat.

### Accepted limitations

- The living-room TRV command changes often, but its radiator contributes little; no special
  case was added.
- The Z-Wave valves apply whatever command is queued at their 30 min wake while the PI
  recomputes every 2 min. Slowing the command rate for those zones is an open tuning item.
- The Spirit in `system_mode: auto` rewrote its own setpoint once (16 to 24 C, valve open for
  99.5 h, with no write from Node-RED). The workshop zone is off, so it was left.

## Sensor health

- Replaying 30 days of readings, a 3 h no-change rule would have fired 19 times, all false
  positives, nearly all overnight (*simulated*). So no-change is a 12 h backstop.
- HA `last_updated` freezes together with the value and is not a liveness signal; HA
  availability showed 0 unavailable samples in 30 days before `expire_after` was set.
- `expire_after: 900` on the three Tasmota BLE sensors makes a silent sensor `unavailable` in
  15 min, and the thermostat then reports `current_temperature` null, which the flow treats as
  stuck at once. Basis: in 30 days of about 25000 publishes there were no gaps between 330 and
  930 s.

## Scheduling

- Schedules set mode, and two zones also set targets. A mode change is what produces a demand
  step, because an off zone's target becomes 16 C.
- Pre-heat crons assert heat repeatedly through their window; presence then keeps or releases
  it.
- **The 08 Nov incident.** Between 19:27 and 19:58, during the bedroom and nursery pre-heat,
  presence-off events switched a zone off and the next cron tick switched it back, three times
  in 31 min. Meanwhile nursery flickered into the lead for two cycles at a time while bedroom
  was the real leader. The PID integral correction then tested for a configuration change in
  any zone, so each lead swap that coincided with a flap counted as a configuration change.
  Three corrections took the PID integral from 0.28 to 3.27 and the command from 58 to 33 C
  with the coldest room 1.5 K short. Two fixes followed: the correction tests only the previous
  leader's own configuration, and a presence-driven off is ignored until 15 min after the
  zone's last pre-heat tick.
- The bedroom radar flips about 5 times as often as the nursery one (269 against 53 raw flips
  per week); the 15 min on / 10 min off delays reduce that to about 5 mode changes per day.

## Observability decisions

- `heating_demand` is exported for every zone allowed to lead, not only the leader, so per-zone
  series do not freeze at their last leadership.
- `is_leader` is written every cycle for every zone. Deriving the leader from `heating_demand`
  in Grafana showed a stale zone for 5 min after it stopped being eligible, and drew two
  leaders on exact ties (14 in one 13 h window).
- Controller intent (`cmd_raw`, `p_term`, TRV integral, `push`) is exported before rounding and
  dedupe, because the reported TRV setpoint moves only in 1 K steps and late.
- TRV writes are a counter, so `increase()` survives restarts.
- HA's prometheus integration exports no HA labels, so the dependency list is published by the
  flow and joined in PromQL.
- Every dashboard query aggregates: a 24 h check passed while a 7-day view showed 4 copies of
  each Node-RED series after restarts and 2 of each OTGW series after a rename.
- The PID panel draws P, I and P+I on one axis with background bands at the output levels.
  Stacking P and I mixed colours where they had opposite signs, and a filled P+I made 0 look
  like a switch point.

## Alert conditions

Replayed over the 30 days to 26 Nov 2026.

- A deficit alone is not a usable alert: above 1 K for 60 min it fired on scheduled evening
  and morning ramps, and still in 62 five-minute samples with a 3 h hold. Slow recoveries
  are a comfort question, not a fault. `HeatingCannotHeat` therefore also requires that the
  controller has no authority left; that combination did not occur.
- An overshoot alone, or with the boiler firing, fired in every zone for hundreds of samples:
  rooms gain heat passively while another zone drives the boiler. Requiring the zone's own
  TRV to report `heating` left 3 episodes, all real: twice the bedroom TRV integral (about
  4.9 K) held the valve open for 1.2 to 2.6 h after the night target dropped, once a
  Eurotronic Spirit in `system_mode: auto`. The first two show that the TRV integral is not
  corrected when the target drops, unlike the PID integral.
- `climate.boiler_direct` is an optimistic MQTT entity without an availability topic, so it
  stays available when the OTGW stops publishing (*inferred* from the HA MQTT
  configuration), and `HeatingDependencyUnavailable` cannot detect a dead gateway.
  `OTGWStale` uses the return temperature instead: its longest flat run was 0.7 h, and a
  60 min window never occurred.

## Evaluating a change

Changes were judged by a comfort score, kept out of the dashboard because its exclusions are
judgement calls, and its script is not published. Per zone, 5 min samples, mode heat only: the
mean of (room - target) squared, in K squared. A sample above target counts only while the
boiler fires for CH and the zone's own TRV command is above 16 C. A sample below target does
not count while the control setpoint is 70 C or more and the zone's TRV command is 29.5 C or
more (both at their limits). The house figure is the daily sum of the per-zone values, workshop
excluded. Baseline 27 Oct to 06 Nov: 0.66 to 2.31 K squared, mean 1.26. Gas per heating degree
day is reported separately, so a change that trades comfort for gas is visible.

## Open items

- A PID integral wound up during solar gain (2.13 to 2.52 over a sunny afternoon) takes about
  2.5 h to unwind after sunset. Candidates: an upper limit on the PID integral, or shedding it
  on an error sign change. A limit at 2.0 would act far beyond sunny afternoons (above 2.0 for
  27 % of minutes over 15 days).
- Command rate for the Z-Wave TRVs, see above.
