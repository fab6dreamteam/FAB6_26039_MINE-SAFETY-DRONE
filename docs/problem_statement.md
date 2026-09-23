# Problem Statement 26039

**Title:** AI-Powered Underground Mine Safety, Monitoring and Rescue System
**Organization:** Government of Jharkhand
**Category:** Hardware
**Theme:** Smart Automation
**Problem Statement ID:** 26039

## Background (paraphrase — replace with official text)

Underground mining remains one of the most hazardous industrial environments. Workers face risks
from toxic and combustible gases, structural instability, poor visibility, and the difficulty of
locating personnel quickly during an emergency. Conventional safety monitoring relies heavily on
fixed sensors and manual inspection, both of which are slow to deploy into a zone that has just
become unsafe.

## Major issues identified

1. Toxic and hazardous gas exposure (CH₄, CO, combustible gases) with no real-time aerial detection
2. Mine structure and obstacles complicating both worker movement and rescue navigation
3. Darkness and poor visibility in unlit or damaged sections
4. No reliable GPS underground — standard positioning does not function
5. Communication loss between rescue teams and the surface
6. Trapped or missing workers who are difficult to locate quickly after an incident

## Expected solution 

Develop an Al-powered mine rescue system consisting of a rugged ground rover or a compact aerial drone capable of operating in hazardous underground mining environments. The system should provide real-time monitoring of toxic gases, temperature, humidity, and structural conditions while transmitting live video and thermal imaging data to a surface control station.

The rover and drone will assist rescue teams by exploring inaccessible areas, detecting hazards, locating trapped workers, and providing situational awareness during emergencies. The solution should improve mine safety, reduce risks to rescue personnel, and enable faster, more informed emergency response in underground mines

## Our solution — mapping to the expected solution

| Expected capability | Our implementation |
|---|---|
| Real-time gas hazard detection | MQ-2 / MQ-4 / MQ-7 array with buffered analog front-end |
| Structural hazard assessment | MPU6050 vibration/shock sensing → on-device TinyML anomaly classifier |
| Navigation without GPS | Ultrasonic obstacle sensing; LoRa relay nodes for a GPS-denied comms path (planned extension beyond current tethered/point-to-point demo link) |
| Locating trapped personnel | MLX90614 IR thermometer on a scanning servo (coarse thermal sector scan) |
| Situational awareness for rescue teams | Ground-station dashboard fusing sensor streams via Gemini / Google AI Studio |
| Rapid, low-risk deployment | F450 quadcopter platform — aerial reconnaissance ahead of human entry |

See `docs/electrical_design.md` for the full hardware implementation of each capability above,
including the design corrections made during electrical review (e.g. why MQ-7 needs a
duty-cycled heater, why the thermal sensor needs a collimator, why structural sensing and flight
stabilization are electrically and physically separate).
