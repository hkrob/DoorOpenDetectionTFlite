# Door Open Detection with TFlite
Door Open Detection using DOODS, TensorflowLite and Home Assistant

# Objective
* Detect open/closed status of a door using a camera feed
* Connect to Home Assistant, enable automations based on state of the door
* Can be run without a Google Coral, i.e. on a normal CPU

# Output
* This is a simple dashboard to monitor the training / accuracy of your new model

![image](https://github.com/hkrob/DoorOpenDetectionTFlite/assets/10833368/ca70d571-b58f-4ed6-b8aa-30e82e24e890)

# Notes /  FAQ
* Can I run two add-ons using the Edge TPU at the same time? e.g. Doods + Frigate?
I tried and it appears the answer is no.
I had Frigate running and tried starting Doods with TPU, can't detect TPU.
Stop Frigate, start Doods --> All OK.
So, appears TPU is exclusive to one add-on at a time.

# Ingredients
* Camera facing the door (provides RTSP stream)
* Home Assistant - https://www.home-assistant.io/
* DOODS v2 Add-on - https://github.com/snowzach/doods2
* Tensorflow Lite model from Teachable Machine (Google) https://teachablemachine.withgoogle.com/

# Suggested Sequence
* To get all this running, I suggest going step by step and getting the following working
- [ ] Home Assistant up and running
- [ ] Sample data (images) captured
- [ ] Model created with Teachable Machine
- [ ] DOODS add-on started and running with your model
- [ ] Home Assistant sensors created
- [ ] Home Assistant Automations created
 

# Preparation
* Home Assistant is installed and working
* Sample data has been captured (see Home Assistant section below)
* Tensorflow Lite model has been created with Teachable Machine
* DOODS v2 is installed and working with your model

# Flow
* Camera captures image
* Image is processed by DOODS
* Sensors are updated in Home Assistant

# Create your model with Teachable Machine
* Using [Teachable Machine]([url](https://teachablemachine.withgoogle.com/)) You will create a model from Teachable Machine
* Create a model with two classes, open and closed

![image](https://github.com/hkrob/DoorOpenDetectionTFlite/assets/10833368/8a97e755-f8d4-4590-8282-7cb340670176)

* Use the training data (open/closed photos) you have collected with Home Assistant (Gather training data with Home Assistant)
* Make sure you have plenty of images, the more the better
* Export a Tensorflow Lite Quantized model
* Upload the model files to where you have DOODS running

![image](https://github.com/hkrob/DoorOpenDetectionTFlite/assets/10833368/d3bfb113-bb4b-4cc9-92e6-ff26bcab200f)

* You should have two files labels.txt & model.tflite
* labels.txt looks like this:
```
0 closed
1 open
```
* model.tflite is a binary file

# DOODS config
* The models from Teachable Machine have been placed in \\192.168.x.x\share\models\tflite-quant
* Add-on configuration:

```
- name: garagedoortf
  type: tflite
  modelFile: /share/models/tflite-quant/model.tflite
  labelFile: /share/models/tflite-quant/labels.txt
  hwAccel: false
  labelsStartFromZero: true
```

If DOODS has loaded your model correctly, you should see something like this in the logs
```
2023-08-09 11:31:33,274 - doods.doods - INFO - Registered detector type:tflite name:garagedoortf
2023-08-09 11:31:33,320 - uvicorn.error - INFO - Started server process [7]
2023-08-09 11:31:33,321 - uvicorn.error - INFO - Waiting for application startup.
2023-08-09 11:31:33,321 - uvicorn.error - INFO - Application startup complete.
2023-08-09 11:31:33,322 - uvicorn.error - INFO - Uvicorn running on http://0.0.0.0:8080 (Press CTRL+C to quit)
```
* Doods has a handy GUI where you can check your model is loaded and working correctly

* This is what you get if you click Detect Image

![image](https://github.com/hkrob/DoorOpenDetectionTFlite/assets/10833368/44c34c15-8860-44bf-acdf-e53c43d461b1)

* And this is Detect JSON

![image](https://github.com/hkrob/DoorOpenDetectionTFlite/assets/10833368/70085040-11e4-4f15-b496-0fd498781083)


# Home Assistant
* There are a few steps here, first is to capture the input data to train your model

## Create a Camera
* First, create a camera which is focussed on the area you want to monitor, in this case, the door
* Teachable Machine expects a square shaped input, so make sure the height/width are the same

```camera:
  - platform: proxy
    entity_id: camera.ha_prod_home_front
    mode: crop
    max_image_height : 650
    max_image_width: 650
    image_left: 102
    image_top: 1182
```
## Gather training data with Home Assistant
* I've created an automation to take a snapshot every minute
* You will want to run this for a couple of days to gather enough training data
* Also, try to open/close the door now and then so you can get samples of both
* Take these snapshots and sort them into open/closed - you will upload this to Teachable Machine later

```
alias: Timelapse - home Front
description: ""
trigger:
  - platform: time_pattern
    minutes: /1
condition: []
action:
  - service: camera.snapshot
    target:
      entity_id: camera.camera_proxy_camera_ha_prod_home_front
    data:
      filename: >-
        /media/timelapse/daily/front/{{ now().strftime("%Y%m%d") }}/front_{{    
        now().strftime("%Y%m%d-%H%M%S") }}.jpg
mode: single
```

### Training data for Closed

![front_20230802-191800](https://github.com/hkrob/DoorOpenDetectionTFlite/assets/10833368/0d51b79e-81e3-4aef-8c67-359551901e16)

### Training data for Open

![front_20230803-113600](https://github.com/hkrob/DoorOpenDetectionTFlite/assets/10833368/be8bc8f7-7612-425e-aded-80177ac2db66)

# Image Processing
* Call DOODS via the image processing integration
* The below will send the image to Doods every 10 seconds
* The labels will need to be the same as you set in Teachable Machine

```
image_processing:
  - platform: doods
    scan_interval: 10
    url: "http://192.168.2.6:8080"
    timeout: 20
    detector: garagedoortf
    source:
      - entity_id: camera.camera_proxy_camera_ha_prod_home_front
    file_out:
      - /config/www/doods-garagedoor.jpg
    labels:
      - name: closed
      - name: open
```

* Image Processing will give you a sensor that looks like this:
```
matches: 
open:
  - score: 25500
    box:
      - 0
      - 0
      - 1
      - 1
closed:
  - score: 0
    box:
      - 0
      - 0
      - 1
      - 1

summary: 
open: 1
closed: 1

total_matches: 2
process_time: 0.05038406798848882
friendly_name: Doods camera_proxy_camera_ha_prod_home_front
```

# Sensors (and more sensors)

* In order to work with the above, we will create some sensors with just the data we want
* DoodshomeGaragePersonClosed / DoodshomeGaragePersonOpen - these are the confidence scores for open/closed
* We've also created doods-garage-persondoor which is going to show open/closed/unknown based on the confidence score, you can fine-tune this to match your desired confidence level


```
template:
  - sensor:
      - name: "DoodshomeGaragePersonClosed"
        # state: {{ state_attr(image_processing.doods_camera_proxy_camera_ha_prod_home_front, 'matches') }} 
        state: "{{ state_attr('image_processing.doods_camera_proxy_camera_ha_prod_home_front', 'matches')['closed'][0]['score'] }}"
        unit_of_measurement: 'confidence'
      - name: "DoodshomeGaragePersonOpen"
        # state: {{ state_attr(image_processing.doods_camera_proxy_camera_ha_prod_home_front, 'matches') }} 
        state: "{{ state_attr('image_processing.doods_camera_proxy_camera_ha_prod_home_front', 'matches')['open'][0]['score'] }}"        
        unit_of_measurement: 'confidence'
#
  - sensor:
      - name: "doods-garage-persondoor"
        state: >
          {% if states('sensor.filtered_doodshomegaragepersonclosed')|float > 22950 %}
            closed
          {% elif states('sensor.filtered_doodshomegaragepersonopen')|float > 22950 %}
            open
          {% else %}
            unknown
          {% endif %}
        icon: >-
          {% if states('sensor.filtered_doodshomegaragepersonclosed')|float > 22950 %}
            mdi:garage-closed
          {% elif states('sensor.filtered_doodshomegaragepersonopen')|float > 22950 %}
            mdi:garage-open
          {% else %}
            mdi:help-rhombus
          {% endif %}           
#
```

* The above will create sensors that look like this:

![image](https://github.com/hkrob/DoorOpenDetectionTFlite/assets/10833368/e7e88217-f0df-4b32-8a63-fb8243745bba)

* With this model, 25500 is the maximum, i.e. 100% confidence

* I want to smooth the sensor in order to avoid spikes/troughs , so I've created yet another sensor
* For more details on options here, check the documentation https://www.home-assistant.io/integrations/filter/
```
sensor:
  - platform: filter
    name: "filtered DoodshomeGaragePersonClosed"
    entity_id: sensor.doodshomegaragepersonclosed
    filters:
      - filter: lowpass
        time_constant: 10
      - filter: time_simple_moving_average
        window_size: "00:05"
        precision: 1
  - platform: filter
    name: "filtered DoodshomeGaragePersonOpen"
    entity_id: sensor.doodshomegaragepersonopen
    filters:
      - filter: lowpass
        time_constant: 10
      - filter: time_simple_moving_average
        window_size: "00:05"
        precision: 1
```        

* This will create sensors that look like this

![image](https://github.com/hkrob/DoorOpenDetectionTFlite/assets/10833368/db98fa55-658f-443d-af2a-111123d42254)

# Notifications
* You might want notifications when something happens, here I am sending a notification via Telegram, including a snapshot of the current camera status

```
alias: Doods - Notify
description: ""
trigger:
  - platform: state
    entity_id:
      - sensor.doods_garage_persondoor
    to: null
condition: []
action:
  - service: notify.notifytelegram
    data:
      message: >-
        {{trigger.entity_id}} has changed from {{ trigger.from_state.state }} to
        {{ trigger.to_state.state }}
      title: Doods
  - service: telegram_bot.send_photo
    data:
      authentication: digest
      file: /config/www/doods-garagedoor.jpg
      caption: >-
        home {{trigger.entity_id}} has changed from {{
        trigger.from_state.state }} to {{ trigger.to_state.state }}
mode: single
```

