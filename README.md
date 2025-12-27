# Aquarea-Mixing-Valve-Control
ESPHome based controller for 3-way mixing valve for floor heating. Panasonics original algorithm maintains higher buffer temperature which reduces efficiency of the heatpump.  ESP32 4-relay board required. Temperature sensors are optional. 

# Heating system 
I use for room and water heating a heatpump Panasonic Aquarea LT with a separate buffer tank. The heatpump is connected to Homeassistant via [Heishamon](https://www.tindie.com/products/thehognl/heishamon-communication-pcb/).

The buffer is required in this heating system,  as I am also using solar thermal heating. Solar thermal heating is sufficient heat supply for more than six months of the year. 

Hot water temperature is controlled by a 3-way mixing valve before being fed into floor heating.

# Motivation 
This setup has been  working fine for two years now. SCOP around 3.8.
But I didn't solve one issue: Panasonic keeps the buffer temperature around 5-10 K higher than the target temperature for the heating. Reason for that is to allow the mixing valve to work and to reduce cycles of the heatpump. But this also reduces efficiency of the heatpump. My wish is to have a control algorithm that uses the maximum available water temperature in the buffer. This is why I decided to build this external controller board. This repository provides the code for ESPHome.  

# Hardware 
The mixing valve needs two relays to turn the valve right and left. Additionally, one switch to turn on the pump. Thus, a minimum of 3 relays are required. 

## ESP32
I used this 4-relay board to implement the control [LINK](https://f1atb.fr/esp32-relay-integrated-230v-ac-power-supply-sensors/). The page explains very well how to setup the board for flashing. 

## Temperature sensors 
Dallas temperature sensors are optional, as the values can be read from Heishamon. This import can be realized via MQTT or Homeassistant protocol. I used the latter. 
Using own sensors allows for better accuracy, as Heishamon delivers temperatures with no digits only. 

# Algorithm 
The ESPhome implementation was inspired by [this](https://community.simon42.com/t/3-punkt-regler-fuer-3-wegeventil/18762) post but slightly modified. 

## Thermostat 
The [thermostat](https://esphome.io/components/climate/thermostat) merely serves to switch on and off the pump when temperature is reached. When switched off, the PID is deactivated to stop it from sending signals to the mixing valve. The visual of the thermostat can be used in the UI of Homeassistant. 

## PID
A [PID](https://esphome.io/components/climate/pid/) controls the actual water temperature by sending open or closing signals to the mixing valve. To convert the continuous output of the PID to on-off signals to the mixing valve,  the Sigma_delta_output is required. 


## Lessons learned 
It is important to release the relais of the mixing valve before engaging the opposite direction. Otherwise the relais may stick due to the electric current. I implemented a 1s break before switching to opposite direction. 

It took me a while to get the mixing valve approaching the 0% position at boot of the device. For that, I had to switch off the PID-component in order not to send signals to the two relais before approaching its zero position. 
