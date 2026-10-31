# Aquarea Mixing Valve Control

ESPHome-based controller for a 3-way mixing valve used in a floor-heating circuit. The project is designed for a Panasonic Aquarea heat pump with a separate buffer tank and aims to reduce the heat pump's unnecessarily high buffer temperature by using a more direct, efficient control strategy.

This configuration is a refactored ESPHome setup for version 8 and uses ESPHome's native thermostat, PID climate, time-based cover, Dallas sensors, GPIO switches, and Home Assistant imports. A short output adapter converts the PID output into bounded relay pulses because the valve uses directional relays rather than a continuous position input.

## Heating System

I use the system for room and water heating with a Panasonic Aquarea LT heat pump and a separate buffer tank. The heat pump is connected to Home Assistant via [Heishamon](https://www.tindie.com/products/thehognl/heishamon-communication-pcb/).

The buffer is required in this setup because solar thermal heating covers a large part of the annual demand, and the buffer provides the thermal store needed to integrate the solar and heat-pump circuits. In a standard Panasonic setup, the buffer temperature is kept several kelvin above the actual heating target to allow the mixing valve to work and to reduce heat-pump cycling. That is efficient for the valve, but it reduces overall system efficiency. This project instead controls the valve more directly to use the maximum available water temperature in the buffer while staying inside the heat pump's operating limits.

Hot water temperature is controlled by a 3-way mixing valve before the water is fed into the floor-heating circuit.

## Motivation

This setup has been working reliably for years with a seasonal COP around 3.8. One remaining issue was that Panasonic keeps the buffer temperature around 5-10 K above the target heating temperature. The reason is to let the mixing valve work and reduce heat-pump cycles, but that also reduces the efficiency of the heat pump. The goal is to have a control algorithm that uses the maximum available water temperature in the buffer without sacrificing stability.

This repository provides the ESPHome code for that external controller board.

## Control Design

- Aquarea is the master. Its power switch and operating mode decide whether the heating circuit may run and whether it runs in heating or cooling, and it supplies the room setpoint. The ESP32 only positions the mixing valve and the circulation pump underneath that decision.
- The `Aquarea Space Mode` diagnostic resolves the heat-pump state into a single value used elsewhere: `0` = no space conditioning, `1` = heating, `2` = cooling. The mode string is matched case-insensitively, so `Auto(heat)` and `Auto(cool)` variants are recognised.
- The local `Thermostat Climate Control` climate uses the local room probe and decides room demand. Its setpoint comes from Aquarea, and the dual-action thermostat is centred on it with a 0.5 °C separation between its low and high targets.
- During heating demand, the heat curve calculates a water target from outside temperature and the two Aquarea heat-curve points. It adds the configured curve offset and half the room-temperature error, clamps the result to 19-40 °C, and caps it at the available-water temperature.
- During cooling demand, `Cooling Water Target` provides the water target. It defaults to 22 °C and is adjustable from 19-30 °C. The 19 °C floor is a fixed installation setting, not dew-point or condensation protection. Verify it is safe for the building before enabling cooling.
- The PID climate is the only normal valve controller. Its positive output opens the valve during heating and closes it during cooling; its negative output closes the valve during heating and opens it during cooling. Each output becomes a single bounded movement of 100-2000 ms, and the time-based cover tracks the estimated position. The cover is internal, so manual Home Assistant commands cannot bypass the controller.
- The pump runs during active thermostat heating/cooling demand. Idle, off, Aquarea power-off, invalid inputs, or loss of the Home Assistant connection turns the pump off and closes the valve fully.

### Aquarea operating modes

The heat pump publishes one of nine mode strings. They map as follows:

| Aquarea mode | Controller behavior |
| --- | --- |
| `Heat`, `Heat+DHW` | Heating; heat curve drives the water target |
| `Auto(heat)`, `Auto(heat)+DHW` | Heating |
| `Cool`, `Cool+DHW` | Cooling; `Cooling Water Target` drives the water target |
| `Auto(cool)`, `Auto(cool)+DHW` | Cooling |
| `DHW` | Idle: the heat pump serves only the hot-water tank, so the pump stops and the valve closes |

Domestic hot water is therefore handled by not interfering: when the heat pump is in DHW-only mode, no heat is requested for the heating circuit, so the controller withdraws and closes the mixing valve. The same applies whenever the heat pump is switched off.

## Hardware

The mixing valve needs two relays to move the valve in each direction. Additionally, one switch is required to turn the pump on and off. Thus, a minimum of 3 relays is required.

### ESP32 board

I used this 4-relay board for the implementation: [ESP32 relay integrated 230V AC power supply and sensors](https://f1atb.fr/esp32-relay-integrated-230v-ac-power-supply-sensors/). The page explains how to prepare the board for flashing.

### Relay and sensor mapping

| Function | GPIO / address | ESPHome ID |
| --- | --- | --- |
| Valve open relay | GPIO 32 | `open_switch` |
| Valve close relay | GPIO 33 | `close_switch` |
| Pump relay | GPIO 25 | `pump_switch` |
| 1-Wire bus | GPIO 13 | `one_wire` |
| Water outlet | `0x780000005aa32228` | `water_outlet_temp` |
| Water return | `0x5B0000003467C428` | `water_return_temp` |
| Water available | `0xBB00000039495D28` | `water_available_temp` |
| Room temperature | `0x0b00000034168328` | `actual_room_temp` |

The valve travel estimates are 105 seconds open and 110 seconds closed. Relays are configured active-high and mutually interlocked. The configuration assumes de-energized relays are safe and that energizing the close relay for the full travel time is safe against the actuator's end stop. Confirm these assumptions and the open/close direction at commissioning.

The known address `0xA500000038E8DA28` is not instantiated in version 8. Version 7 defines it as address 4 but uses address 5 for the actual room sensor.

### Temperature sensors

Dallas temperature sensors are optional because the values can be read from Heishamon. This import can be realised either via MQTT or the Home Assistant protocol. I used the Home Assistant protocol.

Using local sensors allows for better accuracy because Heishamon delivers temperatures with no fractional digits only.

## Algorithm

The ESPHome implementation was inspired by [this 3-point controller for a 3-way valve post](https://community.simon42.com/t/3-punkt-regler-fuer-3-wegeventil/18762), with adjustments for this application.

### Thermostat

The [thermostat](https://esphome.io/components/climate/thermostat) mainly serves to switch the pump on and off once the temperature target is reached. When it switches off, the PID is also deactivated so it stops sending signals to the mixing valve. The thermostat view can be used in the Home Assistant UI.

### PID

A [PID](https://esphome.io/components/climate/pid/) controls the actual water temperature by sending open or close signals to the mixing valve. To convert the continuous PID output into on/off signals for the valve, the `Sigma_delta_output` is required.

## Home Assistant Inputs

The ESPHome node imports these values:

| ESPHome ID | Home Assistant entity |
| --- | --- |
| `target_room_temp` | `number.panasonic_heat_pump_main_z1_heat_request_temp` |
| `cool_request_temp` | `number.panasonic_heat_pump_main_z1_cool_request_temp` |
| `outside_temp` | `sensor.panasonic_heat_pump_main_outside_temp` |
| `heat_curve_outside_high_temp` | `number.panasonic_heat_pump_main_z1_heat_curve_outside_high_temp` |
| `heat_curve_outside_low_temp` | `number.panasonic_heat_pump_main_z1_heat_curve_outside_low_temp` |
| `heat_curve_target_high_temp` | `number.panasonic_heat_pump_main_z1_heat_curve_target_high_temp` |
| `heat_curve_target_low_temp` | `number.panasonic_heat_pump_main_z1_heat_curve_target_low_temp` |
| `aquarea_mode` | `select.panasonic_heat_pump_main_operating_mode_state` |
| `aquarea_main_power` | `switch.panasonic_heat_pump_main_heatpump_state` |

The room setpoint follows the heat pump: the heat-request value in heating and the cool-request value in cooling. If the cool-request entity is unavailable, the heat-request value is used instead. `aquarea_room_temp` and `outlet_temp` remain imported for entity compatibility but do not participate in control; the local room and outlet probes are used by the thermostat and PID respectively.

`Controller Inputs Valid` requires a connected Home Assistant client, a resolved heating or cooling mode, and finite values for every input the active mode needs. The API callbacks identify Home Assistant by the Home Assistant client-info prefix, so log viewers do not count as connections. The three control probes publish `NaN` through a native `timeout` filter if they stop updating (outlet 30 s, available water 90 s, room 180 s), so stale readings fail the same check as missing ones. In heating mode the diagnostic also verifies that the outside-temperature curve endpoints are ordered. Any failure stops the pump and closes the valve. Home Assistant imports carry no source timestamp, so a stale-but-numeric home-assistant value cannot be distinguished from an unchanged valid one; monitor entity availability in Home Assistant as well.

## Adjustable and Persistent Values

These are `config` entities in Home Assistant stored in ESP32 flash as template numbers with `restore_value`, so they survive a power cut:

| Entity | Default | Range |
| --- | --- | --- |
| `Heat Curve Kp` | 0.2 | 0-5 |
| `Heat Curve Ki` | 0 | 0-1 |
| `Heat Curve Kd` | 0.002 | 0-100 |
| `PID Deadband` | 0.5 °C | 0-3 °C |
| `Heat Curve Offset` | 0 °C | 0-10 °C |
| `Cooling Water Target` | 22 °C | 19-30 °C |

### What is restored after a power cut

- Tuning and setpoints above are restored from flash and re-applied to the PID at boot by the `apply_pid_parameters` script. The `control_parameters` in the YAML are intentionally `0`; the persisted numbers are the single source of truth.
- The thermostat restores its own mode and setpoints, and the operating mode is then re-synchronised from Aquarea, which always wins.
- The valve position is deliberately not restored. The actuator has no position feedback, so a stored estimate could be wrong after a power cut. Instead the controller always re-homes: it drives the valve fully closed at boot and again whenever the thermostat goes idle or off. Until this completes, all other valve movement is blocked.
- Relays start de-energised (`ALWAYS_OFF`), and the pump is explicitly switched off before the homing run.

## Avoiding Oscillation

The valve is an integrating actuator: each pulse changes position permanently, so the loop oscillates easily if it is driven too hard. Several mechanisms limit this, all configurable from Home Assistant:

- Thermostat hysteresis. `heat_deadband`/`heat_overrun` and the cooling pair are both `0.2 °C`, giving a 0.4 °C switching band. Both values must stay positive; a negative overrun collapses the band to zero and makes the process unstable.
- PID deadband. The `PID Deadband` is a coolant-/heating-specific deadband around the target; the controller does not move the valve while the process variable is inside it.
- Output clipping and timing. The raw PID output is limited to a small bounded range, and each pulse is limited to a single movement of 100-2000 ms to avoid overshoot and backlash.
- Cover tracking. The internal time-based cover estimates valve position so that the controller never accumulates unlimited motion without feedback.

## Lessons Learned

- It is important to release the relays of the mixing valve before engaging the opposite direction. Otherwise the relays may stick due to the electric current. A 1-second break before switching to the opposite direction is recommended.
- It took a while to get the mixing valve approaching the 0% position correctly at boot. For that, the PID component had to be switched off before the valve was homed to avoid sending conflicting drive signals while the actuator was moving to its zero position.
- The valve start-up homing run is necessary because the actuator has no feedback and stored position estimates are not safe after a power cut.

## Summary

This project provides a practical ESPHome solution for a Panasonic Aquarea heating circuit that uses a 3-way mixing valve and a buffer tank. By separating the master Aquarea control logic from the local valve and pump control loop, the system can keep the heat pump in a more efficient operating region while still staying stable and safe.
