# name
esphome:
  name: watermeter-new
  friendly_name: watermeter new
  
# board
esp32:
  board: esp32-c3-devkitm-1
  framework:
    type: esp-idf

    
# using deepsleep on esp32 to get the values every 4 hours ,then the values are send for 3 min every 5 sec#
deep_sleep:
  run_duration: 120s
  sleep_duration: 4h

# Enable logging
logger: 
  baud_rate: 0


# Enable Home Assistant API
api:
  encryption:
    key: !secret api_encryption_key

ota:
  - platform: esphome
    password: !secret ota_password



wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password




 
sensor:
 # Wifi signal sensor.
 - platform: wifi_signal
   name: garden_watertank_wifi
   update_interval: 600s
   unit_of_measurement: '%'
   filters:
    - lambda: |-
       if (x <= -100) {
         return 0;
       } else {
         if (x >= -50) {
           return 100;
         } else {
           return 2 * (x + 100);
         }
       }


 # Templates for calculated liter & percent
 - platform: template
   name: watertank_wolume
   id: watertank_volume
   icon: 'mdi:water'
   unit_of_measurement: 'l'
   accuracy_decimals: 0
   # this device class can show on homekit as a lux sensor ,so the value is the volume of the well
   device_class: illuminance

  

 - platform: template
   name: watertank_level
   id: watertank_level
   icon: 'mdi:water-percent'
   unit_of_measurement: '%'
   accuracy_decimals: 0
   device_class: distance


 # The actual distance sensor
 - platform: ultrasonic
   trigger_pin:
    number: GPIO6
   echo_pin:
    number: GPIO7
   # pin numbers depends on what you use 
   name: watertank_distance
   update_interval: 5s
   pulse_time: 50us
   filters:
    - filter_out: nan
    - median:
       window_size: 7
       send_every: 4
       send_first_at: 3
    - calibrate_linear:
       - 0.20 -> 1.80
       - 100.0 -> 0.0
         # the well is for me 180cm deep from the sensor and at the top 20cm from the sensor
   on_value:
    then:
     - sensor.template.publish:
        id: watertank_volume
        state: !lambda 'return x * 50;'
       # this *50 is for a well of 5000l

     - sensor.template.publish:
        id: watertank_level
        state: !lambda 'return x * 1;'
       # this always shows in percent from 0 to 100
