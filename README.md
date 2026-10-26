# Tesla Charge Scheduler for Home Assistant

A smart Tesla charging automation for Home Assistant: it charges for your departure time, keeps an NMC/NCA battery in a healthy range while the car sits at home, and only manages charging at the places you choose (home, a chalet, work…). It handles software updates, location-aware voltage detection, per-location amps and a sleeping car.

> **Note: `scylla` is my car's name** (a Tesla Model 3 Long Range, 75 kWh, NMC). Every entity ID in these files (`sensor.scylla_…`, `number.scylla_…`, `input_number.scylla_…`, …) and the automation names and notifications use it. You need to replace `scylla` / `Scylla` with your own car's name — see *Before you start* in the Setup section.

---

## Features

- **Departure-based scheduling** — works out when to start so the car reaches the target % at your departure time. Uses real dates and times, so a charge that crosses midnight, a target set in the evening, or a DST change all plan correctly
- **The plan is stored, not waited for** — the start time is saved in a helper (`input_datetime.scylla_planned_charge_start`) and a time trigger starts the charge. The plan survives HA restarts and the dashboard shows exactly what will happen
- **Battery protection (NMC/NCA)** — with the target at 50% the battery is kept at about 40–42% (47% with a 51% target): when it falls below 40% it charges to 42%, whenever that happens. If you plug in at 40–41% and the car starts by itself, it is allowed to reach 42%. If you plug in above that and no charge is needed, the car's own charge is stopped. Turned off for LFP, see *Battery chemistry*
- **Protection before a planned charge** — if the battery falls below 40% while a higher target is waiting for its planned start, it charges to 42% now and the planned charge does the rest later
- **Charging locations** — scheduling, protection and stops only apply at the places listed in `input_text.scylla_charging_locations` (`home` and/or any zone on the HA map). Away from them (Superchargers, public chargers) the car charges normally and HA never stops it
- **Per-location default amps** — each charging location has its own default amps, set when the car arrives
- **Weekday/weekend departure defaults** — at 18:00 the departure switches to the next day's default, unless you set a custom time
- **Reset when you leave** — leaving a charging location resets the target, departure, target range and speed
- **Reset after the departure time** — if you set a trip (say 80% for 06:00) and stay home, the settings go back to the defaults once the departure time has passed **and the car has reached its target**, so it doesn't keep topping up to 80% every night. A charge that is still a few minutes short at departure time is never cut short. Can be turned off (`reset_after_departure`)
- **Location-aware voltage detection** — uses GPS + OpenStreetMap to detect the country when plugged in and sets the slow/fast voltages
- **Slow/Fast charging speed** — 15 A breaker (12 A), 20 A breaker (16 A) and Fast, each timed with its own voltage
- **Target range input** — set the range you need in km or mi. It's a minimum: the target becomes the higher of the target % and the % that range needs, so a short range never lowers your target (55% with a 30 km range stays 55%)
- **Amperage handling** — the car remembers amps per location (it may restore 32 A at home); HA puts its own value back at plug-in, at wake-up and when a charge starts. Implausible readings are ignored
- **Software update handling** — pauses a charge during an update and resumes it afterwards
- **Optional reminders** — when you plug in, a message with the planned start ("Charging to 80% starts at 01:38, ready for 06:00"), and at 21:00 a reminder if the car is not plugged in but needs a charge. Each can be turned off with a variable (`remind_plan_on_plugin`, `remind_if_not_plugged`)
- **Quiet notifications** — one message when a charge really starts (with its kind: protection or a target), a message when you change a setting and the plan changes, and a warning when something goes wrong (charge didn't start, cable not plugged in at the planned time). No message ahead of a protection charge, no message every time the battery drifts
- **Daily wake check** — once a day (13:00) a car plugged in at a charging location is woken so its battery % is fresh; a car asleep for days would otherwise never report the drop that needs a protection charge
- **Realistic charge times** — power = volts × amps × phases × efficiency − a fixed overhead for the car's own use, so slow (120 V) charges aren't under-estimated
- **Works while the car sleeps** — the last valid battery % is stored and used

---

## Files

| File | What it contains | Where it goes |
|---|---|---|
| `tesla-smart-charge-scheduler.yaml` | Scheduler automation: plans and starts charges, resets, departure defaults, voltage detection | `/config/automations/` (see Step 5) |
| `tesla-smart-charge-helper.yaml` | Helper automation: HA ↔ car sync (limit, amps, speed, range), protection stop, per-location amps, start notification, update handling, battery backup | `/config/automations/` (see Step 5) |
| `script.yaml` | `script.scylla_start_charge`: sets limit and amps, starts the charge, checks it started | `/config/script.yaml` * |
| `input_number.yaml` | Number helpers (capacity, voltages, efficiency, amps, protection caps, last known battery…) | `/config/input_number.yaml` * |
| `input_select.yaml` | Charging speed dropdown | `/config/input_select.yaml` * |
| `input_boolean.yaml` | Flags (protection run, update in progress, protection enabled) | `/config/input_boolean.yaml` * |
| `input_datetimes.yaml` | Departure time and the planned charge start | `/config/input_datetimes.yaml` * |
| `input_text.yaml` | Your charging locations | `/config/input_text.yaml` * |
| `rest.yaml` | OpenStreetMap country-code sensor (and a weather sensor). Put your own contact in the User-Agent | `/config/rest.yaml` * |
| `template.yaml` | Template sensors (at charging location, safe battery level, charge target, charge time, start time, voltage map…) | `/config/template.yaml` * |
| `ui-km.yaml` | Dashboard card (km) | Dashboard → Add card → Manual |
| `ui-mi.yaml` | Dashboard card (miles) | Dashboard → Add card → Manual |

\* **Config files:** if you don't already have that file in `/config`, copy it there and add its include from Step 1. If you already have one (for example your own `template.yaml`, or a `scripts.yaml` made by the UI), merge the entries from this file into yours instead of overwriting it.

**Copy the files, don't paste them into the HA editor.** Use the File editor or Studio Code Server add-on (upload), Samba, or SSH/`scp`. Pasting long YAML into Home Assistant's editors has cut lines and stripped indentation, and the automation editor deletes all comments when it saves.

---

## Requirements

- Home Assistant 2025.10 or later (the template sensors use `default_entity_id`, the automations the `trigger:` / `action:` syntax)
- Tesla integration (official or custom) and TeslaMate via MQTT, providing the following entities (replace `scylla` with your car name):
  - `sensor.scylla_usable_battery_level`
  - `number.scylla_charge_limit`
  - `sensor.scylla_charge_limit_soc` (the raw limit from TeslaMate)
  - `number.scylla_charging_amps`
  - `switch.scylla_charger`
  - `button.scylla_wake_up` (daily wake check; set `daily_wake_check: false` if you don't have it)
  - `binary_sensor.scylla_plugged_in` (cable connected, from TeslaMate MQTT; it updates right away, unlike the former `binary_sensor.scylla_charger` of the Tesla integration)
  - `sensor.scylla_charging_state` (TeslaMate: Charging, Complete, Stopped, Disconnected, NoPower, Starting)
  - `sensor.scylla_state` (online / asleep / offline…)
  - `sensor.scylla_charger_voltage`, `sensor.scylla_charger_phases`, `sensor.scylla_charger_actual_current`
  - `sensor.scylla_time_to_full_charge`
  - `sensor.scylla_rated_battery_range` (TeslaMate MQTT, always in km; `template.yaml` derives the miles version). The scheduler converts a target range with the *rated* range, because since Tesla firmware 2026.14.x the car no longer reports the *estimated* range ([TeslaMate issue #5298](https://github.com/teslamate-org/teslamate/issues/5298)); see *Target range*
  - `device_tracker.scylla_location` (TeslaMate MQTT: `home`, a zone name or `not_home`, plus the latitude/longitude used for voltage detection)
  - `update.scylla_software_update`
- OpenStreetMap Nominatim REST sensor (see `rest.yaml`)

`binary_sensor.scylla_charging` is no longer used: TeslaMate can leave it `on` for hours after a charge has stopped. Everything reads `sensor.scylla_charging_state` instead.

---

## Setup

### Before you start — replace `scylla` with your car's name

These files were written for a car named **Scylla**, so the entity IDs are `sensor.scylla_…`, `input_number.scylla_…` and so on. Home Assistant builds the car's entity IDs from the name of your car, so yours will differ (for example `sensor.mycar_usable_battery_level`).

1. In Home Assistant open **Developer Tools → States**, search for your car's name and note the prefix it uses.
2. Check that your car's entities exist under that prefix (see the list in *Requirements*). Names can differ slightly between Tesla integrations and TeslaMate, so adjust any suffix that doesn't match.
3. Replace the name in **every** file before you copy anything, for example from a terminal in the folder containing these files:

   ```bash
   sed -i 's/scylla/mycar/g; s/Scylla/MyCar/g' *.yaml
   ```

   On macOS, `sed` needs an empty backup suffix after `-i`:

   ```bash
   sed -i '' 's/scylla/mycar/g; s/Scylla/MyCar/g' *.yaml
   ```

   The lowercase form covers entity IDs and `unique_id`s; the capitalised form covers friendly names, the automation names and notification text.

### Step 1 — Add to `configuration.yaml`

```yaml
input_number: !include input_number.yaml
input_select: !include input_select.yaml
input_boolean: !include input_boolean.yaml
input_datetime: !include input_datetimes.yaml
input_text: !include input_text.yaml
script: !include script.yaml
template: !include template.yaml
rest: !include rest.yaml
# the two automations as files (see Step 5)
automation scylla: !include_dir_list automations/
```

If you already have any of these includes, keep yours and merge the contents of the matching file from this repo into it. The UI's own `automation: !include automations.yaml` and `script: !include scripts.yaml` lines stay as they are; if `script:` is already used by `scripts.yaml`, add the `scylla_start_charge:` block from `script.yaml` to the end of `scripts.yaml` instead of adding a second `script:` line.

### Step 2 — Add REST sensor

Add the contents of `rest.yaml` to `/config/rest.yaml` (merge if you already have one). This creates `sensor.scylla_location_country_code` using the car's GPS coordinates and OpenStreetMap.

OpenStreetMap's Nominatim usage policy asks for a User-Agent that identifies you, with a way to contact you, and may block the generic `HomeAssistant`. In `rest.yaml` replace the contact in `User-Agent: "HomeAssistant-Scylla (peter.bell@gmail.com)"` with **your own** email. While the car has no GPS position, the requests use your HA home zone's coordinates (instead of `lat=None`) and the sensors keep their previous values.

### Step 3 — Add template sensors

Add the contents of `template.yaml` to `/config/template.yaml` (merge if you already have one). This creates:
- `binary_sensor.scylla_at_charging_location` — on at one of your charging locations (3-minute delay before turning off)
- `sensor.scylla_usable_battery_level_safe` — always-available battery %; falls back to `input_number.scylla_last_known_battery` when the car sleeps
- `sensor.scylla_rated_battery_range_mi` — rated range in miles, converted from the km sensor
- `sensor.scylla_charging_amps` — the car's live amps while charging (TeslaMate `sensor.scylla_charger_actual_current`), otherwise the planned amps
- `sensor.scylla_charge_target_pct` — the target: the target % after protection (50 → 42, 51 → 47), or what the target range needs if that's higher
- `sensor.scylla_charger_1percent_increase_time` — minutes per 1% (voltage × amps × phases × efficiency − overhead)
- `sensor.scylla_charge_time` — estimated charge duration
- `sensor.scylla_charge_start_at` — the scheduler's planned start ("Charging", a time, or "Not needed")
- `sensor.ev_voltage_map` — world voltage map used by the scheduler

**Upgrading:** `sensor.scylla_charge_limit` ("Scylla Charge Limit") was removed: nothing uses it any more, and its fallback to the HA slider was what overwrote the target after a restart. After restarting, delete the leftover entity in **Settings → Devices & services → Entities** (search "Scylla Charge Limit", it shows as "no longer provided") so it doesn't clutter the list.

The file also contains a few optional extras (range in miles, return-home battery %, climate suggestions). The return-home line on the dashboard card needs the Waze Travel Time integration (`sensor.waze_scylla_to_home`).

### Step 4 — Restart Home Assistant

Required to register all new helpers, the script and template sensors.

### Step 5 — Install the two automations

**Recommended — as files (keeps the comments):** create the folder `/config/automations/` and copy `tesla-smart-charge-scheduler.yaml` and `tesla-smart-charge-helper.yaml` into it. With the `automation scylla: !include_dir_list automations/` line from Step 1, each file is one automation. They have no `id:`, so the UI shows them read-only: edit the files, then **Developer Tools → YAML → Automations** to reload.

If you previously pasted older versions into the automation editor, delete those two automations in **Settings → Automations** first, otherwise you'll have both running.

**Alternative — the automation editor:** **Settings → Automations → + Create automation → ⋮ → Edit in YAML**, paste, save. Works, but the editor deletes every comment and moves `variables:` to the bottom when it saves (that move is only cosmetic). Every step has an `alias:`, which the editor keeps and traces show.

Keep time values quoted (e.g. `'00:01:00'`, `'06:00:00'`) — some YAML parsers read an unquoted `00:01:00` as the number 60, which Home Assistant rejects as a trigger time.

### Step 6 — Set the one-time values

Hand-tuned helpers have no `initial:` (an `initial:` would reset them at every restart and wipe your settings), so a brand-new helper starts at its minimum. Once, in **Settings → Devices & services → Helpers**, set:

| Helper | Set to |
|---|---|
| Scylla battery capacity (kWh) | your usable capacity (Model 3 LR: 75) |
| Scylla charging efficiency | 94 |
| Scylla charging overhead (W) | 250 (see *Battery capacity and efficiency*) |
| Scylla rated range factor | 85 (see *Target range*; a new helper starts at 50) |
| Scylla amps sanity max | your charger's maximum (40 for a 48 A charger limited to 40 A) |
| Scylla charging locations | `home` (add zones, comma-separated: `home, Chalet`) |
| Scylla set charge limit | 50 (or your usual target) |
| EV slow / fast charging voltage | set automatically at the first plug-in |

An existing install keeps its values: removing `initial:` makes HA restore the last value.

### Step 7 — Add dashboard card

In your dashboard, add a **Manual card** and paste either `ui-km.yaml` or `ui-mi.yaml` (the card is short, pasting is fine here).

---

## Configuration

### Scheduler (`tesla-smart-charge-scheduler.yaml`)

```yaml
variables:
  default_target_pct: 50              # Target % restored when leaving a charging location
  default_departure_time_weekday: '06:00:00'
  default_departure_time_weekend: '08:00:00'
  default_charging_amps: 27           # Only a fallback for templates (per-location amps are in the helper)
  enable_battery_protection: true     # true for NMC/NCA, false for LFP (see Battery chemistry)
  protection_limit_low_pct: 42        # Cap when the target is 50%
  protection_limit_mid_pct: 47        # Cap when the target is 51%
  daily_wake_check: true              # Wake a sleeping, plugged-in car at 13:00 to refresh its battery %
  reset_after_departure: true         # Back to the defaults after the departure time, once the target is reached
  remind_plan_on_plugin: true         # Notify the planned start when you plug in
  remind_if_not_plugged: true         # 21:00 reminder if not plugged in and a charge is needed
  notification_target: notify.notify  # Notify service (same in the helper and script.yaml)
  voltage_override_slow: 0            # Set > 0 to override the detected slow voltage
  voltage_override_fast: 0            # Set > 0 to override the detected fast voltage
```

### Helper (`tesla-smart-charge-helper.yaml`)

```yaml
variables:
  location_amps:                      # Default amps per charging location, set on arrival
    home: 27
  default_amps_fallback: 27           # A location not in location_amps, and template fallback
  notification_target: notify.notify
```

Example with a second location: `location_amps: {home: 27, chalet: 16}` and `input_text.scylla_charging_locations` set to `home, Chalet`. The keys are the location names in lower case.

### Script (`script.yaml`)

Only `notification_target` — keep it equal to the other two.

### Charging locations

`input_text.scylla_charging_locations` is a comma-separated list: `home` and/or the names of zones on the HA map (**Settings → Areas, labels & zones → Zones**), exactly as `device_tracker.scylla_location` shows them when the car is there. Not case-sensitive. Empty means `home`.

At these places the scheduler plans and protects, and the helper stops charges at the protection cap. Anywhere else HA leaves charging alone, except that a target % you change in HA is sent to the car right away (and charging starts if plugged in and needed). Leaving any of these places resets the target, departure, target range and speed.

`binary_sensor.scylla_at_charging_location` waits 3 minutes before turning off, so a GPS jump or a TeslaMate reconnect doesn't count as leaving.

### Battery chemistry

Check your car's touchscreen: **Controls → Software → Additional Vehicle Information**

- **NMC/NCA** (most Model 3 LR/Performance, Model S, Model X) → `enable_battery_protection: true`
- **LFP** (Model 3 SR/Standard Range made after 2021) → `enable_battery_protection: false`

NMC/NCA cells last longer when they are not kept full, and the car won't accept a charge limit below 50%. With protection **on**, the target slider's lowest settings (50% and 51%) mean "keep the battery low": HA charges to a lower cap and turns the charger off there. With protection **off** the target is used exactly as set.

| | Protection on (NMC/NCA) | Protection off (LFP) |
|---|---|---|
| Target 50% | Keeps the battery at ~40–42%: charges to **42%** when it falls below 40% | Charges to 50% for departure |
| Target 51% | Keeps the battery at ~45–47% | Charges to 51% for departure |
| Target 52% or more | Charges to the target for departure; a 42% protection charge first if the battery falls below 40% | Charges to the target for departure |
| Car starts by itself at plug-in | Allowed up to 42% if below 42%, otherwise stopped (unless the planned start is due) | Stopped if the planned start isn't due |
| Charger stopped at the cap | Yes, by the helper (at charging locations only) | No |
| "Charging for battery protection" badge and > 90% warning on the card | Shown | Hidden |

The gap between 42% and 40% is `input_number.scylla_protection_hysteresis` (default 2). The car's reading settles about 1% lower after a charge; without the gap, 42% → 41% would restart charging every time.

The scheduler copies the setting to `input_boolean.scylla_battery_protection_enabled` every time it runs, so the helper, the template sensors and the card follow it. That helper keeps `initial: true`, so protection is never silently off after a fresh install.

For LFP, also change `default_target_pct` (50 by default): it is the target restored each time the car leaves a charging location. LFP owners typically want a higher default, such as 100.

### Slow charger breaker

On the card, set **Charging speed** to match the outlet:
- `Slow 12A (15A breaker)` — standard North American 15 A circuit
- `Slow 16A (20A breaker)` — North American 20 A circuit
- `Fast` — Level 2 charger (240 V North America, 230/400 V Europe, …); restores the location's default amps

Slow charges are timed with the slow voltage (120 V), fast ones with the fast voltage (240 V).

### Battery capacity and efficiency

Charge time = % needed × capacity ÷ power reaching the battery, where

> power (W) = volts × amps × phase factor × efficiency − overhead

- **Capacity**: Model 3 LR 75 kWh, Model 3 SR 57.5 kWh, Model Y LR 75 kWh, Model S 100 kWh. Lower it as the battery degrades
- **Phase factor**: a fast voltage of 380 V or more is treated as three-phase (× 1.732)
- **Efficiency** (`scylla_charge_efficiency`, start at **94%**): the losses that grow with the power — charger, cable, battery heat
- **Overhead** (`scylla_charge_overhead_w`, start at **250 W**): what the car uses for itself while it charges. It's a fixed amount, so it hardly matters at 6-11 kW but takes a big share of a slow charge. One pair of settings gives realistic results at every speed:

| Charge | Power in | Reaching the battery | Efficiency |
|---|---|---|---|
| 120 V × 12 A | 1.44 kW | 1.10 kW | ~77% |
| 240 V × 16 A | 3.84 kW | 3.36 kW | ~87% |
| 240 V × 27 A | 6.48 kW | 5.84 kW | ~90% |
| 240 V × 48 A | 11.5 kW | 10.6 kW | ~92% |

Tuning: if **fast** charges end early or late, change the efficiency. If only **slow** charges are late, raise the overhead (and lower it if they're early). With the overhead at 0 (a new helper's value) only the efficiency is used, like the previous version.

**Charges above ~90% finish later than estimated.** The car slows the charging down near full (the last 10% takes noticeably longer than the first 10%) and the estimate doesn't model that. A charge to 100% for a trip can end after your departure time. For those, set the departure time earlier than you really leave, or choose a target of about 90%, which also happens to be easier on an NMC/NCA battery.

### Two-car households

Entity IDs are not parameterised. Duplicate both automations, the script, the dashboard card and the helper/template entries, and replace `scylla` with your second car's prefix everywhere.

---

## How It Works

### The plan

Each time something relevant changes (battery %, target, departure, amps, speed, plug-in, arrival, 00:01, HA start), the scheduler recalculates and decides one of:

| Decision | When | What happens |
|---|---|---|
| wait | a charge is needed but not yet | the start time is stored in `input_datetime.scylla_planned_charge_start` |
| start now | the planned time has come (or is less than 2 minutes away) | `script.scylla_start_charge` sets the limit and amps and turns charging on |
| protection charge | battery below 40% | charges to 42% now (to 47% with a 51% target); a later planned charge stays planned |
| stop and wait | the car started by itself at plug-in, too early | charging is stopped, you get "Charging will start at …" |
| keep going | already charging and the target needs it | nothing, or the car's limit is corrected |
| not plugged in | the planned time has come but the cable isn't connected | one notification; plugging in starts the charge |
| nothing | target reached, or protection only and the battery is fine | the plan is cleared |

Departure: the next occurrence of the departure time (today if still ahead, else tomorrow). Start = departure − charge time, so the charge ends at the departure time.

The plan is only rewritten when it moves by a minute or more, so the 1%-at-a-time battery drift doesn't rewrite it or notify you.

### Daily flow (normal night at home)

1. **18:00** — if the departure is still a default, it switches to the next day's default (weekday 06:00 or weekend 08:00). A custom time is left alone
2. **Any time** — the battery % changes, the plan is updated silently
3. **Planned start** — charging starts; after one minute you get "⚡ Scylla charging 35% → 80% — 240 V · 27 A · …"
4. **13:00** — a sleeping car plugged in at a charging location is woken once to refresh its battery %
5. **21:00** — if the car is at a charging location, not plugged in, and needs a charge, you get a reminder (`remind_if_not_plugged`)
6. **Departure time** — if you stay home, see *After the departure time* below
7. **Leaving** — target, departure, target range and speed reset to the defaults; the car's limit is set back to 50%
8. **Arriving** — the location's default amps are set (and sent to the car if awake)

### After the departure time

You set 80% for a 06:00 departure and then don't leave. Without a reset the car would keep topping up to 80% every night. At the departure time the scheduler turns on `input_boolean.scylla_departure_passed`, and then:

- if the car is **still charging** (say 5-10 minutes short of 80%), nothing happens: the charge is never cut short. When the charge ends, or the battery % updates, and the battery is at the target, the reset happens
- once the battery is **at or above the target** and the car isn't charging, the **target %**, the **target range** and the **departure time** go back to the defaults (the car's own limit too, to 50%). With the default 50% the car is back to battery protection only
- if the target is **not reached** (for instance the car was unplugged at 60%), nothing is reset: the target stays, and the next plug-in charges for the next departure

Speed and amps are **not** reset: they describe the outlet the car is plugged into, which hasn't changed. Leaving a charging location still resets them.

Setting a new target, range or departure time starts a new trip and clears the flag, so the reset can't fire early for the new trip. Turn the whole feature off with `reset_after_departure: false`.

### Battery protection flow

1. The battery falls below 40% (cap 42% − hysteresis 2%), from vampire drain or driving, at a charging location with the cable plugged in
2. Charging starts now; the car's limit is set to 50% (its minimum) and `input_boolean.scylla_protection_charge_active` is turned on
3. After one minute: "🔋 Scylla battery protection: charging 38% → 42%"
4. The helper stops the charge at 42% (silently) and turns the flag off
5. If a higher target is planned, it starts at its planned time. If the planned time is so close that the protection charge would run into it, the charge simply continues to the target
6. Raising the target during a protection charge doesn't make it charge to the new target early: the flag keeps the stop at 42%, and the new target gets its own plan

### Plugging in

- **Below 40%** — protection charge, as above
- **40–41%** with a 50% target — if the car starts by itself it is allowed to reach 42%
- **Above 42%** with a 50% target — a charge the car starts by itself is stopped
- **With a higher target** — if the planned start isn't due, the car's own charge is stopped; if it's due, it charges

With `remind_plan_on_plugin` you get one message when you plug in and a charge is planned: "🔌 Scylla plugged in at 45%. Charging to 80% starts at Sun 01:38, ready for Sun 06:00". If the car started by itself and was stopped, that message already says so, so there is no second one.

TeslaMate publishes "cable connected" and "Charging" as separate messages, and their order isn't guaranteed. "The car started charging by itself" is detected either way: the cable-first order is handled in the `charging_started` run, the charging-first order in the `plugged_in` run (a Charging state that appears out of unknown/unavailable — a restart — is never treated as a car starting by itself). To check on your car: after a plug-in, open the scheduler's traces; in the `charging_started` (or `plugged_in`) run, `self_started` should be `true` and the decision `stop_and_wait`/`mark_protection`.

### Software update flow

1. Update starts mid-charge → `input_boolean.scylla_update_in_progress` on, charging paused, notification
2. Update finished → after 3 minutes, if that flag is on and the cable is plugged in, charging resumes, notification

An update that finishes while no charge was paused does nothing.

### Amps

1. Arriving at a charging location → the location's default amps (`location_amps` in the helper) become the planned amps and the "Fast" default; sent to the car if awake
2. Plugging in → the car often restores the amps it remembers for that place (32 A at home); within 3 minutes of plug-in HA puts its own value back
3. Car waking up at a charging location → HA's amps are sent if different
4. A charge starts → HA's amps are sent if different
5. Changed in the Tesla app → HA follows (capped to the car's maximum)
6. Away from charging locations → HA only mirrors the car's value
7. Edited on the card while the car sleeps → not sent (the car isn't woken); you get a notification, and it's sent at wake-up or when a charge starts

### Charge limit

- HA's target (`input_number.scylla_set_charge_limit`) is sent to the car when a charge starts (50% for a protection charge)
- A limit changed in the Tesla app or on the car's screen becomes the HA target. Only real changes count: a value appearing after the car wakes up or after an HA restart is ignored, so it can't overwrite your target
- Away from charging locations, a target changed in HA is sent to the car right away

### Target range

The target range is a minimum for your next departure, converted to the % it needs from the car's current range per % (rated range × `scylla_rated_range_factor` ÷ battery %). The target is the higher of that and the target % slider:

| Target % | Target range | Charges to |
|---|---|---|
| 55% | 30 km (needs ~8%) | 55% |
| 55% | 350 km (needs ~72%) | 72% |
| 50% (protection) | 30 km | protection only (40–42%) |
| 50% (protection) | 280 km (needs ~58%) | 58% at departure |

**Which range?** The *rated* range (`sensor.scylla_rated_battery_range`) × `scylla_rated_range_factor`. Tesla firmware 2026.14.x stopped sending the car's *estimated* range (TeslaMate shows it as unknown), and the rated range is higher than what you really get (cold, highway). The factor scales it down: at 85%, a rated 230 km counts as ~195 km. Set it to 100 to use the rated range as it is. Tune it from experience: if the car arrives with less range than you asked for, lower it; with much more, raise it. Too low is the safe side (it charges more).

Setting a range never changes the target % slider. A range above a full battery charges to 100% and sends a warning.

### While the car sleeps

`sensor.scylla_usable_battery_level_safe` returns the live value when valid, otherwise `input_number.scylla_last_known_battery`, which the helper updates on every valid reading and which survives restarts.

### Location-aware voltage

1. Car plugged in → REST API called to get country code from GPS
2. Country looked up in the world voltage map
3. `input_number.scylla_slow_voltage` and `input_number.scylla_fast_voltage` set (kept inside their ranges: slow 100–120 V, fast 200–400 V)
4. Override with `voltage_override_slow` / `voltage_override_fast` if needed

---

## Dashboard Card

The card shows (in order):
- **Status badge** — 🔴 Not connected / 🔋 Charging for battery protection / 🟢 Charging / ✅ Charge complete / 🟡 Connected not charging
- **Start at** — the scheduler's plan: "Charging", a time ("01:35", or "Sat 01:35" for another day), or "Not needed"
- **Charge time** — estimated duration
- **Departure time** — editable
- **Current battery %** — live from car
- **Target battery %** — the target after protection
- **Set target %** — editable (50–100%)
- **Rated range** — live from the car (TeslaMate)
- **Set target range** — the range you need; raises the target if the target % isn't enough
- **Charging amps (live / planned)** — the car's live amps while charging (TeslaMate `sensor.scylla_charger_actual_current`), otherwise the planned amps
- **Set amperage** — editable
- **Charging speed** — dropdown (Fast / Slow 12A / Slow 16A)
- **At a charging location** — whether the scheduler manages charging where the car is
- **Voltage/phase info** — slow and fast voltages with phase count, plus live voltage when charging
- **Range → %** — shown when a target range is set: the % it needs and whether it raises the target (e.g. 📍 350 km → 72% (raises the target from 55%), or 📍 30 km → 8% — your 55% target already covers it)
- **NMC/NCA warning** — 🔴 shown when target > 90% (only when battery protection is enabled)
- **Charge history** — 12-hour graph of charger state
- **Return home** — only when the car is away: distance, travel time and the battery % needed to get home (needs Waze Travel Time)

---

## Helpers Created

### `input_number.yaml`

"—" = no `initial:`, the value you set is kept across restarts (set it once, see Step 6).

| Helper | Purpose | `initial` |
|---|---|---|
| `scylla_battery_capacity_kwh` | Battery size for time calculations | — |
| `scylla_charge_efficiency` | Efficiency of the charging chain, applies to every speed (%, start at 94) | — |
| `scylla_charge_overhead_w` | Fixed power the car uses while charging (W, start at 250): matters for slow charges | — |
| `scylla_rated_range_factor` | Share of the rated range you really get (%, start at 85; 100 = the plain rated range) | — |
| `scylla_target_range_km` / `_mi` | Minimum range for the next departure (0 = none); raises the target if needed, never lowers it | — |
| `scylla_slow_voltage` | Slow voltage, 100–120 V, set from GPS | — |
| `scylla_fast_voltage` | Fast voltage, 200–400 V, set from GPS | — |
| `scylla_protection_low` | Cap with a 50% target (written by the scheduler) | 42 |
| `scylla_protection_mid` | Cap with a 51% target (written by the scheduler) | 47 |
| `scylla_protection_hysteresis` | Gap below the cap before a protection charge starts (2–10) | — |
| `scylla_set_charge_limit` | Target % (50–100) | — |
| `scylla_set_charging_amps` | Amps for the next charge (5–48) | — |
| `scylla_default_charging_amps` | Default amps of the current location (written by the helper) | — |
| `scylla_amps_sanity_max` | Amps readings above this are ignored as glitches (10–48) | — |
| `scylla_last_known_battery` | Last valid battery % (fallback while the car sleeps) | — |

### `input_select.yaml`

| Helper | Options |
|---|---|
| `scylla_charging_speed` | Fast / Slow 12A (15A breaker) / Slow 16A (20A breaker) |

### `input_boolean.yaml`

| Helper | Purpose |
|---|---|
| `scylla_protection_charge_active` | A protection charge is running: stop it at the cap |
| `scylla_departure_passed` | The departure time passed and the car hasn't reached its target yet: reset to the defaults when it does |
| `scylla_update_in_progress` | A software update paused a charge: resume it afterwards |
| `scylla_battery_protection_enabled` | Mirror of `enable_battery_protection`, written by the scheduler |

### `input_datetimes.yaml`

| Helper | Purpose |
|---|---|
| `scylla_departure_time` | Departure time (time only) |
| `scylla_planned_charge_start` | The scheduler's plan (date and time). 2000-01-01 = nothing planned. Don't edit it |

### `input_text.yaml`

| Helper | Purpose |
|---|---|
| `scylla_charging_locations` | Comma-separated charging locations (`home`, zone names) |

### `script.yaml`

| Script | Purpose |
|---|---|
| `script.scylla_start_charge` | Sets the car's limit and amps, turns charging on, notifies if it didn't start within 5 minutes |

---

## Notifications

### How notifications are sent

Every notification uses the service in `notification_target` (default `notify.notify`). When the Home Assistant Companion app is installed on a phone, the `mobile_app` integration provides the service automatically. `notify.notify` sends to **every** phone registered with the Companion app, and each phone also gets its own service named `notify.mobile_app_<device name>`.

To send to a single phone, set `notification_target: notify.mobile_app_<device name>` in the scheduler, the helper **and** `script.yaml`. To find the exact name, open **Developer Tools → Actions** and type `notify.`.

### What is sent

- 📅 **Charging planned** — only when *you* changed a setting (target, departure, amps, speed) or plugged in, and the planned start actually changed. Never for a protection charge, never because the battery drifted
- 🔌 **Plugged in, charging will start at …** — when you plug in and a charge is planned (`remind_plan_on_plugin`); also says so when the car started by itself and was stopped
- 🔌 **Not plugged in (21:00)** — the car needs a charge but the cable isn't connected (`remind_if_not_plugged`)
- 🔋 **Battery protection: charging X% → 42%** — a protection charge has run for one minute
- ⚡ **Charging X% → Y%** — a charge to a target has run for one minute, with voltage, amps, phases and the car's own time estimate
- ⚠️ **Didn't start charging** — the car didn't report Charging within 5 minutes of a start
- ⚠️ **Not plugged in** — the planned start time came and the cable isn't connected
- ⚠️ **Range not reachable** — a target range above a full battery (charges to 100%)
- Software update paused / resumed charging
- Amps reading above the sanity ceiling ignored
- Amps edited while the car sleeps (not sent)

Stopping at the protection cap and the reset after the departure time are silent.

---

## Troubleshooting

**Charge time is wrong / charge finishes too early or late**
- Check `input_number.scylla_set_charging_amps` and `input_select.scylla_charging_speed` match your charger
- Check `scylla_battery_capacity_kwh`, `scylla_charge_efficiency` and `scylla_charge_overhead_w` (Step 6; slow charges late → raise the overhead)
- Charges above ~90% always finish later than estimated (the car slows down near full), see *Battery capacity and efficiency*
- Verify `input_number.scylla_fast_voltage` / `scylla_slow_voltage` are correct for your location

**Nothing is planned / "Start at" says Not needed**
- `binary_sensor.scylla_at_charging_location` must be `on`: check `input_text.scylla_charging_locations` against the state of `device_tracker.scylla_location`
- A target of 50% (or 51%) with protection on means protection only: nothing is planned, it charges when the battery falls below 40%

**Car not starting charge**
- Check the traces: Settings → Automations → Scylla - Smart Charge Scheduler → Traces, and Settings → Scripts → Scylla - Start Charge → Traces
- Verify `switch.scylla_charger` exists and can be controlled
- The car must be plugged in — `binary_sensor.scylla_plugged_in` must be `on`

**Car charges past 42% during protection**
- The helper stops at the cap only at a charging location — check `binary_sensor.scylla_at_charging_location`
- Check the helper's traces for the `protection_check` runs as the battery rises
- TeslaMate's update interval affects how quickly the stop happens — typically within 1%

**Values reset after a restart**
- Only `scylla_protection_low`, `scylla_protection_mid` and `scylla_battery_protection_enabled` have an `initial:` (the automations rewrite them anyway). If another helper resets, check that your copy of the file has no `initial:` for it

**Unknown entities in card**
- Wait for car to wake up — template sensors fall back to helpers when car is asleep
- Check Developer Tools → States for the entity values

**Voltage not auto-detecting**
- Verify `sensor.scylla_location_country_code` exists and has a valid ISO country code
- Check the REST sensor is configured in `rest.yaml` with correct latitude/longitude entities
- Use `voltage_override_slow` / `voltage_override_fast` in the scheduler to hardcode values

**Range shows unknown, or a range target is ignored**
- Check that `sensor.scylla_rated_battery_range` has a value in km in Developer Tools → States (miles users: also `sensor.scylla_rated_battery_range_mi`). The estimated range (`est_battery_range`) is no longer used: the car stopped reporting it with firmware 2026.14.x (TeslaMate issue #5298)
- A range target seems too high or too low: adjust `scylla_rated_range_factor`
- A template sensor keeps the entity ID it was first registered with, even if you later change its `name`. If an old entry such as `sensor.scylla_estimated_range_mi` exists, rename it in Settings → Entities (or delete it and restart)

**Schedule skipped while the car sleeps or after a restart**
- Check `sensor.scylla_usable_battery_level_safe` in Developer Tools → States — it should hold the last known battery %, not 0
- If it is 0, check that `input_number.scylla_last_known_battery` has a value; it is filled on the first valid battery reading after you install the helper automation

**Amperage is wrong after getting home**
- Check `location_amps` in the helper and the helper traces for the `arrived` and `amps_changed` runs

**Automation will not save: "Expected HH:MM, HH:MM:SS…"**
- Quote time values (`'00:01:00'`, not `00:01:00`)

---

## World Voltage Reference

Voltages are auto-detected from GPS. Common values:

| Region | Slow | Fast |
|---|---|---|
| North America (CA, US, MX) | 120V | 240V |
| Europe (most countries) | 230V | 400V (three-phase) |
| Japan | 100V | 200V |
| UK | 230V | 230V |
| Australia | 230V | 230V |

The slow helper is limited to 100–120 V, so in a 230 V country it's stored as 120 V: use the Fast speed there. Override with `voltage_override_slow` and `voltage_override_fast` in the scheduler if auto-detection is wrong for your location.
