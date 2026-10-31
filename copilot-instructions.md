# ESP32 heating control
This project contains the instructions and guidelines for controlling heating using an ESP32 microcontroller and ESPHome integration.
The overall home automation is controlled with Home Assistant.
The leading heating control is the Panasonic heat pump with HeishaMon integration.

The ESP32 microcontroller is used to manage the 3-way mixing valve for the heating system, ensuring constant water temperature in the heating system.
device_name: "3-way-valve-heating-controller"

## 3-way mixing valve
The 3-way mixing valve is managing the water temperature in the heating system by mixing hot water from the heat pump with cooler return water from the heating system to achieve the desired temperature.

### Physical relays associated with thermostats
  open_relay_pin:  "32"
  close_relay_pin: "33"
  pump_relay_pin:  "25"

### Open/close times for 3-way valve
  open_time_s: "105s"   # time to fully open the 3-way valve
  close_time_s: "110s"  # time to fully close the 3-way valve
### General ESP32 Configuration
board: esp32dev
framework: arduino


### Values to be imported from Home Assistant
- **aquarea_room_temp**:  sensor.panasonic_heat_pump_main_z1_temp
- **target_room_temp**: number.panasonic_heat_pump_main_z1_heat_request_temp
- **outside_temp**: sensor.panasonic_heat_pump_main_outside_temp
- **outlet_temp**: sensor.panasonic_heat_pump_main_z1_water_temp
- **heat_curve_outside_high_temp**: number.panasonic_heat_pump_main_z1_heat_curve_outside_high_temp
- **heat_curve_outside_low_temp**: number.panasonic_heat_pump_main_z1_heat_curve_outside_low_temp
- **heat_curve_target_high_temp**: number.panasonic_heat_pump_main_z1_heat_curve_target_high_temp
- **heat_curve_target_low_temp**: number.panasonic_heat_pump_main_z1_heat_curve_target_low_temp

### Temperature Sensors
Measure the following four temperatures with DS18B20 sensors on the one-wire bus (GPIO 13):
  ds18b20_addr_1: "0x780000005aa32228" # water_outlet_temp
  ds18b20_addr_2: "0x5B0000003467C428" # water_return_temp
  ds18b20_addr_3: "0xBB00000039495D28" # water_available_temp
  ds18b20_addr_5: "0x0b00000034168328" # actual_room_temp

Address 4 (`0xA500000038E8DA28`) is defined in version 7 but is not used as the room sensor.

### PID Controller
Control `water_outlet_temp` to achieve a water target using one PID controller. In heating mode, calculate the target from the Aquarea heat curve, room-temperature correction, and configurable offset; cap it at the available-water temperature. In cooling mode, use a configurable target with a 22 °C default and a 19-30 °C range. The 19 °C cooling floor is not dew-point protection and must be checked for the installation.
KP, KI, KD, and the PID deadband shall be configurable from Home Assistant and persist on the ESP32. Supervised autotuning shall use bounded valve movement and provide a safe cancel/close control.

### Room Thermostat
A room thermostat named "Thermostat Climate Control" shall use the local room sensor and the Aquarea room setpoint. Aquarea power and operating mode select whether heating or cooling is available. The pump runs during active thermostat demand and turns off when idle. When idle, the valve is fully closed.
