# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A single HeishaMon rules script (`rules.txt`) that automates a Panasonic Aquarea heat pump via the [HeishaMon](https://github.com/heishamon/HeishaMon) firmware. There is no build, test, or lint tooling — the file is uploaded to HeishaMon through its web UI (`/rules`), and the firmware's embedded rules engine parses and runs it. Validation happens at upload time on the device; iterate by editing locally, uploading, and watching live MQTT values / the HeishaMon log.

## Rules-engine language (essential to know before editing)

The script is written in HeishaMon's event-driven DSL. The syntax looks shell-ish but is its own grammar — pay attention to the sigils, because they determine variable lifetime and source:

- `@Name` — **MQTT topic** exposed by HeishaMon. Read-only sensor topics (`@Outside_Temp`, `@DHW_Temp`, `@Heatpump_State`, `@Operating_Mode_State`, `@ThreeWay_Valve_State`, `@Defrosting_State`, `@Main_Outlet_Temp`, `@Main_Target_Temp`, `@Z1_Heat_Request_Temp`, …) and write-capable command topics (`@SetHeatpump`, `@SetOperationMode`, `@SetPump`, `@SetMaxPumpDuty`, `@SetDHWTemp`, `@SetCurves`, `@SetZ1HeatRequestTemperature`, `@SetZ1CoolRequestTemperature`, `@SetQuietMode`, `@SetForceDefrost`). Assigning to a `@Set…` topic publishes a command to the heat pump.
- `#name` — **persistent global** variable, survives across rule firings (and, with HeishaMon's flash persistence, across reboots).
- `$name` — **local** variable, scoped to a single `on … end` block.
- `%name` — **system** variable supplied by the engine (`%hour`, `%minute`).

Events / blocks:

- `on System#Boot then … end` — runs once at firmware start. Used here to seed all `#…` defaults and arm the periodic timer.
- `on timer=<id> then … end` — fires when a timer set by `setTimer(<id>, <seconds>)` elapses. Timer `10` is rearmed each tick to act as a 30-second clock; timers `20`, `30`, `31` are one-shot phases of the DHW/recovery state machine.
- `on @<Topic> then … end` — fires when that MQTT topic's value changes.
- `on <name> then … end` — user-defined event, invoked by calling `<name>()` from another block (used here for `throttle` and `dhw_start`).

Built-ins used: `setTimer(id, seconds)`, `concat(…)` (string concat — used to build the JSON payload for `@SetCurves`), `round()`, `floor()`. The `%` operator is integer modulo.

A subtle parity convention to keep in mind when reading: **`@Operating_Mode_State` even == heating, odd == cooling**, and `+ 4` toggles the DHW-active variant — that's why the code does `@Operating_Mode_State % 2 == 0` to branch heat vs. cool and `#OpModeBeforeDHW + 4` to enter DHW mode while preserving the underlying heat/cool selection.

## What the rules actually do (high-level)

The script is one coordinated controller — editing one block in isolation will usually break another. The pieces:

1. **Boot (`System#Boot`)** — seeds `#…` state, disables the heat pump, sets pump-duty caps (`#maxPumpDutyQuiet = 112`, `#maxPumpDutyPower = 170`), and starts the 30-second master timer (id `10`).

2. **Master clock (`timer=10`)** — the central scheduler. Re-arms itself every 30 s and checks `%hour`/`%minute` for time-of-day actions:
   - **08:00** — if outside is cold, raise pump duty to the "power" cap; if heat pump was paused overnight in heating mode, restart it; clear nighttime quiet floor.
   - **09:00** — fallback restart when it's warm and the system was paused.
   - **13:00** and **anytime `@DHW_Temp < 18`** — trigger the DHW heat-up via `dhw_start()`.
   - **18:00** — decide whether to pause overnight (sets `#offAtNight = 2` and turns the unit off in cooling, or marks it for off in heating).
   - **19:00** — drop pump duty back to the quiet cap; finish the off-at-night transition; raise quiet-mode floor (`#QuietMinimumValue = #QuietAtNight`).

3. **DHW cycle** — driven by the three-way valve state and a small state machine:
   - `on @ThreeWay_Valve_State`: when the valve flips to DHW (`== 1`), the rules **snapshot** the current curve targets and op mode into `#targetLowBeforeDHW` / `#targetHighBeforeDHW` / `#OpModeBeforeDHW`, raise pump duty to the power cap, **compute a dynamic DHW target** by linearly interpolating between 43 °C (at −5 °C outside) and 50 °C (at 30 °C outside), then write temporarily-narrow curves so the heat side doesn't fight DHW. When the valve flips back (`== 0`), curves and op mode are restored — but only if no defrost is in progress (`#DefrostWhileDHW == 1`).
   - `on @Defrosting_State`: if a defrost starts *during* DHW, restore the pre-DHW curves immediately and mark `#DefrostWhileDHW = 2`. When the defrost ends, return to DHW mode (`#OpModeBeforeDHW + 4`).
   - `on @DHW_Power_Consumption`: when power draw drops to 0 while in DHW mode and the target is reached, restore the pre-DHW op mode and on/off state.
   - `dhw_start` + `timer=30` + `timer=31`: cold-weather (`@Outside_Temp < 5`) DHW runs are preceded by a forced defrost (`@SetForceDefrost = 1`) and a 540 s wait; warm-weather DHW runs start after 5 s. `timer=30` captures the pre-DHW state and enters DHW op mode. `timer=31` clears the `#dhwStarted` re-entry guard 35 s later.
   - `timer=20`: 240 s post-DHW recovery — re-asserts the previous heatpump state, restores curves and pump duty, and releases the supplementary pump.

4. **Adaptive request temperature (`throttle`)** — fired (via `throttle()`) whenever `@Main_Target_Temp`, `@Main_Outlet_Temp`, or `@Main_Inlet_Temp` changes. Computes an outlet-vs-target delta, optionally biased by `#roomDelta` when it's cold and the room is already warm, then sets `@SetZ1HeatRequestTemperature` (or the cool equivalent) to nudge the heat curve up or down. Also sets `@SetQuietMode` based on how hard the unit is working — small delta → quiet floor, large delta → quiet level 3.

5. **Smart pause (`on @Heat_Power_Consumption`)** — if the unit is running but drawing 0 W in mild weather (`@Outside_Temp > 10`) with a positive heat request, pause it (`#offAtNight = 2`, `@SetHeatpump = 0`). The morning 08:00 / 09:00 logic in `timer=10` is what un-pauses it.

## Editing guidance

- **Tunables live at the top of `System#Boot`.** Things like `#maxPumpDutyQuiet`, `#maxPumpDutyPower`, `#QuietAtNight`, and the curve seeds are the intended knobs — change those rather than rewriting downstream blocks.
- **DHW target curve** (the 43→50 °C interpolation) is inlined in `on @ThreeWay_Valve_State`; the four boundary constants (`$minDHWTarget`, `$minOutside`, `$maxDHWTarget`, `$maxOutside`) are local `$` vars set on each valve transition.
- **Don't reuse timer IDs.** `10`, `20`, `30`, `31` are taken and carry distinct semantics; pick a new ID for new timers.
- **Pause guards.** Many blocks gate on `@Heatpump_State == 1` so they don't fight a manually-stopped unit. Preserve that pattern.
- **The `#dhwStarted` flag** prevents double-firing of `dhw_start` from the two trigger paths in `timer=10` (the 13:00 schedule and the low-tank fallback). Don't remove the guard without replacing it.
- **`@SetCurves` payload** is built with `concat(…)` into a JSON-ish string the firmware parses — match the exact shape `{zone1:{heat:{target:{high:<n>,low:<n>}}}}` when editing.
