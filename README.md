# PV Flow Card (m0e)

A custom Home Assistant Lovelace card for solar / battery / hybrid inverter
systems. It draws a live, animated energy-flow diagram — **Solar, Battery,
Home and Grid** as nodes with dots travelling along the wires at a speed
proportional to the power flowing — plus stat tiles, info rows, live prices,
preset buttons, charge timers and toggles, all in one card.

It is **integration-agnostic**: point it at whatever sensors your system
provides. It runs in production against GoodWe (core integration) and Victron
(Venus OS) systems, and works the same with SolarEdge, Fronius, Sungrow,
Huawei, ESPHome energy meters — anything that exposes power (W), energy (kWh)
and SOC (%) sensors.

- Single file, no build step, no dependencies
- GPU-composited animation — smooth even on old wall-tablet WebViews
- Only re-renders when *its own* entities change, so it stays cheap on busy
  Home Assistant instances
- Phone-portrait and landscape-tablet layouts, automatic
- Dark-dashboard styling, themable via CSS variables

**Preview:** serve the repo root (e.g. `python3 -m http.server`) and open
`/demo/` — a mock-data playground with selectable scenarios (sunny day,
evening, night import, fast charge).

---

## Install

### HACS

1. HACS → ⋮ → **Custom repositories** → add this repository's URL with type
   **Dashboard**.
2. Search for **PV Flow Card (m0e)** and download it. HACS registers the
   Lovelace resource for you.
3. Add the card to a dashboard (see Quick start).

### Manual

1. Copy `dist/goodwe-flow-card-m0e.js` into your HA `config/www/` folder.
2. Settings → Dashboards → ⋮ → **Resources** → Add →
   URL `/local/goodwe-flow-card-m0e.js`, type **JavaScript module**.

> The file keeps its original name for compatibility with existing installs.
> The card type is `custom:pv-flow-card-m0e`; the original
> `custom:goodwe-flow-card-m0e` still works as an alias.

---

## Quick start

The minimum useful config — four power sensors and an SOC:

```yaml
type: custom:pv-flow-card-m0e
name: MY HOUSE
entities:
  pv_power: sensor.my_pv_power                # W
  house_power: sensor.my_house_consumption    # W
  battery_power: sensor.my_battery_power      # W  (+ = discharging)
  battery_soc: sensor.my_battery_soc          # %
  grid_power: sensor.my_grid_power            # W  (+ = exporting; derived if omitted)
```

Everything else in this README is optional and appears only when configured.

### Sign conventions

The card assumes **battery positive = discharging** and **grid positive =
exporting**. Integrations disagree on this constantly; if a flow animates the
wrong way, flip it in config rather than editing sensors:

```yaml
invert_battery: true   # if "Charging" shows while the battery powers the house
invert_grid: true      # if import/export are swapped
```

If `grid_power` is omitted, the card derives grid flow from PV + battery −
house.

---

## Full example

```yaml
type: custom:pv-flow-card-m0e
name: SYMBOX
layout: wide
battery_capacity_kwh: 16
battery_min_soc: 20
soc_precision: 1
pv_max: 6600
house_max: 5000
tile_columns: 2
info_columns: 2
entities:
  pv_power: sensor.pv_power
  house_power: sensor.house_consumption
  battery_power: sensor.battery_power
  battery_soc: sensor.battery_soc
  grid_power: sensor.grid_power
  grid_price: sensor.electricity_price          # live $/kWh in the grid node
  production_today: sensor.pv_generation_today  # built-in "Production" tile
  battery_today: sensor.battery_discharge_today # built-in "Battery" tile
  grid_import_today: sensor.grid_import_today   # built-in "Grid in" tile
  grid_export_today: sensor.grid_export_today   # built-in "Grid out" tile
  last_update: sensor.inverter_timestamp        # header timestamp
bars:
  - entity: sensor.pv1_power
    name: PV1
    max: 6600
tiles:
  - section: Energy
  - entity: sensor.home_usage_today
    name: Home Usage
    sub: Today
    icon: home
    color: "#9b8cff"
  - entity: sensor.grid_import_today
    name: Grid Today
    sub: Import
    icon: grid
    entity2: sensor.grid_export_today
    name2: Export
  - section: Money
  - entity: sensor.buy_price
    name: Live Prices
    sub: Import
    icon: bell
    entity2: sensor.sell_price
    name2: Export
  - entity: sensor.savings_today
    name: Savings
    sub: Today
    icon: sun
    color: "#35d49a"
    entity2: sensor.savings_yesterday
    name2: Yesterday
info_title: System Information
info:
  - entity: sensor.grid_connection_status
    name: Grid Status
  - entity: sensor.inverter_temperature
    name: Temp
    icon: thermo
  - entity: sensor.battery_time_to_go
    name: Time Left
    format: duration
  - entity: sensor.alarms
    name: Alarms
    ok_states: [OK]
buttons:
  - entity: number.grid_setpoint
    name: Grid Setpoint
    columns: 3
    options: [20, 50, 150, 500, 1000, 1600]
timers:
  - title: Timer Controls
    name: Charge Timer
    toggle: input_boolean.charge_timer_enabled
    start: input_datetime.charge_start
    end: input_datetime.charge_end
switches:
  - entity: switch.fast_charging
    name: Fast Charge
```

---

## Options reference

### Top level

| Option | Default | Description |
| --- | --- | --- |
| `name` | `PV Flow` | Card title |
| `layout` | `auto` | `auto` = flow beside the tiles when the card is wider than ~620 px (landscape tablet), stacked when narrow. `wide` / `tall` force it |
| `low_fx` | `false` | Drop glows, pulses and transitions for weak tablet GPUs; flow dots keep animating |
| `battery_capacity_kwh` | — | Usable pack size. Enables the kWh readout in the SOC ring and the time-to-full / time-to-empty estimate |
| `battery_min_soc` | `0` | Discharge floor (%); time-to-empty counts down to this |
| `soc_precision` | `0` | Decimal places for the SOC readout (0–2) |
| `pv_max` / `house_max` | — | Adds a power ring around the Solar / Home node, filled by live watts against this max (like the SOC ring) |
| `invert_battery` | `false` | Flip battery sign (see Sign conventions) |
| `invert_grid` | `false` | Flip grid sign |
| `tile_columns` | `2` | Columns in the tile grid (1–4) |
| `info_columns` | `1` | Columns in the info list (1–4) |
| `info_title` | — | Section heading above the info list |
| `show_bars` / `show_stats` / `show_separator` | `true` | Hide the bars / tiles / the divider under the flow diagram |

### `entities:`

| Key | Description |
| --- | --- |
| `pv_power` | Total PV power (W). Legacy alias: `solar_power` |
| `house_power` | House consumption (W). Legacy alias: `output_power` |
| `battery_power` | Battery power (W), + = discharging |
| `battery_soc` | State of charge (%). Legacy alias: `battery_percentage` |
| `grid_power` | Grid power (W), + = exporting. Derived from the others if omitted |
| `grid_price` | Live electricity price shown inside the grid node (`$/kWh` or `¢/kWh` sensors, e.g. Amber Electric) |
| `production_today` | Daily PV energy → built-in "Production" tile |
| `battery_today` | Daily battery energy → built-in "Battery" tile. Legacy alias: `battery_capacity` |
| `grid_import_today` / `grid_export_today` | Daily grid energy → built-in tiles |
| `last_update` | Timestamp sensor shown in the header |
| `p1_power` … `p4_power` | Per-string PV sensors (legacy b2500d style; scaled by `max_input_power`…`max_input_power4`) |

Built-in tiles only render when their entity is set.

### `bars:` — labelled power bars

```yaml
bars:
  - entity: sensor.pv1_power
    name: PV1
    max: 6600            # W for a full bar (default 4000)
    color: "#5aa9e6"     # optional, default solar amber
```

A `%` sensor fills the bar directly, no `max` needed. `strings:` is an
accepted alias.

### `tiles:` — stat tiles

```yaml
tiles:
  - section: Energy      # header + starts a new grid; optional columns: 3
  - entity: sensor.x
    name: Title          # bold title (icon renders beside it)
    sub: Today           # dim label (default "Now")
    icon: chart          # sun home battery grid chart bolt clock bell thermo
    color: "#ffb648"     # tints the icon
    format: duration     # optional: duration | datetime | time
    entity2: sensor.y    # second full-size value, right column
    name2: Label2        # label over the second value
    alert_states: [ALARM]   # flash red while state matches...
    ok_states: [OK]         # ...or while it does NOT match these
```

Values auto-format by the sensor's unit: `W`/`kW`, `Wh`/`kWh`, `%`, prices
(`$/kWh`, `¢/kWh`, plain `$`/currency codes), text states as-is. Tapping a
tile opens the main entity's more-info; tapping the second value opens
`entity2`'s.

### `info:` — compact label/value rows

```yaml
info:
  - entity: sensor.x
    name: Label
    icon: bolt           # optional small icon before the label
    color: "#35d49a"     # optional; tints icon and value
    format: duration     # duration | datetime | time
    ok_states: [OK]      # same alert flashing as tiles
```

Rows use the same unit formatting as tiles and open more-info on tap. Lay
them out in columns with `info_columns`.

### `buttons:` — preset grids

```yaml
buttons:
  - entity: number.grid_setpoint     # number, input_number or select
    name: Grid Setpoint              # header shows the live value
    icon: bolt
    columns: 3                       # 1–6
    options: [20, 50, {value: 700, label: Boost}]
```

Tap calls `number.set_value` (or `select.select_option`); the active option
highlights. Numeric labels pick up the entity's unit automatically.

### `timers:` — schedule controls

```yaml
timers:
  - title: Timer Controls            # optional section header
    name: Charge Timer               # pill shows "Charge Timer Enabled/Disabled"
    toggle: input_boolean.timer_on   # optional; tap the pill to toggle
    start: input_datetime.start      # Start tile; tap opens HA's picker
    end: input_datetime.end          # End tile
    start_name: Start                # optional labels
    end_name: End
    icon: clock
```

The pill's second line shows the window ("29 Aug 02:00 → 04:00"; the end
collapses to time-only when same-day).

### `switches:` — toggle pills

```yaml
switches:
  - entity: switch.fast_charging     # switch, input_boolean or automation
    name: Fast Charge
```

Tap toggles via `homeassistant.toggle`. Legacy alias: `custom_settings`.

### `labels:` — every text on the card

All optional; defaults shown:

```yaml
labels:
  solar: Solar            # node labels
  home: Home
  battery: Battery
  grid: Grid
  grid_import: import     # → "Grid · import"
  grid_export: export
  charging: Charging      # battery label while charging
  full: Full              # ETA wording → "Full 2:40 pm"
  empty: Empty
  updated: updated        # header prefix → "updated 12:04 pm"
  today: Today            # sub-label on built-in tiles
  production: Production  # built-in tile titles
  battery_today: Battery
  grid_in: Grid in
  grid_out: Grid out
  enabled: Enabled        # timer pill wording
  disabled: Disabled
```

---

## Behaviour notes

- **Battery ETA** — with `battery_capacity_kwh` set, the battery node shows
  "Full 2:40 pm" while charging and "Empty 11:05 pm" while discharging
  (to `battery_min_soc`), from the current power. Rounded to 5 minutes,
  hidden beyond 24 h.
- **Flows** — dots animate solar→home, solar→battery, solar→grid,
  battery→home, grid→home and grid→battery; speed scales with watts; flows
  under 25 W are treated as zero.
- **Alert flashing** — a tile or info row with `alert_states` / `ok_states`
  pulses red (value and icon too) until the state recovers. Kept even in
  `low_fx` — it's a signal, not decoration.
- **Performance** — the card ignores `hass` updates that don't touch its own
  entities, and the dot animation runs as CSS `transform` keyframes on the
  GPU compositor, so it stays smooth while HA is busy. `low_fx: true`
  additionally removes glow/blur painting for very weak devices.
- **Layouts** — `auto` switches to side-by-side (flow left, everything else
  right of a vertical divider) when the card is wider than ~620 px, via
  ResizeObserver (works on old WebViews that lack container queries).
- **Console badge** — the loaded version prints in the browser console
  (`PV-FLOW-CARD-M0E v…`), useful when chasing caches.

## Theming

Override in your HA theme; the card follows the active theme's background and
text colours:

```yaml
--gw-solar-color: "#ffb648"
--gw-batt-color: "#35d49a"
--gw-grid-color: "#5aa9e6"
--gw-house-color: "#9b8cff"
```

## Migrating from b2500d-card

Legacy entity keys (`solar_power`, `p1_power`, `output_power`,
`battery_percentage`, `production_today`, `battery_capacity`,
`custom_settings`) are accepted — an existing b2500d config drops in with
only the `type:` line changed.
