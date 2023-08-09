# DoorOpenDetectionTFlite
Door Open Detection using DOODS, TensorflowLite and Home Assistant

Objective
* Detect open/closed status of a door using a camera feed
* Connect to Home Assistant, enable automations based on state of the door

Ingredients
* Camera facing the door (provides RTSP stream)
* Home Assistant
* DOODS v2 Add-on - https://github.com/snowzach/doods2
* Tensorflow Lite model from Teachable Machine (Google)

Preparation
* Home Assistant is installed and working
* Sample data has been captured
* Tensorflow Lite model has been created
* DOODS v2 is installed and working

Flow
* Camera captures image
* Image is processed by DOODS
* Sensors are updated in Home Assistant

* 
