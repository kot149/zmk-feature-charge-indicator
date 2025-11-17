# ZMK Feature: Charge Indicator

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

A ZMK feature module to indicate battery charging status on an RGB LED, designed to coexist with the `rgbled_widget`.

## Features

<details>
  <summary>Short video demo</summary>
  See below video for a short demo, charging in progress.

  https://github.com/user-attachments/assets/bfcb3e25-f645-47b0-8549-38c3128ddc15
</details>

- **Devicetree Driven**: Detects charging via a `chg_stat` node label, making it board-agnostic.
- **Widget Coexistence**: Suppresses the `rgbled_widget` only during charging, allowing it to function normally otherwise.
- **Configurable**: Choose to show color based on the current battery level or show a fixed color or turn the LED off while charging.
- **Split-Friendly**: Each split half can indicate its own charging status.
- **Lightweight**: Uses no heap and runs a static thread only when necessary.

## Prerequisites

- Your ZMK keyboard configuration.
- An RGB LED composed of three individual GPIO-controlled LEDs (not "smart" LEDs like WS2812).

## Setup Guide

### Step 1: Update `west.yml`

Add the module to your `config/west.yml`:

```yaml
manifest:
  remotes:
    - name: 4mplelab
      url-base: https://github.com/4mplelab
  projects:
    - name: zmk-feature-charge-indicator
      remote: 4mplelab
      revision: main
  self:
    path: config
```

Then, run `west update`.

### Step 2: Devicetree Configuration

You must configure two items in your Devicetree (`.dts` or overlay files): the `chg-stat` node and the RGB LED aliases.

1.  **Define `chg-stat` Node (Required)**

    This tells the module which pin to monitor. The node must be labeled `chg_stat` for the driver to find it.

    ```dts
    /* In your board.dts or overlay */
    / {
        chg_stat: chg_stat {
            compatible = "custom,chg-stat";
            /* Adjust for your board, e.g., P0.17 for XIAO nRF52840 */
            gpios = <&gpio0 17 GPIO_ACTIVE_LOW>;
            status = "okay";
        };
    };
    ```

2.  **Define LED Aliases (Choose One Method)**

    **Method A: Using `rgbled_adapter`**

    If you are using a board supported by the [`rgbled_adapter`](https://github.com/zmkfirmware/zmk/tree/main/app/boards/shields/rgbled_adapter) shield (like the Seeed XIAO BLE), you can simply add it to your build. It automatically creates the necessary `led-red`, `led-green`, and `led-blue` aliases.

    Add it as a shield in your `build.yaml` or similar build script:
    ```yaml
    ---
    include:
      - board: seeeduino_xiao_ble
        shield: your_shield_name rgbled_adapter
    ```

    **Method B: Manual Definition**

    If you don't use the adapter, define the `gpio-leds` and aliases manually:
    ```dts
    /* In your shield.overlay or board.dts */
    / {
        aliases {
            led-red = &led_red;
            led-green = &led_green;
            led-blue = &led_blue;
        };

        leds {
            compatible = "gpio-leds";
            led_red: led_red       { gpios = <&gpio0 26 GPIO_ACTIVE_LOW>; };
            led_green: led_green   { gpios = <&gpio0 30 GPIO_ACTIVE_LOW>; };
            led_blue: led_blue     { gpios = <&gpio0 6 GPIO_ACTIVE_LOW>; };
        };
    };
    ```

### Step 3: Kconfig Configuration

Enable the feature and set your desired behavior in your `.conf` file.

| Kconfig Option                  | Description                                                                                             | Default |
| ------------------------------- | ------------------------------------------------------------------------------------------------------- | ------- |
| `CONFIG_CHARGE_INDICATOR`       | **Required.** Enables the charge indicator feature.                                                     | `n`     |
| `CONFIG_CHG_POLICY`             | Defines charging behavior: `n` to show a color, `y` to force the LED off.                               | `n`     |
| `CONFIG_CHG_COLOR`              | Sets the color for charging (if policy is `n`). Values `0-7`.                                           | `Red (1)` |
| `CONFIG_CHG_BATTERY_LEVEL_BASED_COLOR` | Use battery level based color instead of fixed color.                                                 | `y`     |
| `CONFIG_CHG_BATTERY_LEVEL_HIGH`        | High battery level percentage.                                                                         | `80`    |
| `CONFIG_CHG_BATTERY_LEVEL_LOW`         | Low battery level percentage.                                                                          | `20`    |
| `CONFIG_CHG_BATTERY_LEVEL_CRITICAL`    | Critical battery level percentage.                                                                     | `5`     |
| `CONFIG_CHG_BATTERY_COLOR_HIGH`        | Color for high battery level (above LEVEL_HIGH).                                                       | Green (`2`)     |
| `CONFIG_CHG_BATTERY_COLOR_MEDIUM`      | Color for medium battery level (between LEVEL_LOW and LEVEL_HIGH).                                     | Yellow (`3`)     |
| `CONFIG_CHG_BATTERY_COLOR_LOW`         | Color for low battery level (below LEVEL_LOW).                                                          | Red (`1`)     |
| `CONFIG_CHG_BATTERY_COLOR_CRITICAL`    | Color for critical battery level (below LEVEL_CRITICAL).                                                | Magenta (`5`)     |
| `CONFIG_CHG_BATTERY_COLOR_MISSING`     | Color for battery not detected.                                                                       | Black (`0`)     |

<details>
<summary>Mapping for color values</summary>

| Color        | Value |
| :----------- | :---- |
| Black (none) | `0`   |
| Red          | `1`   |
| Green        | `2`   |
| Yellow       | `3`   |
| Blue         | `4`   |
| Magenta      | `5`   |
| Cyan         | `6`   |
| White        | `7`   |

</details>

**Example `my_keyboard.conf`:**
```ini
# Enable the charge indicator
CONFIG_CHARGE_INDICATOR=y

# Show a blue color (regardless of the current battery level) when charging
CONFIG_CHG_POLICY=n
CONFIG_CHG_COLOR=4
CONFIG_CHG_BATTERY_LEVEL_BASED_COLOR=n
```

```ini
# Show a battery level based color when charging
CONFIG_CHG_POLICY=n
CONFIG_CHG_BATTERY_LEVEL_BASED_COLOR=y
CONFIG_CHG_BATTERY_LEVEL_HIGH=90
CONFIG_CHG_BATTERY_LEVEL_LOW=20
CONFIG_CHG_BATTERY_LEVEL_CRITICAL=5
CONFIG_CHG_BATTERY_COLOR_HIGH=2
CONFIG_CHG_BATTERY_COLOR_MEDIUM=3
CONFIG_CHG_BATTERY_COLOR_LOW=1
CONFIG_CHG_BATTERY_COLOR_CRITICAL=5
CONFIG_CHG_BATTERY_COLOR_MISSING=0
```

## Behavior

- **When Charging**: The module takes control of the RGB LED and applies your chosen policy (`color` or `off`), suppressing the `rgbled_widget`.
- **When Not Charging**: The module does nothing, allowing the `rgbled_widget` to operate normally without interference.

## Troubleshooting

- **Build error like "undefined node label 'chg_stat'"**: Ensure your Devicetree overlay defines a node with the label `chg_stat:`. For example: `chg_stat: chg_stat { ... };`. Also, verify the overlay is being applied during the build.
- **Incorrect Charging Detection**: Check that the `gpios` property in your `chg_stat` node points to the correct pin for your board's charge status signal.
- **Widget Conflicts**: This module is designed to avoid conflicts when not charging. If issues persist, check for other modules controlling the same LEDs. To hide the `rgbled_widget`'s default USB indicator, you can set `CONFIG_RGBLED_WIDGET_CONN_SHOW_USB=n`.

---

### Acknowledgements

Built to complement the excellent `rgbled_widget` by caksoylar and the ZMK community.
