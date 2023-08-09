# Door Open Detection with TFlite
Door Open Detection using DOODS, TensorflowLite and Home Assistant

Objective
* Detect open/closed status of a door using a camera feed
* Connect to Home Assistant, enable automations based on state of the door

Ingredients
* Camera facing the door (provides RTSP stream)
* Home Assistant - https://www.home-assistant.io/
* DOODS v2 Add-on - https://github.com/snowzach/doods2
* Tensorflow Lite model from Teachable Machine (Google) https://teachablemachine.withgoogle.com/

Preparation
* Home Assistant is installed and working
* Sample data has been captured (see Home Assistant section below)
* Tensorflow Lite model has been created
* DOODS v2 is installed and working

Flow
* Camera captures image
* Image is processed by DOODS
* Sensors are updated in Home Assistant

DOODS config
* The models from Teachable Machine have been placed in \\192.168.x.x\share\models\tflite-quant
* Add-on configuration:

```- name: garagedoortf
  type: tflite
  modelFile: /share/models/tflite-quant/model.tflite
  labelFile: /share/models/tflite-quant/labels.txt
  hwAccel: false
  labelsStartFromZero: true
```

Home Assistant
There are a few steps here, first is to capture the input data to train your model

* First, create a camera which is focussed on the area you want to monitor, in this case, the door
* Teachable Machine expects a square shaped input, so make sure the height/width are the same
'''
camera:
  - platform: proxy
    entity_id: camera.ha_prod_northcliffe_front
    mode: crop
    max_image_height : 650
    max_image_width: 650
    image_left: 102
    image_top: 1182
'''
* I've created an automation to take a snapshot every minute
* You will want to run this for a couple of days to gather enough training data
* Also, try to open/close the door now and then so you can get samples of both
* Take these snapshots and sort them into open/closed - you will upload this to Teachable Machine later

'''
alias: Timelapse - Northcliffe Front
description: ""
trigger:
  - platform: time_pattern
    minutes: /1
condition: []
action:
  - service: camera.snapshot
    target:
      entity_id: camera.camera_proxy_camera_ha_prod_northcliffe_front
    data:
      filename: >-
        /media/timelapse/daily/front/{{ now().strftime("%Y%m%d") }}/front_{{    
        now().strftime("%Y%m%d-%H%M%S") }}.jpg
mode: single
'''

* Call DOODS via the image processing integration
** The below will send the image to Doods every 10 seconds
** The labels will need to be the same as you set in Teachable Machine

'''
image_processing:
  - platform: doods
    scan_interval: 10
    url: "http://192.168.2.6:8080"
    timeout: 20
    detector: garagedoortf
    source:
      - entity_id: camera.camera_proxy_camera_ha_prod_northcliffe_front
    file_out:
      - /config/www/doods-garagedoor.jpg
    labels:
      - name: closed
      - name: open
'''

Image Processing will give you a sensor that looks like this:
'''
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
friendly_name: Doods camera_proxy_camera_ha_prod_northcliffe_front
'''

* In order to work with the above, we will create some sensors with just the data we want
** DoodsNorthcliffeGaragePersonClosed / DoodsNorthcliffeGaragePersonOpen - these are the confidence scores for open/closed
** We've also created doods-garage-persondoor which is going to show open/closed/unknown based on the confidence score, you can fine-tune this to match your desired confidence level


'''
template:
  - sensor:
      - name: "DoodsNorthcliffeGaragePersonClosed"
        # state: {{ state_attr(image_processing.doods_camera_proxy_camera_ha_prod_northcliffe_front, 'matches') }} 
        state: "{{ state_attr('image_processing.doods_camera_proxy_camera_ha_prod_northcliffe_front', 'matches')['closed'][0]['score'] }}"
        unit_of_measurement: 'confidence'
      - name: "DoodsNorthcliffeGaragePersonOpen"
        # state: {{ state_attr(image_processing.doods_camera_proxy_camera_ha_prod_northcliffe_front, 'matches') }} 
        state: "{{ state_attr('image_processing.doods_camera_proxy_camera_ha_prod_northcliffe_front', 'matches')['open'][0]['score'] }}"        
        unit_of_measurement: 'confidence'
#
  - sensor:
      - name: "doods-garage-persondoor"
        state: >
          {% if states('sensor.filtered_doodsnorthcliffegaragepersonclosed')|float > 22950 %}
            closed
          {% elif states('sensor.filtered_doodsnorthcliffegaragepersonopen')|float > 22950 %}
            open
          {% else %}
            unknown
          {% endif %}
        icon: >-
          {% if states('sensor.filtered_doodsnorthcliffegaragepersonclosed')|float > 22950 %}
            mdi:garage-closed
          {% elif states('sensor.filtered_doodsnorthcliffegaragepersonopen')|float > 22950 %}
            mdi:garage-open
          {% else %}
            mdi:help-rhombus
          {% endif %}           
#
'''

Model from Teachable Machine
* Using [Teachable Machine]([url](https://teachablemachine.withgoogle.com/)) ...
* Create a model with two classes, open and closed
![image](https://github.com/hkrob/DoorOpenDetectionTFlite/assets/10833368/8a97e755-f8d4-4590-8282-7cb340670176)

* Use the training data you have collected with Home Assistant
* Export a Tensorflow Lite Quantized model
![image](https://github.com/hkrob/DoorOpenDetectionTFlite/assets/10833368/d3bfb113-bb4b-4cc9-92e6-ff26bcab200f)
