# Enlighten_UX_4_ESPHome
Early working prototype of an Enphase-stle Enlighten UX for ESPHome.

# Enlighten UX

A reusable ESPHome/LVGL energy-flow visualization for touch displays.

Enlighten UX provides a graphical representation of energy moving between:

- Grid
- Solar
- Battery
- House

Energy flows are represented by static paths with animated pulses showing the direction of energy movement.

## Current status

This project is being open sourced in its current working state.

The visual layout and energy-flow animation system are intended to be reusable. The application/data layer is currently tied to the author's Home Assistant energy-monitoring setup and is not included in this initial standalone package.

In other words: the UX is generalized, the data plumbing isn't (yet).

## Main Device Configuration

The Enlighten UX files are included from the main ESPHome device YAML.

### 1. Define the Enlighten UX boundary

Add these substitutions to the main device YAML:

```yaml
substitutions:
  # Enlighten UX substitutions that set the boundary of the widgets and animations
  enlighten_x: "16"
  enlighten_y: "16"
  enlighten_width: "650"
  enlighten_height: "350"
```

These four values define the position and size of the Enlighten UX area. Changing them moves and/or resizes the visualization without requiring the individual widgets or flow paths to be rewritten.

### 2. Include the Enlighten feature package

The intended application-specific integration is:

```yaml
# APPLICATION FEATURES
# ============================================================================

packages:
  - !include enlighten/enlighten_features.yaml
```

`enlighten_features.yaml` is the application-specific layer that connects Home Assistant sensors to the visual widgets and determines which energy flows are displayed.

That file is included in this initial package but note **it is based on the author's HA setup, and includes some values calculated in HA and not available as 'out-of the box' Enphase values**

- **sensor.combined_solar_production** - This is a sensor that combines the current Enphase solar production with the legacy solar production.
- **whole_home_envoy_current_net_power_consumption** - Grid import/export
- **whole_home_envoy_battery** - current battery charge percentage.
- **whole_home_envoy_current_battery_discharge** - Current battery charge/discharge
- **whole_home_envoy_home_consumption** - this is a custom, calculated value that makes a 'best guess' about how much power is not going to either the battery or the grid and assumes that is consumption
- **whole_home_envoy_grid_connected** - calculated value based on both the Enphase admin setting (i.e. manually disconnecting from the grid) and the actual grid connection

### 3. Include the Enlighten widgets and flows

Inside the LVGL page where the Enlighten UX should appear:

```yaml
pages:
  - id: ev_page
    widgets:
      - !include enlighten/enlighten_widgets_grid.yaml
      - !include enlighten/enlighten_flows.yaml
```

### 4. Include the animations

Inside the main `lvgl:` configuration:

```yaml
animations: !include enlighten/enlighten_animations.yaml
```

## Complete Integration Example

The relevant portions of a host device configuration look like:

```yaml
substitutions:
  enlighten_x: "16"
  enlighten_y: "16"
  enlighten_width: "650"
  enlighten_height: "350"

# APPLICATION FEATURES
# ============================================================================

packages:
  - !include enlighten/enlighten_features.yaml

lvgl:
  pages:
    - id: ev_page
      widgets:
        - !include enlighten/enlighten_widgets_grid.yaml
        - !include enlighten/enlighten_flows.yaml

  animations: !include enlighten/enlighten_animations.yaml
```

The host device must also provide the normal ESPHome hardware, display, fonts, and LVGL configuration required by the target display.

## Files

```text
enlighten/
├── enlighten_features.yaml
├── enlighten_widgets_grid.yaml
├── enlighten_flows.yaml
└── enlighten_animations.yaml
```

### `enlighten_widgets_grid.yaml`

Contains the primary visual layout. The current implementation uses a 5-column LVGL grid containing Solar, Battery, Grid, and House.

The layout is positioned and sized using:

```text
enlighten_x
enlighten_y
enlighten_width
enlighten_height
```

### `enlighten_flows.yaml`

Contains the static energy-flow paths and their associated pulse objects.

Current paths:

```text
Grid       -> House
Battery    -> House
Solar      -> House
Solar      -> Grid
Battery    -> Grid
Solar      -> Battery
```

Battery -> Grid is currently included for completeness but hidden by default.

The flow geometry is calculated from the Enlighten boundary substitutions.

### `enlighten_animations.yaml`

Contains the LVGL animations for the pulse objects.

Current animations include:

- Grid -> House
- Battery -> House
- Battery -> Grid
- Solar -> House
- Solar -> Grid
- Solar -> Battery

Some paths use chained animations to move a pulse through multiple segments.

## Current Home Assistant Sensor Dependencies

The current application logic was developed against a Home Assistant energy-monitoring installation.

The following entities are currently known to be used by the Enlighten feature layer:

### Grid

```text
sensor.whole_home_envoy_current_net_power_consumption
```

Used to determine whether the home is importing or exporting energy.

Current interpretation:

```text
negative / below threshold -> Exporting
positive / above threshold -> Importing
```

### Battery

```text
sensor.whole_home_envoy_current_battery_discharge
```

Current interpretation:

```text
approximately 0       -> Idle
positive              -> Discharging
negative              -> Charging
```

Current thresholds:

```text
-0.05 < power < 0.05   -> Idle
power >= 0.05          -> Discharging
power < -0.05          -> Charging
```

### Solar

The Solar widget is connected to a Home Assistant solar-production sensor in the application-specific feature layer.

The exact entity ID is intentionally not guessed here because the feature file was not included in this release package.

### House

The House widget is connected to a Home Assistant house-consumption sensor in the application-specific feature layer.

The exact entity ID is intentionally not guessed here because the feature file was not included in this release package.

## Known Limitations / Warts

This is not presented as a polished ESPHome component.

Current limitations include:

- The application/data layer is coupled to specific Home Assistant entities.
- Energy-flow direction is based on the sign conventions of the current sensors.
- Flow geometry uses calculated coordinates based on the Enlighten boundary.
- Some LVGL layout coordinates require manual offsets.
- Animation definitions contain duplicated geometry.
- Battery -> Grid flow exists but is intentionally disabled in the current setup.
- The implementation assumes the current LVGL/display environment.
- The project has primarily been tested on the original hardware/configuration.
- ESPHome/LVGL behavior can change between ESPHome releases.
- Requires "**fonts/materialdesignicons-webfont.ttf**"

The most useful future improvement would be making the sensor/entity layer configurable without making the display configuration significantly harder to understand.

## Hardware

The original development target is a Guition ESP32-P4 display with a 1024 x 600 MIPI DSI display.

The Enlighten UX itself is not intended to depend on that exact resolution. The visualization is positioned using the four Enlighten substitutions rather than hard-coded screen coordinates.

The host device must provide:

- ESPHome
- LVGL
- A compatible display
- The fonts required by the widget definitions
- The icon fonts used by the widget definitions

Hardware-specific display, touch, networking, and power configuration should remain in the host device YAML rather than in the Enlighten UX files.

## Contributing

Useful contributions would include:

1. Making Home Assistant entity IDs configurable.
2. Separating energy-flow state logic from the display implementation.
3. Supporting different sensor sign conventions.
4. Supporting additional energy sources and destinations.
5. Simplifying the animation geometry.
6. Testing on other ESPHome LVGL displays.

Pull requests that improve the architecture without making the configuration significantly harder to understand are preferred.

## License
# Copyright (c) 2026 [John Offenhartz]
# Licensed under the MIT License.
# See LICENSE for details.

Add the project's chosen license here.
