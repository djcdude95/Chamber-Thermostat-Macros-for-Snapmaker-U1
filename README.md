# Chamber Thermostat Macros for Snapmaker U1 (Klipper)

A lightweight thermostat-style chamber exhaust control system for the paxx12 extended firmware running on the Snapmaker U1.

This project adds true hysteresis-based chamber cooling behavior using Klipper macros and the built-in Snapmaker purifier module.

---

# Credits & Dependencies

## Required Firmware

This project requires the excellent Snapmaker U1 Extended Firmware project by paxx12:

[https://github.com/paxx12-snapmaker-u1/SnapmakerU1-Extended-Firmware](https://github.com/paxx12-snapmaker-u1/SnapmakerU1-Extended-Firmware)

The macros and purifier functionality used in this project depend on features provided by the extended firmware, including:

* Klipper integration
* Purifier module support
* `SET_PURIFIER` commands
* Chamber temperature access
* Expanded printer configuration functionality

Huge thanks to paxx12 and contributors for making the U1 significantly more open and customizable.

---

## Hardware / Purifier Research Credits

A large amount of useful hardware and purifier information came from:

[https://github.com/WilliamTheMaker/BuildMyOwnBlackBox_for_Snapmaker_U1](https://github.com/WilliamTheMaker/BuildMyOwnBlackBox_for_Snapmaker_U1)

Specifically helpful:

* Top fan connector pinout information
* Wiring information
* Purifier connection details
* `SET_PURIFIER` usage examples
* Understanding how the Snapmaker purifier module behaves

This information was extremely helpful for understanding how to safely interface with the stock purifier system. Please reference his guides for wiring.

---
I found in my testing that using a simple 24v to 12v buck converter for the VCC and GND to the 12v PWM fan works fine. The PWM signal to the fan is direct out of the printer plug. I will be adding a wiring diagram at some point in the near future, but for now the details in the above project should suffice.

---

## Original Inspiration

The original inspiration for this project came from this Facebook post/discussion:

[https://www.facebook.com/groups/snapmakeru1/posts/1274997118037665](https://www.facebook.com/groups/snapmakeru1/posts/1274997118037665)

The discussion and experimentation there helped inspire the idea of implementing smarter thermostat-style chamber cooling behavior for the U1.

---

# Why This Exists

The stock Snapmaker purifier cooling behavior supports:

* chamber temperature monitoring
* dynamic fan control
* purifier automation
* cooling modes

However, it does not behave like a traditional thermostat, at least in my testing... Could be user error.

Observed behavior:

* fan ramps correctly
* fan turns on
* fan does not reliably turn back off once chamber temperature drops below target
* the PWM control was fairly noisey for the specific fan I was using. Full speed was basically noiseless.

This project solves that by implementing:

* full ON / full OFF logic
* hysteresis temperature control
* configurable target temperatures
* configurable hysteresis
* Fluidd macro integration
* OrcaSlicer variable support

---

# Features

## Automatic Thermostat Cooling

* Turns exhaust fan ON above target temperature
* Turns exhaust fan OFF below target minus hysteresis

Example:

* ON at 40°C
* OFF at 35°C

---

## Full Manual Control

Additional macros:

* full fan ON
* full fan OFF

---

## Fluidd Integration

Macros automatically appear in in the macro list and console commands.

Supports parameter input directly from Fluidd.

---

## OrcaSlicer Integration

Supports:

* automatic chamber temp variables
* fallback defaults
* filament-specific chamber targets


---

# Hardware Requirements

*  Molex Micro-Fit 3.0mm - 6 Pin
*  10k ohm resistor
*  PWM fan (12v or 24v)
*  24v to 12v buck converter (if using 12v PWM fan)
  
---

# Software Requirements

* Snapmaker U1
* Snapmaker U1 Extended Firmware by paxx12
* Klipper
* Fluidd or Mainsail
* OrcaSlicer

---

# Installation

## 1. Create Config File

Create:

```text
chamber_fan.cfg
```

inside your Klipper config directory.

---

## 2. Add Include To printer.cfg

```ini
[include chamber_fan.cfg]
```

---

## 3. Paste This Into chamber_fan.cfg

```ini
[gcode_macro AUTO_CHAMBER_FAN]
description: Automatic chamber cooling control
variable_enabled: 0
variable_state: 0
variable_target: 40
variable_hysteresis: 5

gcode:
    {% set target = params.TARGET|default(printer["gcode_macro AUTO_CHAMBER_FAN"].target)|float %}
    {% set hysteresis = params.HYST|default(printer["gcode_macro AUTO_CHAMBER_FAN"].hysteresis)|float %}

    SET_GCODE_VARIABLE MACRO=AUTO_CHAMBER_FAN VARIABLE=enabled VALUE=1
    SET_GCODE_VARIABLE MACRO=AUTO_CHAMBER_FAN VARIABLE=target VALUE={target}
    SET_GCODE_VARIABLE MACRO=AUTO_CHAMBER_FAN VARIABLE=hysteresis VALUE={hysteresis}

    RESPOND MSG="Auto chamber fan enabled"

    UPDATE_DELAYED_GCODE ID=CHAMBER_FAN_LOOP DURATION=1


[gcode_macro STOP_AUTO_CHAMBER_FAN]
description: Stop automatic chamber cooling

gcode:
    SET_GCODE_VARIABLE MACRO=AUTO_CHAMBER_FAN VARIABLE=enabled VALUE=0
    SET_GCODE_VARIABLE MACRO=AUTO_CHAMBER_FAN VARIABLE=state VALUE=0

    SET_PURIFIER FAN=exhaust SPEED=0

    RESPOND MSG="Auto chamber fan disabled"


[gcode_macro CHAMBER_FAN_ON]
description: Turn chamber exhaust fan fully ON

gcode:
    SET_PURIFIER FAN=exhaust SPEED=1
    RESPOND MSG="Chamber exhaust fan ON"


[gcode_macro CHAMBER_FAN_OFF]
description: Turn chamber exhaust fan fully OFF

gcode:
    SET_PURIFIER FAN=exhaust SPEED=0
    RESPOND MSG="Chamber exhaust fan OFF"


[delayed_gcode CHAMBER_FAN_LOOP]
gcode:
    {% set temp = printer["temperature_sensor cavity"].temperature %}

    {% set enabled = printer["gcode_macro AUTO_CHAMBER_FAN"].enabled %}
    {% set state = printer["gcode_macro AUTO_CHAMBER_FAN"].state %}
    {% set target = printer["gcode_macro AUTO_CHAMBER_FAN"].target %}
    {% set hysteresis = printer["gcode_macro AUTO_CHAMBER_FAN"].hysteresis %}

    {% set off_temp = target - hysteresis %}

    {% if enabled == 1 %}

        {% if temp >= target and state == 0 %}
            SET_PURIFIER FAN=exhaust SPEED=1
            SET_GCODE_VARIABLE MACRO=AUTO_CHAMBER_FAN VARIABLE=state VALUE=1
            RESPOND MSG="Chamber fan ON"

        {% elif temp <= off_temp and state == 1 %}
            SET_PURIFIER FAN=exhaust SPEED=0
            SET_GCODE_VARIABLE MACRO=AUTO_CHAMBER_FAN VARIABLE=state VALUE=0
            RESPOND MSG="Chamber fan OFF"
        {% endif %}

        UPDATE_DELAYED_GCODE ID=CHAMBER_FAN_LOOP DURATION=5

    {% endif %}
```

---

## 4. Restart Klipper

Click "Save and Restart, or Run:

```gcode
RESTART
```

---

# Fluidd Usage

Macros will automatically appear in Fluidd:

* AUTO_CHAMBER_FAN
* STOP_AUTO_CHAMBER_FAN
* CHAMBER_FAN_ON
* CHAMBER_FAN_OFF

---

# Usage Examples

## Automatic Mode

Default:

```gcode
AUTO_CHAMBER_FAN
```

Custom:

```gcode
AUTO_CHAMBER_FAN TARGET=45 HYST=8
```

Meaning:

* ON at 45°C
* OFF at 37°C

---

## Manual Mode

Full ON:

```gcode
CHAMBER_FAN_ON
```

Full OFF:

```gcode
CHAMBER_FAN_OFF
```

---

## Stop Automation

```gcode
STOP_AUTO_CHAMBER_FAN
```

---

# OrcaSlicer Integration

## Start G-code

You can use the chamber_temperature variable in your filament profile to automatically set the desired MAX chamber temperature. In this example I have it default to 40c if no chamber_temperature is definded. You do _NOT_ need to check "Activate Temperature Control" in the filament profile for this to work.

```gcode
AUTO_CHAMBER_FAN TARGET={if chamber_temperature[0] > 0}{chamber_temperature[0]}{else}40{endif} HYST=5
```

Behavior:

* uses filament chamber temperature if defined
* defaults to 40°C otherwise

A simpler version uses a hardcoded target, and doesnt use the chamber_temperature variable.

```gcode
AUTO_CHAMBER_FAN TARGET=40 HYST=5
```


---

## End G-code

```gcode
STOP_AUTO_CHAMBER_FAN
```

---


# Tested With

* Snapmaker U1 Extended Firmware
* Klipper
* Fluidd
* Moonraker
* OrcaSlicer (NOT SNORCA)

---


# Additional Credits

Additional thanks again to the following projects and community members whose work and experimentation made this possible:

## paxx12 Extended Firmware

[https://github.com/paxx12-snapmaker-u1/SnapmakerU1-Extended-Firmware](https://github.com/paxx12-snapmaker-u1/SnapmakerU1-Extended-Firmware)

For building the extended firmware environment and exposing the purifier functionality required for this project.

---

## WilliamTheMaker

[https://github.com/WilliamTheMaker/BuildMyOwnBlackBox_for_Snapmaker_U1](https://github.com/WilliamTheMaker/BuildMyOwnBlackBox_for_Snapmaker_U1)

For purifier pinouts, fan wiring information, hardware experimentation, and useful purifier macro examples.

---

## Snapmaker U1 Community Discussion

[https://www.facebook.com/groups/snapmakeru1/posts/1274997118037665](https://www.facebook.com/groups/snapmakeru1/posts/1274997118037665)

For the original inspiration for me to make this and community experimentation surrounding the chamber fan wiring.

---
