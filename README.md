# Door Open Detection with TFlite
Door Open Detection using DOODS, TensorflowLite and Home Assistant

Objective
* Detect open/closed status of a door using a camera feed
* Connect to Home Assistant, enable automations based on state of the door

Ingredients
* Camera facing the door (provides RTSP stream)
* Home Assistant
* DOODS v2 Add-on - https://github.com/snowzach/doods2
* Tensorflow Lite model from Teachable Machine (Google) https://teachablemachine.withgoogle.com/

Preparation
* Home Assistant is installed and working
* Sample data has been captured
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

Model from Teachable Machine
* Using [Teachable Machine]([url](https://teachablemachine.withgoogle.com/)) ...
* Create a model with two classes, open and closed
![image](https://github.com/hkrob/DoorOpenDetectionTFlite/assets/10833368/8a97e755-f8d4-4590-8282-7cb340670176)

* 
* Export a Tensorflow Lite Quantized model
![image](https://github.com/hkrob/DoorOpenDetectionTFlite/assets/10833368/d3bfb113-bb4b-4cc9-92e6-ff26bcab200f)
