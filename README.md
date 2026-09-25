AI-Powered Underground Mine Safety, Monitoring & Rescue System

Smart India Hackathon 2026 · Problem Statement 26039 Theme: Smart Automation · Category: Hardware · Organization: Government of Jharkhand Team FAB6 · 142638

Nischall S Haritas · Rishit Negi · Jeffrey Simon Chris · D Dharnesh · Theertha Santhosh · Navya S N

 Explainer video: [https://meet.google.com/quj-nbpf-eog](https://drive.google.com/drive/folders/1RHaMHPGrmijn2lN6XHrRpclHmiOnbnKA?usp=drive_link)

Problem

Underground mine accidents — toxic gas exposure, structural collapse, poor visibility, GPS-denied navigation, communication loss, and trapped or missing workers — are difficult to assess quickly and safely with manual inspection alone. Rescue teams need reconnaissance data before they enter a hazardous zone, not after.

Our approach

An F450-based quadcopter carries a sensor payload that performs autonomous aerial reconnaissance of a mine environment before human entry, reporting gas hazards, structural anomalies, thermal signatures, and obstacle data to a ground station in real time.

method ->

Toxic gas detection	MQ-2 (combustible/smoke), MQ-4 (methane), MQ-7 (carbon monoxide)

Environmental monitoring	DHT22 (temperature + humidity)

Structural integrity	MPU6050 vibration/shock sensing → Edge Impulse TinyML anomaly classifier

Obstacle avoidance	HC-SR04 ultrasonic ranging

Trapped-worker detection	MLX90614 IR thermometer + servo sector scan

Visual feed (demo)	ESP32-CAM (OV2640)

Telemetry	LoRa (SX1278, 433 MHz)

Hazard synthesis / dashboard	Gemini via Google AI Studio

Project status

This is an active student build. Status is tracked honestly rather than overstated:

 Component selection, sourcing and full bill of materials finalized
 
 Complete electrical design — power architecture, pin assignments, wiring table, power budget, 25-point design review (see docs/electrical_design.md)
 
 Physical payload layout and mounting plan
 
 Hardware assembly and bring-up — in progress
 
 Firmware — in progress
 
 TinyML structural-anomaly model — not yet trained (depends on flight vibration data)
 
 LoRa relay mesh for GPS-denied comms — planned, not yet built; current working link is tethered/point-to-point
 
Repository contents

docs/problem_statement.md — problem statement, identified gaps, and our solution mapping

docs/electrical_design.md — full electrical design documentation: power system, controller pin assignments, complete wiring table, power budget, and final design review

docs/component_list.xlsx — priced bill of materials with sourcing notes

hardware/payload_layout.html — physical mounting layout (plan view, elevation, pod detail)

presentation/ — SIH round 1 idea presentation

Hardware platform

F450 quadcopter frame · 4× A2212 1000KV brushless motors · 4× 30A ESC · 3S 2200mAh LiPo · APM 2.8 flight controller (stabilization) · ESP32-WROOM-32 (sensor payload, independent of flight control)

Team

Nischall S Haritas,	
Rishit Negi,	
Jeffrey Simon Chris,
D Dharnesh,	
Theertha Santhosh,	
Navya S N
