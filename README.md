# Aquarea-Mixing-Valve-Control
ESPHome based controller for 3-way mixing valve for heating. Panasonics original algorithm maintains higher buffer temperature which reduces efficiency.  ESP32 4-relay board required. Temperature sensors are optional. 

# Hardware 
The mixing valve needs two switches to turn the valve right and left. Additionally, one switch to turn on the pump.

## ESP32
I used this 4-relay board to implement the control [LINK](https://f1atb.fr/esp32-relay-integrated-230v-ac-power-supply-sensors/). The page explains very well how to setup the board for flashing. 

## Temperature sensors 
Dallas temperature sensors are optional, as the values can be read from Heishamon. This import can be realized via MQTT or Homeassistant protocol. I used the latter. 
Using own sensors allows for better accuracy, as Heishamon delivers temperatures with no digits only. 
