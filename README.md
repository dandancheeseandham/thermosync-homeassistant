# ThermoSync — Smart Hot Water Optimizer

A Home Assistant blueprint that treats your hot water tank as a thermal
battery. Pre-heats during the cheapest electricity windows and coasts
through expensive periods.

## Safety-First Defaults

Out of the box, ThermoSync will **not heat water above 50 °C**. This
protects against scalding — hot water over 50 °C can cause serious
burns in seconds, especially to children, elderly people, and anyone
with reduced mobility or sensation.

**Legionella protection is disabled by default.** Enabling it heats the
tank above 60 °C, which is dangerous without thermostatic mixing valves
(TMVs). Read the legionella section carefully before enabling.

**Sensor failures are handled safely.** If your electricity rate sensor
goes down, ThermoSync defaults to a high rate (999 p/kWh) rather than
zero — preventing accidental heating at full grid price during API
outages.

## Features

- **Planned windows** via TargetTimeframes (cheapest slots in each time band)
- **Cheap-rate extension** — heats outside planned windows when rates drop
- **Solar diversion** — excess solar with W/kW auto-detection and forecast smoothing
- **Negative-rate opportunism** — uses free electricity safely
- **Legionella protection** — three-phase min/max/price scheduling
- **Effective rate calculation** — solar-adjusted pricing for smart deferral
- **Configurable demand profiles** — tune for household size and schedule
- **Peak avoidance** — blocks heating during expensive hours (midnight-wrapping safe)
- **Manual boost** — override for "I need hot water NOW"
- **Relay protection** — hysteresis deadband and minimum dwell times
- **Stale data fallback** — degrades gracefully when sensors fail
- **HA restart safe** — evaluates immediately on startup after sensor init
- **Multiple heater types** — climate, water_heater, switch, input_boolean
- **Two control modes** — temperature-setting or on/off switching

---

## Quick Start (15 minutes)

### Step 1 — Install the blueprint

[![Open your Home Assistant instance and import this blueprint.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=[https://github.com/ha-community/thermosync/blob/main/blueprints/automation/thermosync.yaml](https://github.com/dandancheeseandham/thermosync-homeassistant/blob/main/blueprint/automation/thermosync.yaml))

Or manually: copy `thermosync.yaml` into
`config/blueprints/automation/thermosync/`

### Step 2 — Set up TargetTimeframes (recommended)

This is the most important prerequisite — it finds the cheapest
electricity slots for each heating window. See the **Setting Up
TargetTimeframes** section below.

### Step 3 — Create the automation

1. Go to **Settings → Automations & Scenes → Create Automation**
2. Click **Use Blueprint** and select **ThermoSync**
3. Fill in the **Core Configuration**:

| Field                    | What to enter                              |
|--------------------------|--------------------------------------------|
| Tank Temperature Sensor  | Your tank temp entity                      |
| Electricity Rate Sensor  | Your tariff rate entity                    |
| Rate → pence multiplier  | 100 for Octopus Energy, 1 if already p/kWh |
| Hot Water Heater Entity  | Your heating control entity                |
| Control Mode             | Match your system (see below)              |

4. Under **Schedule Windows**, select the three TargetTimeframes binary
   sensors you created
5. Save — ThermoSync is now running with safe defaults (max 50 °C)

### Step 4 — Optional: enable legionella protection

Only do this if you have TMVs on all hot water outlets. See the
**Legionella Protection** section below.

### Step 5 — Optional: create helpers

| Helper | Purpose | How to create |
|--------|---------|---------------|
| Manual Boost | "Hot water NOW" button | Settings → Helpers → Toggle |
| Enable/Disable | Pause ThermoSync | Settings → Helpers → Toggle |

Select these in the **Resilience & Safety** section of the blueprint.

---

## Setting Up TargetTimeframes

### Install the integration

1. Open **HACS** (Home Assistant Community Store)
2. Go to **Integrations → Explore & Download Repositories**
3. Search for **"Target Timeframes"** by BottlecapDave
4. Click **Download** and restart Home Assistant
5. Go to **Settings → Devices & Services → Add Integration**
6. Search for **"Target Timeframes"** and add it

Repository: https://github.com/BottlecapDave/HomeAssistant-TargetTimeframes

### Create the three ThermoSync windows

You need **three separate target entities**. Each finds the cheapest
slots within a time band.

**IMPORTANT — size for winter.** Cold mains water in winter (6–10 °C)
needs significantly more heating time than summer (15–18 °C). A window
that works in July may fail in January. Set hours for the worst case.
In summer, the tank simply reaches target sooner and ThermoSync stops
heating — no energy is wasted.

Go to **Settings → Devices & Services → Target Timeframes** → Configure.

#### Window 1: Overnight (morning shower pre-heat)

| Setting          | Value                          |
|------------------|--------------------------------|
| Name             | `ThermoSync Overnight`         |
| Hours            | `2.0` (winter-sized)           |
| Type             | `Intermittent`                 |
| Start time       | `22:00`                        |
| End time         | `07:00`                        |
| Offset           | `-00:00:30`                    |
| Rate sensor      | Your electricity rate entity   |

Creates `binary_sensor.target_thermosync_overnight`.

#### Window 2: Midday (afternoon/evening top-up)

| Setting          | Value                          |
|------------------|--------------------------------|
| Name             | `ThermoSync Midday`            |
| Hours            | `1.5` (winter-sized)           |
| Type             | `Intermittent`                 |
| Start time       | `09:00`                        |
| End time         | `15:30`                        |
| Offset           | `-00:00:30`                    |
| Rate sensor      | Your electricity rate entity   |

Creates `binary_sensor.target_thermosync_midday`.

#### Window 3: Evening (last top-up)

| Setting          | Value                          |
|------------------|--------------------------------|
| Name             | `ThermoSync Evening`           |
| Hours            | `1.5` (winter-sized)           |
| Type             | `Intermittent`                 |
| Start time       | `19:00`                        |
| End time         | `22:00`                        |
| Offset           | `-00:00:30`                    |
| Rate sensor      | Your electricity rate entity   |

Creates `binary_sensor.target_thermosync_evening`.

### Adjusting window hours

These times assume heating a full tank from cold. If your immersion
only tops up by 10–15 °C, you can reduce the hours.

| Heater power | 100 L | 150 L | 200 L | 300 L |
|-------------|-------|-------|-------|-------|
| 1.5 kW      | 2.0 h | 2.5 h | 3.5 h | 5.0 h |
| 3 kW        | 1.0 h | 1.5 h | 2.0 h | 2.5 h |
| 6 kW        | 0.5 h | 0.75 h| 1.0 h | 1.5 h |

Times shown are for winter worst-case (mains at ~8 °C, heating to
~55 °C). Summer heating is roughly 30–40% faster.

### Intermittent vs Continuous

Use **Intermittent** for ThermoSync — it finds the cheapest individual
half-hour slots (they don't need to be consecutive). Your tank's
thermostat handles on/off cycling naturally.

### Without TargetTimeframes

ThermoSync works without it — leave the schedule window fields empty.
The system will rely on cheap-rate extension, solar diversion, and
negative-rate opportunism. This is less optimal because it can't plan
ahead.

**Important:** if you don't configure TargetTimeframes AND your sensors
go stale (API outage, network failure), ThermoSync has no fallback
windows. The degraded-mode logic ("heat during planned windows at base
temp") requires windows to exist. Without them, the only remaining
trigger is depletion recovery — the system will completely stop heating
until sensors recover or the tank empties. If this concerns you,
configure at least the overnight TargetTimeframes window as a safety
net.

---

## Control Mode — Which One?

| Your setup                                          | Mode               |
|-----------------------------------------------------|--------------------|
| Smart controller / heat pump with climate entity     | **Set temperature** |
| System that heats whenever tank is below setpoint    | **Set temperature** |
| Immersion relay (switch) with physical thermostat    | **On / Off**        |
| Dedicated hot water controller with on/off commands  | **On / Off**        |

**Set temperature** mode writes a high target when heating is needed and
a low idle target (default 10 °C) when it isn't. The heater entity
stays "on" and heats autonomously to whatever setpoint ThermoSync
writes. The target is re-enforced every 5 minutes, correcting any
manual thermostat drift within one evaluation cycle.

**On / Off** mode turns the entity on to heat and off to stop. For
climate and water_heater entities, it also writes the target temperature
before turning on.

**Both modes** now always call `turn_on` on climate and water_heater
entities when heating, ensuring the device is actually running — not
just receiving a setpoint while physically switched off.

---

## Sensor Requirements & Gotchas

### Sensor refresh cadence

ThermoSync's stale-data detection checks each sensor's `last_updated`
timestamp. If the timestamp is older than `stale_data_minutes` (default
30), the system enters degraded mode.

**Template sensors** (e.g. a fixed-tariff sensor that outputs a
constant value based on time of day) may only update when their input
changes. If the rate never changes overnight, the sensor's
`last_updated` can drift past the stale threshold. Fix this by adding
a periodic update trigger or shortening the template's `scan_interval`.

**Physical sensors** (e.g. slow-polling DS18B20) that update
infrequently should have `stale_data_minutes` set above their longest
expected update gap.

### Tank sensor placement

ThermoSync uses a single temperature sensor. In a stratified tank,
placement matters:

- **Middle of the tank** — best general-purpose position. Reads the
  boundary between hot and cold layers during draw-off, giving the
  most useful "how much hot water is left" signal.
- **Near the top** — reads close to the output temperature but gives
  late warning of depletion (sensor stays hot until the tank is nearly
  empty).
- **Near the element** — reads the hottest point. May show a successful
  legionella cycle temperature while the rest of the tank hasn't
  reached it. **Not recommended** — can falsely confirm pasteurisation.

For legionella protection, position the sensor at **mid-tank or higher**
to ensure the recorded temperature reflects the bulk of the stored
water, not just the zone nearest the heating element.

### Device target limits

Some `climate` or `water_heater` integrations have min/max temperature
bounds enforced by the device or integration. A 10 °C idle setpoint may
be rejected if the device's minimum is 20 °C, and a 70 °C legionella
target may be rejected if the device's maximum is 65 °C. Check your
entity's attributes in Developer Tools → States for `min_temp` and
`max_temp` and adjust ThermoSync's settings accordingly.

---

## Understanding the Temperature Settings

```
┌────────────────────────────────────────────────────────────────┐
│ NORMAL OPERATION                                               │
│                                                                │
│   Smart target = base_temp + demand_boost + price_boost        │
│   Clamped to: [base_temp ... max_temperature]                  │
│   Default: 45 + 5 overnight boost = 50 °C (safe cap)          │
│   Used when: planned window active or rate is cheap            │
│                                                                │
│   NOTE: max_temperature caps the SMART TARGET only.            │
│   Negative-rate, solar, manual boost, and legionella each      │
│   have their own separate temperature settings that can        │
│   exceed max_temperature.                                      │
├────────────────────────────────────────────────────────────────┤
│ SPECIAL CONDITIONS                                             │
│                                                                │
│   Negative rate  →  negative_rate_temp (default 50 °C)         │
│                     Requires rate_valid AND not stale           │
│                     (API outage → rate defaults to 999p,        │
│                      NOT zero — no false negative-rate trigger) │
│                                                                │
│   Tank depleted  →  emergency_temp (default 50 °C)             │
│     ┌────────────────────────────────────────────────────┐     │
│     │ Trigger: sensor drops below depletion_threshold    │     │
│     │          (default 23 °C ≈ "one shower left")       │     │
│     │ Action:  heat to emergency_temp immediately        │     │
│     └────────────────────────────────────────────────────┘     │
│                                                                │
│   Manual boost   →  max(max_temp, neg_temp)                    │
│                     Overrides peak blocks and price logic       │
│                     Requires valid tank sensor to operate       │
│                     Auto-disables when target reached           │
│                                                                │
│   Legionella due →  legionella_temp (default 65 °C)            │
│                     DISABLED unless tracker helper is set       │
│                                                                │
│   Data degraded  →  base_temp during planned windows only      │
│                     No price logic, no solar, no neg-rate       │
├────────────────────────────────────────────────────────────────┤
│ NOT HEATING                                                    │
│                                                                │
│   Hysteresis: won't re-engage until tank drops 2 °C below      │
│   target (prevents relay chatter on small fluctuations)        │
│                                                                │
│   Dwell time: heater must stay in same state for 5 min before  │
│   ThermoSync will change it (protects physical relays)         │
│                                                                │
│   Set-temp mode  →  idle_temperature (default 10 °C)           │
│   On/off mode    →  entity turned off                          │
└────────────────────────────────────────────────────────────────┘
```

**Depletion threshold vs emergency temperature** — these are a pair.
The threshold (23 °C) is the *trigger*: "sensor reads 23 °C, about one
shower left." The emergency temperature (50 °C) is the *action*: "heat
to 50 °C to recover." Set the threshold by running hot water until it
starts going tepid, then note your sensor reading and use that value.

---

## Legionella Protection — Read Before Enabling

### The trade-off

Legionella bacteria grow in stored water between 20–45 °C. Periodic
heating to 60 °C+ kills them. However, **water at 60 °C causes
full-thickness burns in under 5 seconds**, and at 70 °C almost
instantaneously.

**Without TMVs on all outlets: DO NOT ENABLE.** A scald is a more
immediate and serious risk than legionella for most domestic situations.

**With TMVs on all outlets:** enabling is recommended. TMVs blend hot
water with cold before it reaches the tap, limiting delivered
temperature to 43–48 °C regardless of tank temperature.

### Who is at highest risk of scalding?

Children (thinner skin), elderly people (reduced sensation), anyone with
reduced mobility, and people with diabetes or peripheral neuropathy.

### Three-phase scheduling

ThermoSync uses an intelligent three-phase approach rather than a fixed
interval:

| Phase | Condition | Behaviour |
|-------|-----------|-----------|
| **Ignore** | Less than `min_days` (default 5) since last cycle | Don't schedule. If the tank reaches 60 °C+ naturally (e.g. via negative-rate heating), it's recorded but doesn't reset the active scheduling clock. |
| **Opportunistic** | Between `min_days` and `max_days` (5–10) | Schedule a cycle when the rate drops below `legionella_price_threshold` (default 8 p). This finds a cheap window rather than heating at any price. |
| **Forced** | More than `max_days` (default 10) without a cycle | Force a pasteurisation during the next overnight window regardless of price. Safety backstop. |

### Physical thermostat limits

Many immersion heaters have a mechanical high-limit thermostat (often
factory-set to 60–65 °C). If your physical thermostat cuts power before
ThermoSync's legionella target, the tracker would never update and
legionella scheduling would loop forever.

ThermoSync handles this with a two-tier acceptance:

1. **Within 3 °C of target** — if `leg_temp` is 65 °C and the tank
   reaches 62 °C, the cycle is recorded
2. **Above 60 °C while actively trying** — if the tank reaches 60 °C
   during a legionella attempt, the cycle is recorded (60 °C meets the
   HSE minimum for legionella kill)

**If your physical thermostat limits to e.g. 63 °C, set `leg_temp` to
63 °C** rather than fighting the physical limit.

### Sensor placement for legionella

ThermoSync relies on a single tank sensor to confirm pasteurisation.
If the sensor is positioned **near the heating element**, it may
register 60 °C+ while the rest of the tank volume hasn't reached that
temperature — resulting in a false confirmation that doesn't actually
kill bacteria throughout the tank.

**Position the sensor at mid-tank or higher** for legionella purposes.
This ensures the recorded temperature reflects the bulk of the stored
water. If your sensor is near the bottom (common for depletion
detection), consider adding a second sensor higher up and using it as
the tank sensor for ThermoSync, or accept that the legionella cycle
may need to run longer than theoretically necessary to ensure the
mid/upper tank reaches temperature.

### How to enable

#### Step 1 — Create the tracker helper

1. Open Home Assistant → **Settings** (in the sidebar)
2. Click **Devices & Services**
3. Click the **Helpers** tab at the top
4. Click the blue **+ CREATE HELPER** button at the bottom right
5. From the list, select **Date and/or time**
6. Fill in the form:

| Field      | Value                                |
|------------|--------------------------------------|
| Name       | `ThermoSync Last Legionella Cycle`   |
| Icon       | `mdi:shield-check` (optional)        |
| Has date   | **Toggle ON**                        |
| Has time   | **Toggle ON**                        |

7. Click **CREATE**

This creates the entity `input_datetime.thermosync_last_legionella_cycle`.

#### Step 2 — Link it to ThermoSync

1. Open the ThermoSync automation
2. Expand the **Legionella Protection** section
3. In **Legionella Cycle Tracker**, select the helper you just created
4. Adjust intervals and temperature if needed
5. Save

#### Step 3 — Verify safety

Before going to bed on the first night legionella is enabled, run the
shower and kitchen tap. Confirm the delivered water temperature is
limited by the TMV (warm but not dangerously hot). If any outlet
delivers scalding water, **disable legionella protection immediately**.

---

## Raising the Temperature Limits

The safe defaults cap everything at 50 °C. To store more thermal energy,
raise the limits — **but only if you have TMVs fitted**.

| Setting                    | Safe default | With TMVs fitted |
|----------------------------|--------------|------------------|
| Base temperature           | 45 °C        | 50–55 °C         |
| Maximum temperature        | 50 °C        | 60–65 °C         |
| Negative rate temperature  | 50 °C        | 65–70 °C         |
| Solar target temperature   | 50 °C        | 55–60 °C         |
| Overnight demand boost     | +5 °C        | +8 to +10 °C     |

Always raise **Maximum Temperature** together with boosts or base —
otherwise the smart target is clamped silently.

---

## Example Profiles

### Safe defaults (no TMVs, any household size)

| Setting                  | Value  |
|--------------------------|--------|
| Base temperature         | 45 °C  |
| Maximum temperature      | 50 °C  |
| Overnight demand boost   | +5 °C  |
| Midday demand boost      | +3 °C  |
| Evening demand boost     | +0 °C  |
| Legionella protection    | Disabled |
| TargetTimeframes windows | 3 (overnight 2 h + midday 1.5 h + evening 1.5 h) |

All targets capped at 50 °C.

### Couple with TMVs (150–200 L tank)

| Setting                  | Value  |
|--------------------------|--------|
| Base temperature         | 50 °C  |
| Maximum temperature      | 60 °C  |
| Overnight demand boost   | +8 °C  |
| Midday demand boost      | +3 °C  |
| Evening demand boost     | +0 °C  |
| Negative rate temp       | 65 °C  |
| Legionella protection    | Enabled (65 °C, min 5 / max 10 days, 8 p) |
| TargetTimeframes windows | 2 (overnight 2 h + midday 1 h) |

### Family of 4 with TMVs (200–300 L tank)

| Setting                  | Value  |
|--------------------------|--------|
| Base temperature         | 55 °C  |
| Maximum temperature      | 65 °C  |
| Overnight demand boost   | +8 °C  |
| Midday demand boost      | +3 °C  |
| Evening demand boost     | +0 °C  |
| Negative rate temp       | 70 °C  |
| Legionella protection    | Enabled (65 °C, min 5 / max 10 days, 8 p) |
| TargetTimeframes windows | 3 (overnight 2 h + midday 1.5 h + evening 1.5 h) |

### Large family with TMVs (5+, 300 L+ tank)

| Setting                  | Value  |
|--------------------------|--------|
| Base temperature         | 58 °C  |
| Maximum temperature      | 68 °C  |
| Overnight demand boost   | +10 °C |
| Midday demand boost      | +5 °C  |
| Evening demand boost     | +3 °C  |
| Negative rate temp       | 70 °C  |
| Legionella protection    | Enabled (65 °C, min 5 / max 10 days, 8 p) |
| TargetTimeframes windows | 3 (overnight 2.5 h + midday 2 h + evening 1.5 h) |
| Cheap rate threshold     | 8 p    |

---

## Supported Tariffs

ThermoSync works with **any sensor that provides a live electricity
rate**.

| Tariff / Integration          | Rate entity example                           | Multiplier |
|-------------------------------|-----------------------------------------------|-----------|
| Octopus Agile (UK)            | `sensor.octopus_energy_electricity_..._current_rate` | 100  |
| Octopus Intelligent Go (UK)   | `sensor.octopus_energy_electricity_..._current_rate` | 100  |
| Octopus Cosy (UK)             | `sensor.octopus_energy_electricity_..._current_rate` | 100  |
| Tibber (EU/Nordic)            | `sensor.tibber_prices`                        | 100 or 1  |
| Nordpool (EU)                 | `sensor.nordpool_kwh_..._current_price`       | 100       |
| Amber Electric (AU)           | `sensor.amber_general_price`                  | 1         |
| Fixed / Economy 7 (UK)        | Template sensor with time-based pricing       | 1         |

For **fixed-rate tariffs** or **Economy 7**, create a template sensor
that outputs your rate in p/kWh based on time of day.

---

## Solar Diversion

### How it works

ThermoSync uses two signals for stability:

1. **Real-time export** — grid sensor must be negative (exporting) and
   exceed the threshold (default 500 W)
2. **Forecast** — expected generation in the next hour must exceed 1 kWh

The 5-minute evaluation interval provides natural debounce against
cloud cover changes.

### Automatic unit detection

ThermoSync reads the `unit_of_measurement` attribute from your grid
sensor and auto-converts between W and kW. A sensor reading `-150`
with `unit_of_measurement: "W"` is correctly interpreted as 0.15 kW
export — below the 0.5 kW threshold. No manual configuration needed.

### Forecast rollover smoothing

Solar forecast integrations (Forecast.Solar, Solcast) often drop the
current-hour reading to zero in the last few minutes as the hour rolls
over. ThermoSync smooths this by using `max(current_hour, next_hour)`.
At 14:55 when `energy_current_hour` drops to 0 but `energy_next_hour`
reads 1.9 kWh, the effective value stays at 1.9 — preventing a brief
heater cycle-off at the hour boundary.

### Solar + planned windows

When solar is active during a planned TargetTimeframes window,
ThermoSync uses the higher of the smart target and the solar target.
This ensures demand boosts (e.g. +5 °C for morning showers) aren't
silently overridden by the more conservative solar target.

---

## Advanced Price Logic

Three optional enhancements under the **Advanced Price Logic** section.
Each is a no-op if the relevant sensors aren't configured.

### 1. Cheap-rate look-ahead (default: 2 hours)

Prevents heating at a merely-cheap rate when a cheaper planned window
is about to start.

**Example**: It's 13:30, rate just dropped to 8 p (below 5 p cheap
threshold is not met, but below say a user-raised 10 p threshold),
TargetTimeframes window fires at 14:00 at 3 p. Without look-ahead:
heats at 8 p. With look-ahead: waits for 3 p.

### 2. Effective rate (solar-adjusted pricing)

When solar forecast sensors, base load, and heater power are all
configured, ThermoSync computes an effective rate:

```
solar_net      = max(0, solar_forecast − base_house_load)
coverage       = min(1, solar_net / heater_power)
effective_rate = grid_rate × (1 − coverage)
```

**Example**: Grid rate 14 p, solar forecast 1.9 kWh, base load 0.7 kWh,
3 kW heater. Net solar = 1.2 kWh, coverage = 40%, effective = 8.4 p.

The cheap-rate threshold check uses this effective rate, so nominally
expensive grid rates with strong solar coverage can qualify as cheap.

### 3. Window deferral

During a planned window, if the next hour's effective rate is at least
20% cheaper (configurable) and the tank has a comfortable buffer (above
depletion + 10 °C), ThermoSync defers.

**Requires**: the next-hour rate sensor. Solar forecast sensors are
optional — without them, deferral compares raw grid rates directly
(effective rate equals grid rate when solar coverage is zero). This
means deferral still works for users without solar panels.

---

## Template Sensors for Advanced Features

### Next-hour rate (Octopus Energy)

```yaml
template:
  - sensor:
      - name: "Octopus Next Hour Rate"
        unit_of_measurement: "£/kWh"
        state: >
          {% set rates = state_attr(
            'sensor.octopus_energy_electricity_YOUR_MPAN_current_day_rates',
            'rates') %}
          {% set target = now().replace(minute=0, second=0, microsecond=0)
                          + timedelta(hours=1) %}
          {% set target_ts = as_timestamp(target) %}
          {% if rates %}
            {% for r in rates %}
              {% if as_timestamp(r.start) <= target_ts
                    < as_timestamp(r.end) %}
                {{ r.value_inc_vat }}
              {% endif %}
            {% endfor %}
          {% endif %}
```

Replace `YOUR_MPAN` with your meter's identifier.

### Forecast.Solar setup

[Forecast.Solar](https://www.home-assistant.io/integrations/forecast_solar/)
provides per-hour generation forecasts. After installation:

- **Solar Diversion → Solar Forecast (next hour)** =
  `sensor.energy_next_hour`
- **Solar Diversion → Solar Forecast (current hour)** =
  `sensor.energy_current_hour`

---

## Resilience Features

### API outage protection

If the electricity rate sensor becomes `unavailable` or `unknown`, the
rate defaults to **999 p/kWh** (not 0). This prevents the previous
critical bug where an API outage would trigger negative-rate heating
at full grid price.

Additionally, the negative-rate check requires both `rate_valid` AND
`not rate_stale` — a sensor that stopped updating 30+ minutes ago
cannot trigger negative-rate heating even if its last value was zero.

### Stale data fallback

If either the rate or tank sensor hasn't updated within the stale data
threshold (default 30 minutes), ThermoSync enters **degraded mode**:

- Heat only during planned TargetTimeframes windows
- Target is base_temperature only (no demand or price boosts)
- No negative-rate logic, no cheap-rate extension, no solar pricing
- No deferral decisions

This ensures the system doesn't make price-optimised decisions based
on stale data, but still provides basic hot water via planned windows.

### Relay protection

Two mechanisms prevent physical relay damage:

1. **Hysteresis (deadband)**: Default 2 °C. Heating stops at the
   target (e.g. 50 °C) and won't restart until the tank drops to 48 °C.
   Within the deadband zone, ThermoSync maintains the heater's current
   state rather than toggling.

2. **Minimum dwell time**: Default 5 minutes. The dwell timer only
   gates turning heating **ON** — if the heater was off less than 5
   minutes ago, ThermoSync won't restart it even if conditions change.
   However, turning heating **OFF** or lowering a setpoint is always
   immediate — safety and cost savings are never delayed by the dwell
   timer. If the heater is already on and should continue heating,
   dwell is also bypassed (no interruption to an active cycle).

### HA restart safety

ThermoSync triggers on `homeassistant: start` with a 30-second delay.
This allows sensors to initialise from `unavailable` before the first
evaluation, preventing the stale-data fallback from engaging
unnecessarily on every reboot.

### Manual boost

When the optional manual boost switch is turned on, ThermoSync heats
to `max(max_temperature, negative_rate_temperature)` immediately,
overriding peak blocks and all price logic. The switch auto-disables
when the target is reached.

### Execution mode

The automation uses `mode: queued` (max 3) instead of `restart`. This
prevents a rapid sensor update from aborting a service call mid-
execution — each trigger is processed in order rather than killing the
previous one.

---

## Decision Priority Reference

```
P0  Not enabled                            →  Do nothing
P1  Manual boost on (+ tank sensor valid)  →  Heat to max(max_temp, neg_temp)
P2  Tank sensor invalid                    →  Do nothing (fail safe)
P3  Rate ≤ 0 AND valid AND not stale       →  Heat to negative_rate_temp
P4  Tank ≤ depletion                       →  Emergency heat
P5  Peak hours AND rate > threshold         →  Block heating
P6  Solar exporting (valid, not degraded)  →  Heat to max(smart_target, solar_target)
P7  Cheap rate AND no planned window soon  →  Heat to smart target
P8  Planned window AND not deferring       →  Heat to smart target
P9  Data degraded AND planned window       →  Heat to base_temp only
P10 Legionella due AND window active       →  Heat to legionella_temp
P11 Otherwise                              →  Don't heat
```

**Legionella scheduling** (P10) covers both opportunistic (min–max days
+ price + valid rate) and forced (overdue). Both require an active
planned window — forced legionella will not heat at arbitrary times or
prices, only during the next TargetTimeframes window.

**Deferral** (P8) compares the current and next hour's effective rate
(grid rate adjusted for solar coverage). If solar sensors are not
configured, it still works using raw grid rates. Requires the
next-hour rate sensor to be configured; otherwise deferral is disabled.

All decisions are further gated by **hysteresis** (prevents re-engaging
within the deadband), **dwell time** (prevents rapid relay cycling),
and **sensor validity** (aborts if tank reading is non-numeric).

---

## Troubleshooting

### "It's not heating during planned windows"

1. Check your TargetTimeframes binary sensor history — is it ever ON?
2. Check the tank temp — if already above smart_target, heating won't
   activate. View the automation trace for calculated values.
3. Verify the rate sensor is returning valid data (not `unavailable`).

### "It heats too often or at expensive times"

1. Lower the cheap rate threshold (e.g. from 5 p to 3 p).
2. Check the peak block threshold.
3. Verify the rate multiplier — if your sensor reads 15 (pence) and
   the multiplier is 100, ThermoSync sees 1500 p.

### "Family ran out of hot water"

1. If you have TMVs, raise max_temperature and base_temperature per
   the profiles table.
2. Increase the demand boost for the relevant period (ensure
   max_temperature is raised too).
3. Check TargetTimeframes hours — especially in winter with cold mains
   water, you may need longer windows.
4. Consider adding the midday or evening window if you're only using
   the overnight one.

### "Tank gets too hot / someone was burned"

1. **Immediately** clear the legionella tracker from the blueprint
   configuration (disables legionella heating).
2. Set max_temperature to 50 °C.
3. Set negative_rate_temperature to 50 °C.
4. Do not re-enable until TMVs are fitted and verified.

### "Legionella tracker never updates"

1. Verify the input_datetime helper has both date AND time toggled on.
2. Check your tank sensor reading vs `legionella_temperature`. The
   tracker accepts within 3 °C OR above 60 °C. If your physical
   thermostat cuts well below 60 °C, you may need to raise the
   thermostat's physical limit.
3. Set `legionella_temperature` to match your physical thermostat
   limit, not higher.

### "Solar diversion triggers but I'm not exporting"

1. Check your grid sensor's `unit_of_measurement` attribute. ThermoSync
   auto-detects W vs kW, but the sensor must read negative when
   exporting. If it reads positive when exporting, your sensor is
   inverted — fix it at the integration level.
2. Verify the export threshold is appropriate for your sensor's scale.

### "Heating at expensive rates during API outage"

This should not happen with the current version. If the rate sensor
becomes unavailable, ThermoSync defaults to 999 p/kWh and enters
degraded mode (planned windows at base temperature only). If you're
still seeing this, check the automation trace for `rate_valid` and
`data_degraded` values.

### "Heater ignores the temperature ThermoSync sets"

Some `climate` or `water_heater` integrations enforce min/max bounds on
the temperature setpoint. Common issues:

- **Idle temperature rejected**: if your device's minimum is 20 °C,
  writing 10 °C (ThermoSync's default idle) will be silently ignored
  or clamped. Raise `idle_temperature` to match your device's minimum.
- **Legionella target rejected**: if your device's maximum is 60 °C,
  writing 65 °C will be clamped. Lower `legionella_temperature` or
  raise the device's limit if possible.

Check your entity's `min_temp` and `max_temp` attributes in
**Developer Tools → States** and ensure ThermoSync's settings fall
within them.

### Checking what ThermoSync decided

Go to **Settings → Automations** → find ThermoSync → three-dot menu →
**Traces**. The trace shows every intermediate variable: `rate_valid`,
`rate_pence`, `data_degraded`, `effective_rate_now`, `smart_target`,
`should_heat_raw`, `should_heat`, `active_target`, `dwell_ok`, and
more. This is your primary debugging tool.

---

## Upgrading from Manual Schedules

1. **Week 1**: Install ThermoSync with safe defaults alongside your
   existing schedule. Monitor automation traces.
2. **Week 2**: Disable the fixed schedule. Let ThermoSync run at 50 °C
   safe cap. Check if hot water is sufficient.
3. **Week 3**: If you have TMVs, raise limits per the profile tables.
   If not, stay at 50 °C.
4. **Ongoing**: Watch the HA energy dashboard to compare costs.

---

## Complete Input Reference

### Core Configuration
| Input | Default | Description |
|-------|---------|-------------|
| tank_temperature_sensor | *required* | Tank temperature entity |
| electricity_rate_sensor | *required* | Electricity rate entity |
| rate_multiplier | 100 | Converts rate to pence (Octopus = 100) |
| hot_water_heater | *required* | Heater control entity |
| control_mode | set_temperature | How to drive the heater |
| idle_temperature | 10 °C | Setpoint when not heating |

### Schedule Windows
| Input | Default | Description |
|-------|---------|-------------|
| overnight_window | *(empty)* | TargetTimeframes binary sensor |
| midday_window | *(empty)* | TargetTimeframes binary sensor |
| evening_window | *(empty)* | TargetTimeframes binary sensor |

### Solar Diversion
| Input | Default | Description |
|-------|---------|-------------|
| grid_sensor | *(empty)* | Grid import/export (W or kW, auto-detected) |
| solar_forecast_sensor | *(empty)* | Next hour forecast (kWh) |
| current_hour_solar_sensor | *(empty)* | Current hour forecast (kWh) |
| solar_export_threshold | 0.5 kW | Minimum export to trigger |
| solar_forecast_threshold | 1.0 kWh | Minimum forecast to allow |
| solar_target_temp | 50 °C | Target on solar alone |

### Temperature Settings
| Input | Default | Description |
|-------|---------|-------------|
| base_temperature | 45 °C | Starting point for smart target |
| max_temperature | 50 °C | Hard cap (safe without TMVs) |
| negative_rate_temperature | 50 °C | Target when paid to consume |
| emergency_temperature | 50 °C | Recovery target when depleted |
| depletion_threshold | 23 °C | "One shower left" sensor reading |
| hysteresis | 2 °C | Deadband to prevent relay chatter |

### Demand Profile
| Input | Default | Description |
|-------|---------|-------------|
| overnight_boost | +5 °C | Extra for morning showers |
| midday_boost | +3 °C | Extra for afternoon/evening |
| evening_boost | +0 °C | Extra for last heat of day |
| overnight_start/end | 22 / 7 | Period boundaries (wraps midnight) |
| midday_start/end | 9 / 16 | Period boundaries |

### Price Thresholds
| Input | Default | Description |
|-------|---------|-------------|
| cheap_rate_threshold | 5 p | Extend heating outside windows |
| very_cheap_threshold | 3 p | Add bonus degrees |
| very_cheap_bonus | +5 °C | Bonus at very cheap rates |
| moderate_cheap_bonus | +2 °C | Bonus at moderate cheap rates |
| peak_start/end | 16 / 19 | Peak hours (wraps midnight) |
| peak_rate_threshold | 15 p | Block heating above this in peak |

### Advanced Price Logic
| Input | Default | Description |
|-------|---------|-------------|
| cheap_rate_lookahead_hours | 2 h | Skip cheap if window coming |
| next_hour_rate_sensor | *(empty)* | Next hour rate for deferral |
| base_house_load | 0.3 kW | Household baseline draw |
| heater_power_kw | 3 kW | Heater element power |
| defer_threshold_percent | 20% | Defer if next hour this much cheaper |

### Legionella Protection
| Input | Default | Description |
|-------|---------|-------------|
| legionella_tracker | *(empty)* | input_datetime helper (empty = disabled) |
| legionella_min_days | 5 | Don't schedule sooner than this |
| legionella_max_days | 10 | Force cycle after this (safety) |
| legionella_price_threshold | 8 p | Schedule when rate below this |
| legionella_temperature | 65 °C | Target (accepted within 3 °C or ≥60 °C) |

### Resilience & Safety
| Input | Default | Description |
|-------|---------|-------------|
| stale_data_minutes | 30 | Degrade after this long without update |
| min_cycle_minutes | 5 | Minimum on/off dwell time |
| manual_boost | *(empty)* | input_boolean for "hot water NOW" |
| enable_switch | *(empty)* | input_boolean to pause ThermoSync |

---

## Contributing

Issues, feature requests, and pull requests welcome. Please test any
changes against the example profiles above before submitting.

## License

MIT — use freely, attribution appreciated.
