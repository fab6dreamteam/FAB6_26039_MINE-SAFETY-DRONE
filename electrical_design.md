# SIH 2026 — PS 26039
# AI-Powered Underground Mine Safety, Monitoring & Rescue System
## Electrical Design Documentation — Payload Avionics

**Team:** FAB6 (SIH_CU_010) · **Platform:** F450 quadcopter · **Rev:** 1.0
**Scope:** Sensor/compute payload electrical design. The flight stack (FC, ESCs, motors, RC link) is treated as an existing, separate electrical domain and appears only as a power source and a mass/current constraint.

---

# 1. DESIGN VALIDATION

Findings from reviewing the previously agreed component set. Nine required checks, plus the corrections applied before the schematic was drawn.

## 1.1 Electrical incompatibilities

| # | Finding | Severity | Resolution applied |
|---|---------|----------|--------------------|
| V1 | **MQ-2 / MQ-4 / MQ-7 analog outputs swing to ~5.0 V.** ESP32 GPIO absolute max is 3.6 V. Direct connection destroys the pin. | **Critical** | Resistive divider ÷2 on every MQ AOUT → 2.50 V max, then a unity-gain buffer. §5, §6 Sheet 3. |
| V2 | **HC-SR04 ECHO is a 5 V push-pull output.** Direct connection to GPIO is out of spec. | **Critical** | 2.2 kΩ / 3.3 kΩ divider → 3.00 V. §6 Sheet 5. |
| V3 | **SX1278 (Ra-02) is 3.3 V only — no pin is 5 V tolerant**, including NSS/SCK/MOSI/RST. | **Critical** | Entire SPI bus runs at 3.3 V (ESP32 is natively 3.3 V, so no shifter needed). Module powered from dedicated `+3V3_AUX`, never 5 V. §6 Sheet 6. |
| V4 | **DHT22 powered at 5 V would present a 5 V data line** (open-drain pulled to VCC). | Major | DHT22 powered from `+3V3_AUX` with a 10 kΩ pull-up to the same 3.3 V rail. Datasheet range is 3.3–6 V, so 3.3 V is legal and removes the shifter entirely. §6 Sheet 5. |
| V5 | **MPU6050 breakout (GY-521) has an on-board LDO** whose dropout is not met if fed 3.3 V, and 5 V feed puts its I²C pull-ups at 3.3 V anyway (they tie to the LDO output). | Minor | Powered from `+3V3_AUX`; the LDO passes through at ≈3.3 V, inside the MPU6050's 2.375–3.46 V VDD window. |

## 1.2 Voltage-level problems

All resolved above. Post-correction, **no node in the payload exceeds 3.3 V at any ESP32 pin**. Verified in §9.

## 1.3 GPIO conflicts

| # | Finding | Resolution |
|---|---------|------------|
| G1 | **ADC2 is unusable whenever the Wi-Fi radio is initialised** on ESP32 — a silent, intermittent failure that would have made gas readings randomly return 0. GPIO 25/26/27/14/12/13/4/0/2/15 are all ADC2. | **All three gas sensors moved to ADC1 only** (GPIO 34, 35, 32). ADC2 pins are used for digital functions only. |
| G2 | **GPIO12 (MTDI) is a strapping pin** — if pulled high at boot it sets the flash regulator to 1.8 V and the board fails to boot. | GPIO12 left unconnected and explicitly marked DO-NOT-USE on the schematic. |
| G3 | **GPIO5 (LoRa NSS) is a strapping pin that must be HIGH at boot.** A floating or low NSS at reset causes boot-mode issues. | 10 kΩ pull-up to `+3V3_AUX` on NSS. Also correctly deselects the radio at power-up. |
| G4 | GPIO6–11 are bonded to the SPI flash. | Excluded from the map. |
| G5 | GPIO34–39 are **input-only, with no internal pull-ups**. | Used only for ADC inputs, which is exactly their intended role. |

## 1.4 Bus conflicts (I²C / SPI / UART)

| Bus | Devices | Conflict? |
|-----|---------|-----------|
| I²C0 (GPIO21/22) | MPU6050, MLX90614 | None. Both are I²C; MLX90614 is SMBus-compatible. Bus clocked at **100 kHz** (MLX90614 limit). |
| VSPI (GPIO18/19/23/5) | SX1278 only | None. Sole device on the bus. |
| UART0 (GPIO1/3) | USB-serial / flashing | Reserved. Not used for payload data. |
| UART2 (GPIO16/17) | Tether transceiver | None. |

## 1.5 I²C address conflicts

| Device | Address | Conflict |
|--------|---------|----------|
| MPU6050 (AD0 → GND) | `0x68` | — |
| MLX90614 (factory default) | `0x5A` | — |

**No conflict in the current build.** Your prompt referenced XSHUT sequencing — that technique belongs to VL53L0X time-of-flight sensors, which are not in this design. Your obstacle sensor is an **HC-SR04, which is not an I²C device at all** (discrete TRIG/ECHO), so no address arbitration is required for it.

*Forward-looking:* if a second MLX90614 is ever added, both would answer at `0x5A`. Two valid fixes, documented for completeness:
1. **EEPROM re-address** — the MLX90614's SMBus address lives in EEPROM word `0x2E`. Connect one sensor alone, write a new address (e.g. `0x5B`), power-cycle, then bus them together. This is the preferred solution (zero extra hardware).
2. **TCA9548A 1-to-8 I²C multiplexer** — if more than two identical devices are ever needed.

If multiple HC-SR04s are added, they need separate TRIG/ECHO pairs **and** time-multiplexed firing (≥60 ms apart) to prevent acoustic crosstalk between units.

## 1.6 Power-budget problems

| # | Finding | Resolution |
|---|---------|------------|
| P1 | **A single 5 V/2 A BEC feeding MQ heaters + servo + VTX + ESP32 will brown out.** Peak demand is 1.67 A on logic alone; the 5.8 GHz VTX adds 300–400 mA of continuous, switching-noisy load. | **Split into two supplies.** BEC1 (5 V/3 A) = sensors + logic + servo. BEC2 (5 V/2 A) or direct-from-battery = FPV camera + VTX only. Keeps the video transmitter's noise and current pulses off the analog sensor rail. |
| P2 | **SG90 stall current is ~700 mA** and its inrush will drop the shared 5 V rail and reset the ESP32. | 1000 µF bulk capacitor at the servo's power pins + its own supply branch back to the BEC (star ground). |
| P3 | **LoRa TX draws a 120 mA pulse at +20 dBm.** On a shared rail this causes brownout-induced packet loss. | Dedicated `+3V3_AUX` LDO for the radio + sensors, separate from the ESP32 DevKit's own on-board regulator. 10 µF + 100 nF at the module. |

Full numbers in §7. Verdict: **the payload is well within the F450's electrical and thrust budget**, with a ~3 min flight-time cost.

## 1.7 Missing supporting components

Components absent from the earlier parts list that the design genuinely requires:

| Part | Qty | Purpose |
|------|-----|---------|
| Resistors 100 kΩ (1 %) | 6 | MQ analog dividers |
| Resistors 2.2 kΩ / 3.3 kΩ | 1 ea | HC-SR04 ECHO divider |
| Resistor 10 kΩ | 3 | DHT22 pull-up, MOSFET gate pull-down, LoRa NSS pull-up |
| Resistor 100 Ω | 1 | MOSFET gate series |
| Resistors 1 kΩ | 3 | ADC anti-alias |
| **MCP6004** quad rail-to-rail op-amp (DIP-14) | 1 | Unity-gain buffers for the 3 gas channels — see §5 |
| **IRLZ44N** logic-level N-MOSFET | 1 | MQ-7 heater cycling — see §1.9(d) |
| **AMS1117-3.3** regulator module | 1 | Dedicated `+3V3_AUX` rail |
| **5 V/3 A switching UBEC** | 1 | BEC1, payload rail |
| **5 V/2 A switching UBEC** | 1 | BEC2, video rail (omit if VTX accepts 7–24 V) |
| **MAX3491** full-duplex RS-422 transceiver | 2 | Tether link — see §1.9(i) |
| Polyfuse 2 A (MF-R200) | 1 | Payload branch protection |
| **IRF4905** P-MOSFET | 1 | Reverse-polarity protection |
| TVS SMBJ15A | 1 | Battery-side transient clamp |
| Caps: 470 µF/25 V, 1000 µF/10 V, 2× 220 µF/10 V, 100 µF/10 V, 10 µF, 8× 100 nF | — | Bulk + decoupling |
| Ferrite bead | 1 | VTX supply filtering |
| 433 MHz antenna (SMA or λ/4 = 17.3 cm wire) | 2 | LoRa — **must be fitted before power-up** |

## 1.8 Parts requiring conditioning circuitry

| Component | Needs | Value / part |
|-----------|-------|--------------|
| MQ-2 AOUT | Divider + buffer | 100 k/100 k → MCP6004 |
| MQ-4 AOUT | Divider + buffer | 100 k/100 k → MCP6004 |
| MQ-7 AOUT | Divider + buffer | 100 k/100 k → MCP6004 |
| MQ-7 heater | **MOSFET low-side switch + PWM** | IRLZ44N, 100 Ω gate, 10 k pull-down |
| HC-SR04 ECHO | Divider | 2.2 k / 3.3 k → 3.00 V |
| DHT22 DATA | Pull-up | 10 kΩ → +3V3_AUX |
| I²C SDA/SCL | Pull-ups | 4.7 kΩ (already fitted on GY-521 and GY-906 breakouts — see §9) |
| LoRa NSS | Pull-up | 10 kΩ → +3V3_AUX (also satisfies GPIO5 strapping) |
| SG90 | Bulk decoupling | 1000 µF |
| Battery input | Fuse + reverse-polarity + TVS | MF-R200, IRF4905, SMBJ15A |

## 1.9 Corrections to previously assumed functionality

These are places where the earlier project notes assumed behaviour the hardware does not actually have. Each is corrected in the schematic.

**(a) MPU6050 is NOT the flight-stabilisation IMU.**
Earlier notes described it as serving both flight stabilisation and structural sensing. It cannot do both. The F450's flight controller has its own IMU and closes the attitude loop internally; our MPU6050 is not in that loop and never will be. It is a **payload-only vibration/shock sensor**.
*Consequence for the build:* the two roles have opposite mounting requirements. A flight IMU is soft-mounted on damping gel to reject frame vibration. Ours must be **rigidly bolted to the frame arm** — frame vibration is the signal we are trying to measure. Mount it hard, close to the airframe structure, not on the payload pod's foam.

**(b) The FPV camera does not connect to the ESP32 — at all.**
There is no electrical path between them, and there should not be. The camera is analog CVBS into a 5.8 GHz video transmitter; that is a self-contained RF chain from camera → VTX → goggles/receiver. The ESP32 has no composite-video input peripheral. The only way to get video into an ESP32 is the DVP parallel bus on an ESP32-CAM/OV2640, which consumes ~13 GPIOs and would collide with the I²C bus, the SPI bus, and the ADC channels simultaneously — the sensor payload and camera cannot coexist on one ESP32.
*Design decision:* keep the two domains fully independent. They share only the battery and the airframe. This is shown correctly on Sheet 8 and is also the honest framing for the presentation.

**(c) The MLX90614 does not produce a thermal image, and its default variant has a 90° field of view — which severely limits the servo-scan concept as originally described.**
The affordable GY-906 breakout carries the **MLX90614ESF-BAA, FOV ≈ 90°**. Sweeping a 90° sensor across a scene gives almost no spatial discrimination, because adjacent scan positions see almost the same scene.

Worse, the physics of a wide FOV limits *detection range*, not just resolution. The sensor reports the area-weighted radiometric average of everything in its cone:

```
T_meas⁴ ≈ f·T_target⁴ + (1−f)·T_background⁴        f = target's fraction of FOV area
```

Taking a human torso as 0.5 m², skin at 34 °C (307 K), mine wall at 20 °C (293 K):

| Config | Dist | FOV footprint | f | Measured rise over background |
|--------|------|---------------|---|-------------------------------|
| Bare BAA, 90° | 1 m | ⌀2.0 m, 3.14 m² | 0.159 | **+2.4 °C** — clearly detectable |
| Bare BAA, 90° | 2 m | ⌀4.0 m, 12.6 m² | 0.040 | +0.6 °C — marginal vs. noise |
| Bare BAA, 90° | 3 m | ⌀6.0 m, 28.3 m² | 0.018 | +0.3 °C — **not reliably detectable** |
| **+ collimator, 30°** | 3 m | ⌀1.61 m, 2.03 m² | 0.246 | **+3.6 °C** — clearly detectable |

*Correction applied:* fit a **mechanical collimator** — a 6 mm ID × 17–20 mm long tube over the sensor can, interior matte black. Half-angle `θ = atan(d / 2L) = atan(6 / 40) ≈ 8.5°`, giving roughly a 17–30° effective FOV once the can's own aperture is accounted for. This extends usable trapped-person detection from ~1 m to ~3 m, as computed above.
*Trade-off to state honestly:* a blackened tube absorbs off-axis IR and then re-radiates at its own temperature, adding a thermal offset. Compensate in firmware using the MLX90614's built-in ambient (`Ta`) register, which tracks the tube temperature closely, and calibrate the offset at the start of each flight.
*Presentation wording:* this is a **coarse thermal sector scan** (5–7 discrete positions across ~120°), not thermal imaging. That phrasing is both accurate and defensible under questioning.

**(d) MQ-7 cannot be run on a constant 5 V heater.**
The MQ-7 CO sensor requires a **150 s dual-temperature duty cycle**: 60 s at 5.0 V (high-temperature clean/purge phase), then 90 s at 1.4 V (low-temperature sensing phase), with the reading taken at the end of the low phase. Run at constant 5 V it does not measure CO meaningfully. This needs a hardware switch, which was not in the earlier parts list.
*Correction applied:* low-side IRLZ44N MOSFET on the heater, PWM-driven from GPIO33.

**The duty cycle is not 28 %.** Heater temperature follows *power*, so thermal equivalence is set by RMS voltage, not average voltage:

```
V_rms = V_supply × √D   →   1.4 V = 5.0 V × √D   →   D = (1.4/5)² = 7.84 %
```

Cross-check against the datasheet, with heater R_H = 33 Ω ±5 %:

```
High phase : P = 5.0²/33          = 758 mW  for 60 s
Low  phase : P = 1.4²/33          =  59 mW  for 90 s
Cycle mean : (758×60 + 59×90)/150 = 339 mW
```

The MQ-7 datasheet specifies heating consumption **< 350 mW**, which this matches. A 28 % duty would give 212 mW in the low phase and a 335 mW→492 mW cycle mean — out of spec and thermally wrong. The 7.84 % figure is the correct one.

**(e) MQ sensors need burn-in and preheat.**
A new MQ element needs **24–48 h of continuous powered burn-in** before its baseline stabilises, and **3–5 minutes of preheat** on every power-up before readings are valid. Firmware must suppress/flag readings during preheat. Plan the burn-in to happen days before the demo, not on demo day.

**(f) MQ sensors are indicative, not certified instruments.**
They are non-selective resistive elements with significant humidity and temperature cross-sensitivity, they require oxygen to be present, and they are not DGMS-certifiable gas detection. Describe the output as **relative hazard indication and trend detection**, not calibrated ppm compliance measurement. This matters for the credibility framing already agreed for the presentation.

**(g) Thermal cross-talk between the MQ heaters and the temperature sensors.**
Three MQ heaters dissipate ~1.6 W continuously inside the payload pod. That will bias a nearby DHT22 by several degrees and can appear in the MLX90614's ambient channel.
*Correction applied:* DHT22 mounted ≥60 mm from the MQ cluster and positioned in prop-wash; MLX90614 mounted on the opposite face of the pod with its collimator's line of sight clear of the MQ bodies.

**(h) LoRa band selection for India.**
The Ra-02 is an **SX1278 (~433 MHz)**. For Indian low-power wireless, 865–867 MHz is the band normally used for LPWAN; 433 MHz is also used for delicensed low-power devices. Verify the current WPC delicensing notification before any deployment outside a lab. The Ra-02 is fine for the prototype; a production build would likely move to the **Ra-01H (SX1276, 868 MHz)**, which is pin-compatible on this schematic — no board change required.

**(i) The tether: plain UART will not survive the cable, and CAT5 is far too heavy to fly.**
Raw 3.3 V single-ended UART over more than ~2 m of unshielded wire, next to four ESCs switching tens of amps, will corrupt. And a 15 m run of CAT5 weighs ~570 g — more than three times the entire rest of the payload.
*Correction applied:*
- **Signalling:** MAX3491 full-duplex RS-422 transceivers at both ends. Differential, ground-referenced, good for 100 m+. Full-duplex was chosen specifically so that **no DE/RE direction-control GPIO is needed** (DE tied high, RE̅ tied low permanently) — saving two ESP32 pins and eliminating bus-contention risk at boot.
- **Cable:** 4× 30 AWG conductors as two twisted pairs, plus one ground conductor. ≈2.8 g/m → **15 m ≈ 45 g**. Fly with a payout spool and keep the tether slack; 15 m is the practical limit for an F450.
- **Fallback:** for a bench demo under 2 m, the MAX3491s can be omitted and GPIO17/16 wired straight across. The schematic supports both.

---

# 2. FINAL SYSTEM ARCHITECTURE

Three electrically separate domains sharing one battery and one airframe.

```
┌──────────────────────── F450 AIRFRAME ────────────────────────┐
│                                                               │
│  DOMAIN A — FLIGHT (pre-existing, untouched)                  │
│    3S LiPo ─► PDB ─► 4× 30 A ESC ─► 4× 2212/920 KV            │
│                  └─► Flight Controller (own IMU) ─► RC RX     │
│                                                               │
│  DOMAIN B — SENSOR / COMPUTE PAYLOAD  (this document)         │
│    3S LiPo ─► fuse ─► rev-prot ─► BEC1 5 V/3 A                │
│                                    ├─► ESP32-WROOM-32         │
│                                    ├─► MQ-2 / MQ-4 / MQ-7     │
│                                    ├─► HC-SR04                │
│                                    ├─► SG90 scan servo        │
│                                    └─► AMS1117 ─► +3V3_AUX    │
│                                                 ├─ LoRa SX1278│
│                                                 ├─ MPU6050    │
│                                                 ├─ MLX90614   │
│                                                 ├─ DHT22      │
│                                                 ├─ MCP6004    │
│                                                 └─ MAX3491    │
│                                                               │
│  DOMAIN C — ANALOG VIDEO (independent RF chain)               │
│    3S LiPo ─► BEC2 5 V/2 A ─► FPV camera ─(CVBS)─► 5.8 GHz VTX│
│                                                               │
└───────────────────────────────────────────────────────────────┘
        │ RS-422 tether (wired)          ((( 433 MHz )))   ((( 5.8 GHz )))
        ▼                                      ▼                  ▼
  ┌──────────────────────────────┐     ┌─────────────┐    ┌──────────────┐
  │ GROUND STATION               │     │ LoRa relay  │    │ 5.8 GHz RX + │
  │ ESP32 #2 + MAX3491 + SX1278  │◄────┤ node (PLAN) │    │ goggles/     │
  │   └─USB─► Laptop             │     └─────────────┘    │ monitor      │
  │            └─ Python bridge  │                        └──────────────┘
  │               └─ Gemini /    │
  │                  AI Studio   │
  │                  dashboard   │
  └──────────────────────────────┘
```

**Why the domains are separated:** a fault anywhere in Domain B or C cannot bring down Domain A. The payload taps the battery through its own fuse and reverse-polarity MOSFET, so a payload short blows a 2 A polyfuse instead of browning out the flight controller mid-flight. This is standard practice on any instrumented UAV and is worth stating to a judge.

---

# 3. POWER ARCHITECTURE

## 3.1 Sheet 1 — Power distribution

```
  3S LiPo  11.1 V nom (12.6 V full / 9.9 V cutoff)
  2200–3000 mAh, 25–35 C
      │
      ├──────────────────► [F450 PDB] ──► 4× 30 A ESC ──► motors
      │                                └─► FC + RC RX        (DOMAIN A)
      │
      │  ── PAYLOAD BRANCH — separate pigtail, parallel tap ──
      ▼
   F1  PolyFuse 2 A  (MF-R200)
      │
      ▼
   Q1  IRF4905  P-ch reverse-polarity  (RDS(on) 0.02 Ω)
       S ── VBAT_IN          Gate held at GND via R1 100 kΩ.
       D ── VBAT_PROT        VGS = −11.1 V, inside −20 V limit.
       G ── R1 100k ── GND   Loss @1.5 A = 1.5²×0.02 = 45 mW.
      │
   VBAT_PROT ──┬──────────┬────────────────────┬──────────────────┐
               │          │                    │                  │
              D1         C1                    │                  │
           SMBJ15A    470 µF/25 V              │                  │
             TVS       low-ESR                 │                  │
               │          │                    │                  │
              GND        GND                   ▼                  ▼
                                   ┌───────────────────┐  ┌───────────────────┐
                                   │ BEC1  5 V / 3 A   │  │ BEC2  5 V / 2 A   │
                                   │ switching (MP1584 │  │ (OMIT if the AIO  │
                                   │ / XL4015 UBEC)    │  │  VTX accepts      │
                                   └─────────┬─────────┘  │  7–24 V direct)   │
                                             │            └─────────┬─────────┘
                                        +5V_SYS                +5V_VID
                                             │                     │
                             C2 220 µF ──────┤         C10 220 µF ─┤
                             C3 100 nF ──────┤              FB1 ───┤ ferrite
                                             │                     │
                                             │                     ├─► FPV CAMERA
                                             │                     └─► 5.8 G VTX
                                             │                        (DOMAIN C)
      ┌──────────┬──────────┬────────────┬───┴────────┬──────────────────┐
      │          │          │            │            │                  │
      ▼          ▼          ▼            ▼            ▼                  ▼
  ESP32 VIN   MQ-2 VCC  MQ-4 VCC   MQ-7 H+/VCC   HC-SR04 VCC      U2 AMS1117-3.3
  (+C11                                           (C9 100 nF)      IN ──┬── OUT
   100 µF                                                                │
   +100 nF)                              SG90 V+ ◄──┬── C8 1000 µF   C6 10 µF
                                                    │                C7 100 nF
                                                   GND                    │
                                                                          ▼
                                                                     +3V3_AUX
        ┌────────────┬─────────────┬────────────┬────────────┬────────────┐
        ▼            ▼             ▼            ▼            ▼            ▼
   LoRa Ra-02    MPU6050      MLX90614       DHT22       MCP6004     MAX3491
   C4 10 µF      (GY-521)     (GY-906)     +R 10k PU     C12 100nF   C13 100nF
   C5 100 nF     C14 100nF    C15 100nF
```

## 3.2 Two-rail rule — do not violate

`ESP32 DevKit 3V3 pin` and `+3V3_AUX` are **two different regulators**. They must **never be tied together** — paralleling two regulators causes one to sink the other's output.

They share **GND only**. Both sit at 3.3 V ±3 %, so logic levels between the ESP32 and the AUX-powered devices are fully compatible. Both derive from `+5V_SYS`, so they power up and down together — no back-powering through GPIO protection diodes.

## 3.3 Regulator sizing

| Rail | Source | Load (typ / peak) | Device rating | Margin |
|------|--------|-------------------|---------------|--------|
| `+5V_SYS` | BEC1 switching | 712 mA / 1666 mA | 3000 mA | **1.8×** at peak |
| `+5V_VID` | BEC2 switching | 400 mA / 450 mA | 2000 mA | 4.4× |
| `+3V3_AUX` | AMS1117-3.3 LDO | 43 mA / 151 mA | 800 mA | 5.3× |

AMS1117-3.3 dissipation at peak: `(5.0 − 3.3) × 0.151 = 0.26 W`. SOT-223 θJA ≈ 60 °C/W → ΔT ≈ 16 °C. **No heatsink required.**

An LDO (not a buck) was deliberately chosen for `+3V3_AUX` because this rail feeds the MCP6004 analog buffers and the MLX90614 — a switching regulator's ripple would inject noise directly into the gas-sensing and IR measurement paths.

---

# 4. CONTROLLER PIN ASSIGNMENT

## 4.1 Sheet 2 — ESP32-WROOM-32 (DevKit V1, 30-pin)

```
                        ┌──────────────────────────────────┐
        +5V_SYS ───────►│ VIN                        3V3   │──► ESP32 core only
                        │                                  │    (NOT +3V3_AUX)
            GND ───────►│ GND                         EN   │──┬─ R 10k ─► 3V3
                        │                                  │  └─ C 100nF ─► GND
                        │        ── ADC1 (Wi-Fi safe) ──   │
  MQ2_ADC  0–2.50 V ───►│ GPIO34  ADC1_CH6   [input only]  │
  MQ4_ADC  0–2.50 V ───►│ GPIO35  ADC1_CH7   [input only]  │
  MQ7_ADC  0–2.50 V ───►│ GPIO32  ADC1_CH4                 │
                        │                                  │
                        │        ── digital out ──         │
  MQ7_HEAT_PWM ◄────────│ GPIO33  LEDC ch0, 1 kHz          │──► Q2 gate
  US_TRIG      ◄────────│ GPIO25  10 µs pulse              │──► HC-SR04 TRIG
  SERVO_PWM    ◄────────│ GPIO4   LEDC ch1, 50 Hz          │──► SG90 signal
  STATUS_LED   ◄────────│ GPIO2   on-board LED             │
                        │                                  │
                        │        ── digital in ──          │
  US_ECHO      ────────►│ GPIO13  RMT capture, 3.00 V      │◄── divider
  LORA_DIO0    ────────►│ GPIO26  RX-done IRQ              │◄── SX1278 DIO0
                        │                                  │
                        │        ── bidirectional ──       │
  DHT_DATA     ◄───────►│ GPIO27  1-wire, 10k PU           │◄─► DHT22
                        │                                  │
                        │        ── I²C0 @ 100 kHz ──      │
  I2C_SDA      ◄───────►│ GPIO21                           │
  I2C_SCL      ◄────────│ GPIO22                           │
                        │                                  │
                        │        ── VSPI @ 8 MHz ──        │
  SPI_SCK      ◄────────│ GPIO18                           │
  SPI_MISO     ────────►│ GPIO19                           │
  SPI_MOSI     ◄────────│ GPIO23                           │
  LORA_NSS     ◄────────│ GPIO5   [strap: HIGH @ boot]     │── R 10k ─► +3V3_AUX
  LORA_RST     ◄────────│ GPIO14                           │
                        │                                  │
                        │        ── UART2 @ 115200 ──      │
  TETHER_TX    ◄────────│ GPIO17  U2TXD                    │──► MAX3491 DI
  TETHER_RX    ────────►│ GPIO16  U2RXD                    │◄── MAX3491 RO
                        │                                  │
                        │  ══ RESERVED — DO NOT CONNECT ══ │
                        │ GPIO12  MTDI strap — MUST be LOW │
                        │         at boot. LEAVE OPEN.     │
                        │ GPIO0   BOOT button              │
                        │ GPIO1/3 UART0 — USB / flashing   │
                        │ GPIO15  unused (strap)           │
                        │ GPIO36/39  spare ADC1            │
                        │ GPIO6–11  bonded to SPI flash    │
                        └──────────────────────────────────┘

   C11 100 µF ∥ C16 100 nF across VIN–GND, ≤10 mm from the board.
```

## 4.2 GPIO budget

19 of 26 available GPIOs used. **7 free** (GPIO0, 1, 3, 12, 15, 36, 39) — two of which (36, 39) are spare ADC1 channels available for a fourth gas sensor or a battery-voltage monitor without any rework.

## 4.3 Firmware constraints imposed by the hardware

| Constraint | Reason |
|------------|--------|
| ADC1 only, `ADC_ATTEN_DB_11`, 12-bit | ADC2 is dead whenever Wi-Fi is up |
| I²C clock ≤ 100 kHz | MLX90614 SMBus maximum |
| Call `esp_wifi_stop()` on the drone node | Saves ~70 mA, removes TX current pulses, removes 2.4 GHz noise. Comms go over tether/LoRa — Wi-Fi is useless underground anyway. |
| HC-SR04 echo timed with the **RMT** peripheral, not `pulseIn()` | RMT is hardware-timed; `pulseIn()` is interrupt-blocked by the radio stack and gives jittery distance |
| MQ-7 sample only in the final 2 s of the 90 s low phase | Datasheet sensing window |
| Suppress all MQ readings for the first 300 s after boot | Preheat |
| Attach LoRa antenna before applying power | Unterminated PA output damages the SX1278 |

---

# 5. ANALOG FRONT-END DESIGN

## 5.1 Divider calculation

MQ module AOUT swings 0 V → VCC (5.0 V). Target ≤ 2.50 V for ADC1 at 11 dB attenuation (full scale ≈ 3.1 V, linear region comfortably covers 2.5 V).

```
   Vadc = Vaout × R2 / (R1 + R2)
   R1 = R2 = 100 kΩ (1 %)   →   Vadc = Vaout × 0.500
   Vadc(max) = 5.00 × 0.500 = 2.50 V          ✔ within 3.3 V, within ADC range
```

**Why 100 kΩ and not 10 kΩ:** the MQ module's AOUT node is the junction of the sensing element R_S and the module's load resistor R_L (typically 1–10 kΩ). A 10 k/10 k divider presents 20 kΩ across that node and materially shifts R_L, distorting the R_S/R_0 ratio the gas calculation depends on. At 100 k/100 k the loading is 200 kΩ — negligible against R_L.

## 5.2 Why a buffer is required

100 k/100 k gives a source impedance of `100k ∥ 100k = 50 kΩ`. The ESP32's SAR ADC charges an internal sampling capacitor and needs a **source impedance below ~10 kΩ** to settle within the sampling window; 50 kΩ produces large, input-dependent errors.

Resolution: **MCP6004 quad rail-to-rail op-amp as a unity-gain follower** on each channel. Input impedance ~10¹² Ω (no loading on the divider), output impedance <10 Ω.

```
                     DIVIDER                    UNITY-GAIN BUFFER
   MQ-2 AOUT ──┬── R 100k ──┬── R 100k ── GND
   (0–5.0 V)   │            │
               │            │   0–2.50 V
               │            └──────────────────►│+\
               │                                 │  >──┬── R_f 1k ──┬──► GPIO34
               │                              ┌─►│−/                │
               │                              │                  C_f 100nF
               │                              └──────────────────┤
               │                                                GND
```

- **Supply:** MCP6004 VDD = `+3V3_AUX`, VSS = GND, 100 nF at pin 4. Rail-to-rail input/output; max input 2.50 V is well inside the 0–3.3 V range.
- **Anti-alias RC:** `R_f = 1 kΩ, C_f = 100 nF → f_c = 1/(2π·1k·100n) = 1.59 kHz`. Also drops source impedance seen by the ADC to 1 kΩ. ✔
- **Unused 4th amplifier (U3D):** must not float — wire as a follower with `+` tied to GND.

## 5.3 Safety check

| Node | Max voltage | ESP32 limit | Verdict |
|------|-------------|-------------|---------|
| MQ-2/4/7 AOUT raw | 5.00 V | — | Never reaches a GPIO |
| Divider output | 2.50 V | 3.6 V abs max | ✔ 1.44× margin |
| Buffer output | 2.50 V (clamped by 3.3 V rail) | 3.6 V | ✔ Cannot exceed 3.3 V even under fault |
| HC-SR04 ECHO raw | 5.00 V | — | Never reaches a GPIO |
| ECHO divider output | 3.00 V | 3.6 V | ✔ Above V_IH 2.48 V, below limit |

The buffer provides a second layer of protection: because it is powered from 3.3 V, its output **physically cannot exceed 3.3 V** even if a divider resistor fails open.

---

# 6. FINAL ELECTRICAL SCHEMATIC

## Sheet 3 — Gas sensor front-end and MQ-7 heater driver

```
 ══ ANALOG CHANNELS ═══════════════════════════════════════════════════════

  MQ-2 MODULE                    ÷2 DIVIDER            MCP6004 (U3A)
 ┌───────────────┐            R2 100k   R3 100k       ┌──────────┐
 │ VCC ──────────┼◄── +5V_SYS   ┌─/\/\─┬─/\/\─┐       │ 3 ┌──┐   │
 │ GND ──────────┼──► GND       │      │      │       └──►│+ \  1│
 │ AOUT ─────────┼──────────────┘      │     GND          │   >──┼─┬─R8 1k─┬─► GPIO34
 │ DOUT  (n/c)   │                     └──────────────────►│− /  │ │       │
 └───────────────┘                     │                 2└──┘   │ │    C17 100n
   0 – 5.00 V                          │                         │ │       │
                                       └─────────────────────────┘ │      GND
                                                                   │
                                                          (feedback tie)

  MQ-4 ─► R4 100k / R5 100k ─► U3B (pins 5,6,7) ─► R9 1k / C18 100n ─► GPIO35
  MQ-7 ─► R6 100k / R7 100k ─► U3C (pins 10,9,8) ─► R10 1k / C19 100n ─► GPIO32
  U3D  ─► UNUSED: pin 12 (+) → GND, pin 13 (−) ↔ pin 14 (out)   [no floating input]

  U3 MCP6004:  pin 4 = VDD = +3V3_AUX   ·   pin 11 = VSS = GND   ·   C12 100 nF at pin 4


 ══ MQ-7 HEATER CYCLING DRIVER ════════════════════════════════════════════

                    +5V_SYS
                       │
                  ┌────┴─────┐
                  │  MQ-7    │   R_H = 33 Ω ±5 %
                  │  H+      │
                  │  H−  ────┼──────┐
                  └──────────┘      │
                                    │  D
   GPIO33 ──┬── R11 100Ω ──► G ┌────┴────┐
            │                  │  Q2     │  IRLZ44N  logic-level N-ch
         R12 10k               │ IRLZ44N │  V_GS(th) 1–2 V
            │                  └────┬────┘  R_DS(on) ≈ 0.03 Ω @ V_GS 3.3 V
           GND                      │  S    P_diss = 0.152² × 0.03 = 0.7 mW
                                   GND      → no heatsink

   R12 (10 kΩ gate pull-down) holds the heater OFF during reset and before
   GPIO33 is configured — without it the gate floats and the heater state
   is undefined at power-up.

   CYCLE (150 s, repeating):
     ┌── 60 s ──┬─────── 90 s ───────┐
     │ D = 100% │ D = 7.84 % @ 1 kHz │
     │  5.00 V  │  V_rms = 1.40 V    │      sample ADC here ──┐
     │  151.5mA │  I_avg = 11.9 mA   │                        ▼
     └──────────┴──────────────────[last 2 s]─────────────────┘

   Derivation:  V_rms = V × √D  →  D = (1.4 / 5.0)² = 0.0784
   Verification vs datasheet (<350 mW):
       (758 mW × 60 s + 59 mW × 90 s) / 150 s = 339 mW   ✔
   Bench check: a DC voltmeter on H− reads 5 × 0.0784 = 0.39 V during the
   low phase; a true-RMS meter reads 1.40 V. Both are correct.
```

## Sheet 4 — I²C bus

```
                +3V3_AUX
                    │
            ┌───────┴───────┐
         R_PU 4k7        R_PU 4k7        ◄── already fitted on the GY-521 and
            │               │                GY-906 breakouts. Do NOT add a
   SDA ─────┴───────────┬───┼─────────┐      third external pair — see §9.
   SCL ─────────────────┼───┴───┬─────┼───┐
                        │       │     │   │
   GPIO21 ──────────────┘       │     │   │
   GPIO22 ──────────────────────┘     │   │
                                      │   │
        ┌─────────────────────────────┘   │
        │        ┌────────────────────────┘
        ▼        ▼
  ┌─────────────────────┐        ┌─────────────────────┐
  │ MPU6050  (GY-521)   │        │ MLX90614 (GY-906)   │
  │  VCC ◄── +3V3_AUX   │        │  VCC ◄── +3V3_AUX   │
  │  GND ──► GND        │        │  GND ──► GND        │
  │  SDA ◄──► I2C_SDA   │        │  SDA ◄──► I2C_SDA   │
  │  SCL ◄─── I2C_SCL   │        │  SCL ◄─── I2C_SCL   │
  │  AD0 ──► GND  =0x68 │        │  ADDR = 0x5A (EEPROM)│
  │  INT   (n/c)        │        │  C15 100 nF at VCC   │
  │  XDA/XCL (n/c)      │        └─────────────────────┘
  │  C14 100 nF at VCC  │                  │
  └─────────────────────┘                  │  ≤150 mm flexible silicone
        RIGID mount to frame arm           │  28 AWG, strain-relieved at
        (vibration IS the signal)          │  the servo horn
                                           ▼
                                   mounted on SG90 horn
                                   + 6 mm ID × 20 mm collimator

  Bus: 100 kHz.  Total capacitance with ≤200 mm runs ≈ 80 pF ✔
  Power-up note: SDA must NOT be held low at MLX90614 power-up, or the part
  enters PWM output mode instead of SMBus. The pull-ups guarantee this.
```

## Sheet 5 — Ultrasonic, servo, humidity

```
 ══ HC-SR04 ULTRASONIC ════════════════════════════════════════════════════

  ┌──────────────┐
  │ HC-SR04      │
  │  VCC ◄───────┼── +5V_SYS   (C9 100 nF at the connector)
  │  GND ────────┼──► GND
  │  TRIG ◄──────┼─────────────────────────────────── GPIO25
  │              │     3.3 V drive; HC-SR04 V_IH ≈ 2.0–2.5 V ✔
  │  ECHO ───────┼──┬── R13 2.2k ──┬────────────────── GPIO13
  └──────────────┘  │              │
      5.0 V out     │           R14 3.3k
                    │              │
                   (5 V)          GND

   V_GPIO13 = 5.00 × 3.3k/(2.2k+3.3k) = 5.00 × 0.600 = 3.00 V   ✔
   ESP32 V_IH = 0.75 × 3.3 = 2.48 V  →  3.00 V is a solid logic HIGH
   Source Z = 2.2k ∥ 3.3k = 1.32 kΩ  →  fine for a digital input
   Timing: capture with the RMT peripheral, not pulseIn()


 ══ SG90 SCANNING SERVO ═══════════════════════════════════════════════════

               +5V_SYS ──┬──────────────► SG90  RED   (V+)
                         │
                    C8 1000 µF          SG90  BROWN (GND) ──► GND  ★ star point
                         │
                        GND             SG90  ORANGE(SIG) ◄── GPIO4

   C8 is NOT optional. SG90 inrush/stall is ~700 mA; without local bulk
   capacitance the 5 V rail dips and the ESP32 brown-out-resets.
   Run the servo's GND back to the BEC star point, NOT through the
   sensor ground daisy-chain.
   Drive: LEDC 50 Hz, 500–2400 µs. Sweep 5–7 discrete positions over ~120°,
   settling 300 ms at each before taking the MLX90614 reading.


 ══ DHT22 / AM2302 TEMPERATURE + HUMIDITY ═════════════════════════════════

                +3V3_AUX ──┬───────────────► DHT22 pin 1 (VDD)
                           │
                       R15 10k                DHT22 pin 2 (DATA) ─┬─► GPIO27
                           │                                      │
                           └──────────────────────────────────────┘
                                                DHT22 pin 3 (NC)
                                                DHT22 pin 4 (GND) ──► GND

   3.3 V operation is within the AM2302's 3.3–6 V range, and keeps the
   data line at 3.3 V — no level shifter needed.
   MOUNTING: ≥60 mm from the MQ cluster (≈1.6 W of heater dissipation),
   in clean prop-wash. Minimum 2 s between reads.
```

## Sheet 6 — LoRa SX1278 (Ra-02)

```
                      +3V3_AUX  ◄── 3.3 V ONLY. No pin is 5 V tolerant.
                          │
              ┌───────────┼───────────┐
           C4 10 µF    C5 100 nF   R16 10k
              │           │           │
             GND         GND          │
                                      │
  ┌───────────────────────────┐       │
  │  Ra-02  (SX1278, 433 MHz) │       │
  │                           │       │
  │  VCC  ◄── +3V3_AUX ───────┤       │
  │  GND  ──► GND             │       │
  │  NSS  ◄─────────────────┬─┼───────┴──────────── GPIO5
  │  MOSI ◄─────────────────┼─┼────────────────────  GPIO23
  │  MISO ─────────────────►┼─┼────────────────────  GPIO19
  │  SCK  ◄─────────────────┼─┼────────────────────  GPIO18
  │  RST  ◄─────────────────┼─┼────────────────────  GPIO14
  │  DIO0 ─────────────────►┼─┼────────────────────  GPIO26
  │  DIO1–5   (n/c)         │ │
  │  ANT  ──► 433 MHz SMA whip  OR  λ/4 wire = 17.3 cm
  └───────────────────────────┘

  R16 (10 kΩ NSS pull-up) does two jobs:
    1. Deselects the radio whenever the ESP32 is in reset
    2. Satisfies the GPIO5 strapping requirement (must be HIGH at boot)

  C4/C5 must sit within 10 mm of the module's VCC pin — the +20 dBm TX
  burst is a 120 mA step and will otherwise pull the rail down mid-packet.

  ★ NEVER power the module without an antenna fitted.  An unterminated
    PA output reflects power back and destroys the SX1278.

  Suggested air config: SF9, BW 125 kHz, CR 4/5, +17 dBm, 433.0 MHz.
  Upgrade path: Ra-01H (SX1276, 868 MHz) is pin-for-pin identical here.
```

## Sheet 7 — RS-422 tether (physical wired link)

```
 ══ DRONE SIDE ════════════════════╗   ╔══ GROUND-STATION SIDE ════════════
                                   ║   ║
  +3V3_AUX ──┬── C13 100 nF ── GND ║   ║ +3V3 ──┬── C 100 nF ── GND
             │                     ║   ║        │
   ┌─────────┴──────────┐          ║   ║  ┌─────┴──────────────┐
   │  U4  MAX3491       │          ║   ║  │  U5  MAX3491       │
   │  (full duplex)     │          ║   ║  │  (full duplex)     │
   │                    │          ║   ║  │                    │
   │ VCC ◄── +3V3_AUX   │          ║   ║  │ VCC ◄── +3V3       │
   │ GND ──► GND        │          ║   ║  │ GND ──► GND        │
   │                    │          ║   ║  │                    │
   │ DI  ◄── GPIO17 ────┤          ║   ║  │ RO  ──► GPIO16     │
   │ RO  ──► GPIO16 ────┤          ║   ║  │ DI  ◄── GPIO17     │
   │                    │          ║   ║  │                    │
   │ DE  ◄── VCC  (HIGH)│  ← driver always on                  │
   │ RE̅  ◄── GND  (LOW) │  ← receiver always on                │
   │                    │   no direction-control GPIO needed   │
   │                    │          ║   ║  │                    │
   │ Y ─────────────────┼══ PAIR 1 ══► │ A ◄────────┬──────────┤
   │ Z ─────────────────┼══ (TX)  ══► │ B ◄────────┤          │
   │                    │          ║   ║  │      R_T 120 Ω     │
   │ A ◄────────┬───────┼◄══ PAIR 2 ══┼─│ Y                   │
   │ B ◄────────┤       │◄══ (RX)  ══┼─│ Z                   │
   │        R_T 120 Ω   │          ║   ║  └────────────────────┘
   └────────────────────┘          ║   ║
                                   ║   ║
   GND ═══════════════ COMMON GND ═╩═══╩═ GND   (5th conductor)

   FAIL-SAFE BIAS (both ends, across the receiver inputs):
        A ── R 1 kΩ ── +3V3       B ── R 1 kΩ ── GND
   Defines an idle logic state so a disconnected or cut tether produces
   silence rather than a stream of framing errors.

   CABLE: 4 × 30 AWG as two twisted pairs + 1 × 30 AWG ground.
          ≈ 2.8 g/m  →  15 m ≈ 45 g.       ★ Do NOT use CAT5 — 15 m ≈ 570 g.
   Baud:  115200 (RS-422 supports far more; 115200 keeps margin over
          15 m of unshielded thin-gauge cable next to four ESCs).
   Deploy with a payout spool, keep slack, 15 m practical maximum.

   SHORT-RUN FALLBACK (bench demo ≤ 2 m):
   omit U4/U5 entirely and wire GPIO17 → GPIO16 and GPIO16 → GPIO17
   across the two boards, with common ground.
```

## Sheet 8 — Analog video chain (electrically independent)

```
   ┌──────────────────────────────────────────────────────────────┐
   │  DOMAIN C — NO ELECTRICAL CONNECTION TO THE ESP32            │
   │                                                              │
   │   +5V_VID ──► FPV CAMERA  ──(CVBS 1 Vpp, 75 Ω)──► 5.8 G VTX  │
   │       │            │                                 │       │
   │      GND ◄─────────┴─────────────────────────────────┘       │
   │                                              ANT ──┐         │
   └────────────────────────────────────────────────────┼─────────┘
                                                        │
                                             ((( 5.8 GHz )))
                                                        │
                                                        ▼
                                          5.8 GHz RX + goggles / monitor

   WHY NO ESP32 CONNECTION:  the ESP32 has no composite-video peripheral.
   Ingesting video would require an ESP32-CAM (OV2640 on the DVP parallel
   bus), which consumes ~13 GPIOs and collides with I²C, VSPI and ADC1
   simultaneously. The sensor payload and a camera cannot share one ESP32.
   Keeping the analog chain separate is the correct engineering answer and
   costs nothing.

   NOISE DISCIPLINE:
     · FB1 ferrite bead in series with +5V_VID
     · C10 220 µF at the VTX supply pins
     · Mount the 5.8 GHz antenna ≥150 mm from the 433 MHz LoRa antenna,
       ideally cross-polarised. (The bands are far apart, so the real
       risk is broadband switching noise from the VTX regulator, not
       co-channel interference.)
     · ★ Never power the VTX without its antenna fitted.
```

## Sheet 9 — Ground station

```
   ┌─────────────────────────────────────────────────────────────┐
   │  GROUND STATION                                             │
   │                                                             │
   │   Laptop ──USB 5 V──► ESP32 #2 (DevKit)                      │
   │      ▲                   │                                  │
   │      │                   ├── UART2 (16/17) ──► U5 MAX3491   │
   │      │                   │                      │           │
   │      │                   │                  RS-422 tether ══╪══► drone
   │      │                   │                                  │
   │      │                   └── VSPI (18/19/23/5/14/26)        │
   │      │                              │                       │
   │      │                        Ra-02 SX1278 ((( 433 MHz ))) ═╪══► drone
   │      │                                                      │
   │      └── USB CDC serial, 115200, JSON lines                 │
   │                                                             │
   │   Laptop: Python bridge  ──►  Gemini / Google AI Studio      │
   │                               live hazard dashboard          │
   └─────────────────────────────────────────────────────────────┘

   Ground ESP32 keeps Wi-Fi ENABLED (it is above ground and may serve the
   dashboard on the LAN). Its ADC2 pins are unused, so the Wi-Fi/ADC2
   conflict does not apply here.
   Powered from USB 5 V (500 mA available; the node draws ~180 mA peak). ✔
```

---

# 7. COMPLETE WIRING TABLE

Every physical connection in the schematic. `+3V3_AUX` = AMS1117 output; `ESP32 3V3` = DevKit's own regulator (separate).

| # | Component | Component pin | Connects to | Interface | Voltage | Purpose |
|---|-----------|---------------|-------------|-----------|---------|---------|
| 1 | 3S LiPo | + | F1 polyfuse | Power | 11.1 V | Payload branch tap |
| 2 | 3S LiPo | − | GND star | Power | 0 V | System ground |
| 3 | F1 MF-R200 | out | Q1 source | Power | 11.1 V | 2 A overcurrent protection |
| 4 | Q1 IRF4905 | gate | GND via R1 100 k | Power | −11.1 V | Reverse-polarity protection |
| 5 | Q1 IRF4905 | drain | VBAT_PROT | Power | 11.1 V | Protected battery rail |
| 6 | D1 SMBJ15A | A/K | VBAT_PROT / GND | Power | 11.1 V | Transient clamp |
| 7 | C1 470 µF | +/− | VBAT_PROT / GND | Power | 11.1 V | Input bulk |
| 8 | BEC1 5 V/3 A | IN+/IN− | VBAT_PROT / GND | Power | 11.1 V | Payload supply |
| 9 | BEC1 | OUT+ | `+5V_SYS` | Power | 5.0 V | Sensor/logic rail |
| 10 | C2 220 µF, C3 100 nF | — | `+5V_SYS` / GND | Power | 5.0 V | Rail decoupling |
| 11 | BEC2 5 V/2 A | IN+/IN− | VBAT_PROT / GND | Power | 11.1 V | Video supply |
| 12 | BEC2 | OUT+ | `+5V_VID` via FB1 | Power | 5.0 V | Isolated video rail |
| 13 | U2 AMS1117-3.3 | IN | `+5V_SYS` | Power | 5.0 V | LDO input |
| 14 | U2 AMS1117-3.3 | OUT | `+3V3_AUX` | Power | 3.3 V | Sensor/radio rail |
| 15 | C6 10 µF, C7 100 nF | — | `+3V3_AUX` / GND | Power | 3.3 V | LDO output decoupling |
| 16 | **ESP32** | VIN | `+5V_SYS` | Power | 5.0 V | Board supply |
| 17 | ESP32 | GND | GND star | Power | 0 V | Ground |
| 18 | C11 100 µF, C16 100 nF | — | VIN / GND | Power | 5.0 V | Board bulk |
| 19 | ESP32 | EN | 10 k→3V3, 100 nF→GND | Reset | 3.3 V | Clean power-on reset |
| 20 | **MQ-2** | VCC | `+5V_SYS` | Power | 5.0 V | Heater + comparator |
| 21 | MQ-2 | GND | GND star | Power | 0 V | Ground |
| 22 | MQ-2 | AOUT | R2 100 k | Analog | 0–5.0 V | Raw LPG/smoke signal |
| 23 | R2/R3 junction | — | U3A pin 3 (+) | Analog | 0–2.50 V | Divided signal |
| 24 | U3A MCP6004 | pin 1 (out) | R8 1 k → GPIO34 | Analog | 0–2.50 V | Buffered MQ-2 → ADC1_CH6 |
| 25 | C17 100 nF | — | GPIO34 / GND | Analog | — | Anti-alias, f_c 1.59 kHz |
| 26 | **MQ-4** | VCC | `+5V_SYS` | Power | 5.0 V | Heater + comparator |
| 27 | MQ-4 | GND | GND star | Power | 0 V | Ground |
| 28 | MQ-4 | AOUT | R4 100 k | Analog | 0–5.0 V | Raw CH₄ signal |
| 29 | U3B MCP6004 | pin 7 (out) | R9 1 k → GPIO35 | Analog | 0–2.50 V | Buffered MQ-4 → ADC1_CH7 |
| 30 | C18 100 nF | — | GPIO35 / GND | Analog | — | Anti-alias |
| 31 | **MQ-7** | VCC / H+ | `+5V_SYS` | Power | 5.0 V | Heater high side |
| 32 | MQ-7 | H− | Q2 drain | Power | switched | Heater low side |
| 33 | MQ-7 | AOUT | R6 100 k | Analog | 0–5.0 V | Raw CO signal |
| 34 | U3C MCP6004 | pin 8 (out) | R10 1 k → GPIO32 | Analog | 0–2.50 V | Buffered MQ-7 → ADC1_CH4 |
| 35 | C19 100 nF | — | GPIO32 / GND | Analog | — | Anti-alias |
| 36 | **Q2 IRLZ44N** | gate | GPIO33 via R11 100 Ω | Digital | 0/3.3 V | Heater PWM |
| 37 | R12 10 k | — | Q2 gate / GND | Digital | — | Gate pull-down, OFF at reset |
| 38 | Q2 IRLZ44N | source | GND star | Power | 0 V | Heater return |
| 39 | **U3 MCP6004** | pin 4 (VDD) | `+3V3_AUX` | Power | 3.3 V | Op-amp supply |
| 40 | U3 MCP6004 | pin 11 (VSS) | GND | Power | 0 V | Ground |
| 41 | C12 100 nF | — | U3 pin 4 / GND | Power | 3.3 V | Op-amp decoupling |
| 42 | U3D (unused) | pin 12 (+) | GND | Analog | 0 V | Prevent floating input |
| 43 | U3D (unused) | pin 13 ↔ 14 | — | Analog | — | Unity-gain tie-off |
| 44 | **HC-SR04** | VCC | `+5V_SYS` | Power | 5.0 V | Sensor supply |
| 45 | HC-SR04 | GND | GND star | Power | 0 V | Ground |
| 46 | HC-SR04 | TRIG | GPIO25 | Digital | 3.3 V | 10 µs trigger pulse |
| 47 | HC-SR04 | ECHO | R13 2.2 k | Digital | 5.0 V | Raw echo (never to GPIO) |
| 48 | R13/R14 junction | — | GPIO13 | Digital | 3.00 V | Divided echo, RMT capture |
| 49 | R14 3.3 k | — | GPIO13 / GND | Digital | — | Divider lower leg |
| 50 | C9 100 nF | — | HC-SR04 VCC / GND | Power | 5.0 V | Local decoupling |
| 51 | **SG90 servo** | V+ (red) | `+5V_SYS` | Power | 5.0 V | Servo supply |
| 52 | SG90 servo | GND (brown) | GND star (direct) | Power | 0 V | High-current return |
| 53 | SG90 servo | SIG (orange) | GPIO4 | PWM | 3.3 V | LEDC 50 Hz, 500–2400 µs |
| 54 | C8 1000 µF | — | Servo V+ / GND | Power | 5.0 V | Inrush/stall bulk — mandatory |
| 55 | **DHT22** | pin 1 VDD | `+3V3_AUX` | Power | 3.3 V | Sensor supply |
| 56 | DHT22 | pin 2 DATA | GPIO27 | 1-wire | 3.3 V | Temp + humidity |
| 57 | R15 10 k | — | GPIO27 / `+3V3_AUX` | 1-wire | 3.3 V | Bus pull-up |
| 58 | DHT22 | pin 4 GND | GND star | Power | 0 V | Ground |
| 59 | **MPU6050** | VCC | `+3V3_AUX` | Power | 3.3 V | IMU supply |
| 60 | MPU6050 | GND | GND star | Power | 0 V | Ground |
| 61 | MPU6050 | SDA | GPIO21 | I²C | 3.3 V | Data, addr 0x68 |
| 62 | MPU6050 | SCL | GPIO22 | I²C | 3.3 V | Clock, 100 kHz |
| 63 | MPU6050 | AD0 | GND | I²C | 0 V | Address select → 0x68 |
| 64 | C14 100 nF | — | MPU6050 VCC / GND | Power | 3.3 V | Decoupling |
| 65 | **MLX90614** | VCC | `+3V3_AUX` | Power | 3.3 V | IR sensor supply |
| 66 | MLX90614 | GND | GND star | Power | 0 V | Ground |
| 67 | MLX90614 | SDA | GPIO21 | SMBus | 3.3 V | Data, addr 0x5A |
| 68 | MLX90614 | SCL | GPIO22 | SMBus | 3.3 V | Clock, 100 kHz |
| 69 | C15 100 nF | — | MLX90614 VCC / GND | Power | 3.3 V | Decoupling |
| 70 | **LoRa Ra-02** | VCC | `+3V3_AUX` | Power | 3.3 V | **3.3 V only** |
| 71 | LoRa Ra-02 | GND | GND star | Power | 0 V | Ground |
| 72 | LoRa Ra-02 | NSS | GPIO5 | SPI | 3.3 V | Chip select |
| 73 | R16 10 k | — | GPIO5 / `+3V3_AUX` | SPI | 3.3 V | Deselect at reset + GPIO5 strap |
| 74 | LoRa Ra-02 | MOSI | GPIO23 | SPI | 3.3 V | VSPI data out |
| 75 | LoRa Ra-02 | MISO | GPIO19 | SPI | 3.3 V | VSPI data in |
| 76 | LoRa Ra-02 | SCK | GPIO18 | SPI | 3.3 V | VSPI clock, 8 MHz |
| 77 | LoRa Ra-02 | RST | GPIO14 | Digital | 3.3 V | Module reset |
| 78 | LoRa Ra-02 | DIO0 | GPIO26 | IRQ | 3.3 V | TX/RX-done interrupt |
| 79 | LoRa Ra-02 | ANT | 433 MHz antenna | RF | — | **Fit before power-up** |
| 80 | C4 10 µF, C5 100 nF | — | LoRa VCC / GND | Power | 3.3 V | TX-burst decoupling |
| 81 | **U4 MAX3491** | VCC | `+3V3_AUX` | Power | 3.3 V | Transceiver supply |
| 82 | U4 MAX3491 | GND | GND star | Power | 0 V | Ground |
| 83 | U4 MAX3491 | DI | GPIO17 (U2TXD) | UART | 3.3 V | Outbound telemetry |
| 84 | U4 MAX3491 | RO | GPIO16 (U2RXD) | UART | 3.3 V | Inbound commands |
| 85 | U4 MAX3491 | DE | VCC | Control | 3.3 V | Driver permanently on |
| 86 | U4 MAX3491 | RE̅ | GND | Control | 0 V | Receiver permanently on |
| 87 | U4 MAX3491 | Y, Z | Tether pair 1 | RS-422 | ±2 V diff | Differential TX |
| 88 | U4 MAX3491 | A, B | Tether pair 2 | RS-422 | ±2 V diff | Differential RX |
| 89 | R_T 120 Ω | — | A / B | RS-422 | — | Line termination |
| 90 | Bias 1 k / 1 k | — | A→3V3, B→GND | RS-422 | 3.3 V | Idle-state fail-safe |
| 91 | C13 100 nF | — | U4 VCC / GND | Power | 3.3 V | Decoupling |
| 92 | ESP32 | GPIO2 | On-board LED | Digital | 3.3 V | Status indicator |
| 93 | ESP32 | GPIO12 | **NOT CONNECTED** | — | — | MTDI strap — must stay low |
| 94 | **FPV camera** | V+ / GND | `+5V_VID` / GND | Power | 5.0 V | Domain C only |
| 95 | FPV camera | CVBS out | VTX video in | Analog | 1 Vpp / 75 Ω | Composite video |
| 96 | **5.8 G VTX** | V+ / GND | `+5V_VID` / GND | Power | 5.0 V | Domain C only |
| 97 | 5.8 G VTX | ANT | 5.8 GHz antenna | RF | — | **Fit before power-up** |
| 98 | FB1 ferrite | — | BEC2 out → `+5V_VID` | Power | 5.0 V | Switching-noise filter |
| 99 | C10 220 µF | — | `+5V_VID` / GND | Power | 5.0 V | VTX bulk |

**Wireless links (no conductor — shown as RF, not wire):**

| Link | Band | Endpoints | Notes |
|------|------|-----------|-------|
| LoRa telemetry | 433 MHz | Drone SX1278 ←→ Ground SX1278 | SF9/BW125/CR4-5, +17 dBm |
| Analog video | 5.8 GHz | VTX → RX/goggles | One-way, no ESP32 involvement |
| RC control | 2.4 GHz | TX → RC RX | Domain A, pre-existing |

---

# 8. POWER BUDGET

## 8.1 `+3V3_AUX` rail

| Component | Voltage | Typical | Peak | Typical power |
|-----------|---------|---------|------|---------------|
| LoRa SX1278 (RX / TX +20 dBm) | 3.3 V | 12 mA | 120 mA | 40 mW |
| MPU6050 | 3.3 V | 4 mA | 4 mA | 13 mW |
| MLX90614 | 3.3 V | 2 mA | 2 mA | 7 mW |
| DHT22 | 3.3 V | 1.5 mA | 1.5 mA | 5 mW |
| MCP6004 (4 ch) | 3.3 V | 0.6 mA | 0.6 mA | 2 mW |
| MAX3491 | 3.3 V | 20 mA | 20 mA | 66 mW |
| I²C pull-ups | 3.3 V | 3 mA | 3 mA | 10 mW |
| **TOTAL** | **3.3 V** | **43 mA** | **151 mA** | **143 mW** |

AMS1117-3.3 dissipation at peak = `(5.0 − 3.3) × 0.151 = 0.26 W`. ✔

## 8.2 `+5V_SYS` rail

| Component | Voltage | Typical | Peak | Typical power |
|-----------|---------|---------|------|---------------|
| MQ-2 (heater 33 Ω + comparator/LED) | 5 V | 162 mA | 162 mA | 810 mW |
| MQ-4 (heater 33 Ω + comparator/LED) | 5 V | 162 mA | 162 mA | 810 mW |
| **MQ-7 (cycled heater, 150 s average)** | 5 V | **78 mA** | 162 mA | 390 mW |
| HC-SR04 | 5 V | 15 mA | 15 mA | 75 mW |
| SG90 servo (sweeping / stall) | 5 V | 120 mA | **700 mA** | 600 mW |
| ESP32 DevKit at VIN (Wi-Fi off / TX) | 5 V | 130 mA | 300 mA | 650 mW |
| AMS1117 input (= 3V3_AUX load) | 5 V | 45 mA | 165 mA | 225 mW |
| **TOTAL** | **5 V** | **712 mA** | **1666 mA** | **3.56 W** |

**BEC1 requirement:** ≥ 2 A continuous. **Specified 3 A → 1.80× margin at absolute peak.** ✔

## 8.3 `+5V_VID` rail

| Component | Voltage | Typical | Peak | Typical power |
|-----------|---------|---------|------|---------------|
| FPV camera | 5 V | 100 mA | 120 mA | 500 mW |
| 5.8 GHz AIO VTX (25–200 mW) | 5 V | 300 mA | 400 mA | 1500 mW |
| **TOTAL** | **5 V** | **400 mA** | **520 mA** | **2.00 W** |

**BEC2 requirement:** ≥ 0.8 A. **Specified 2 A → 3.8× margin.** ✔

## 8.4 Battery-side totals

```
  P(5V_SYS)  typ 3.56 W  /  peak 8.33 W
  P(5V_VID)  typ 2.00 W  /  peak 2.60 W
  ───────────────────────────────────────
  Payload    typ 5.56 W  /  peak 10.93 W
  ÷ 0.85 BEC efficiency  →  6.54 W / 12.86 W drawn from the pack
  ÷ 11.1 V               →  0.59 A typical, 1.16 A peak
```

Against a hover current of 16–20 A, **the payload adds ≈3 % to the electrical load.** Electrically negligible.

## 8.5 Mass budget and flight-time impact

| Item | Mass |
|------|------|
| ESP32 DevKit | 10 g |
| MQ-2 / MQ-4 / MQ-7 modules (3 × 7 g) | 21 g |
| DHT22 | 3 g |
| MPU6050 (GY-521) | 2 g |
| HC-SR04 | 9 g |
| MLX90614 (GY-906) + collimator tube | 6 g |
| SG90 servo + bracket | 12 g |
| LoRa Ra-02 + antenna | 16 g |
| FPV camera + AIO VTX + antenna | 13 g |
| BEC1 + BEC2 | 10 g |
| Perfboard, conditioning circuitry, wiring, connectors | 28 g |
| 3D-printed payload pod + fasteners | 25 g |
| **Airborne payload subtotal** | **155 g** |
| Tether, 15 m × 30 AWG ×5 | 45 g |
| **Total with tether** | **200 g** |

```
  Base F450 AUW (frame, 4× 2212/920 KV, 4× 30 A ESC, FC, RX, 3S 2200 mAh)
      ≈ 1100–1250 g
  + payload 200 g  →  AUW ≈ 1300–1450 g

  Static thrust, 2212/920 KV with 1045 props on 3S ≈ 750 g per motor
      Total available thrust ≈ 3000 g
      Thrust-to-weight = 3000 / 1450 = 2.07 : 1          ✔ (target ≥ 2:1)

  Hover current rises from ~16–18 A to ~19–23 A
      Flight time falls from ~10–12 min to ~7–9 min      ← plan the demo
                                                            around this
```

**Verdict: the F450 carries this payload with healthy margin.** The binding constraint is flight time, not thrust or current. Budget a 7-minute working window per battery and carry at least three packs to the demo.

## 8.6 Recommended safety margins — summary

| Domain | Margin achieved | Target | Status |
|--------|-----------------|--------|--------|
| BEC1 (5 V logic/sensors) | 1.80× at peak | ≥1.5× | ✔ |
| BEC2 (5 V video) | 3.85× | ≥1.5× | ✔ |
| AMS1117 (3.3 V) | 5.3× | ≥2× | ✔ |
| Thrust-to-weight | 2.07 : 1 | ≥2 : 1 | ✔ |
| Battery C-rating headroom | 23 A draw vs 55 A (2200 mAh 25C) | ≥1.5× | ✔ 2.4× |
| GPIO availability | 7 spare of 26 | ≥3 | ✔ |
| ADC headroom | 2.50 V vs 3.3 V | — | ✔ 1.32× |

---

# 9. FINAL ELECTRICAL REVIEW

Independent second pass over the completed schematic. Every item was checked; issues found were corrected in the drawings above rather than merely noted.

| # | Check | Result | Action |
|---|-------|--------|--------|
| 1 | **Power polarity** | Q1 IRF4905 source→battery, drain→load, gate→GND. Correct orientation for P-channel reverse protection. Body diode conducts on correct polarity, blocks on reverse. | ✔ Pass |
| 2 | **Voltage compatibility** | No ESP32 pin exceeds 3.3 V. MQ AOUT and HC-SR04 ECHO both divided. LoRa on 3.3 V only. DHT22 on 3.3 V so its data line is 3.3 V. | ✔ Pass — 5 corrections applied |
| 3 | **GPIO conflicts** | Each of the 19 assigned GPIOs used once. Cross-checked against the WROOM-32 pin list. | ✔ Pass |
| 4 | **ADC2/Wi-Fi conflict** | All three gas channels on ADC1 (34/35/32). | ✔ Fixed in §1.3 |
| 5 | **ADC voltage limits** | Max ADC input 2.50 V vs 3.3 V rail. Buffer output physically clamped by its 3.3 V supply. | ✔ Pass, dual protection |
| 6 | **ADC source impedance** | 1 kΩ after the RC (would have been 50 kΩ without the buffer). | ✔ Fixed by MCP6004 |
| 7 | **I²C address conflicts** | 0x68 and 0x5A. No collision. Re-addressing procedure documented for future expansion. | ✔ Pass |
| 8 | **I²C pull-ups** | GY-521 and GY-906 each carry 4.7 kΩ. In parallel: 2.35 kΩ → sink current `3.3 V / 2.35 kΩ = 1.40 mA`, within the 3 mA I²C limit. **Do not add a third external pair** — that would give 1.57 kΩ and 2.1 mA, still legal but with no benefit and reduced noise margin. | ✔ Pass — external pull-ups deliberately omitted |
| 9 | **Ground connections** | Single star point at BEC1's ground terminal. Servo and MQ heaters (the two high-current returns) wired directly to the star, not daisy-chained through the sensor grounds. All three rails share this ground. | ✔ Pass |
| 10 | **Regulator capacity** | BEC1 1.80×, BEC2 3.85×, AMS1117 5.3×. | ✔ Pass |
| 11 | **Sensor current requirements** | Dominant loads correctly identified: MQ heaters (continuous), SG90 (transient), VTX (continuous, isolated to its own rail). | ✔ Pass |
| 12 | **Boot / strapping pins** | GPIO5 pulled high (R16). GPIO12 unconnected and marked. GPIO2 drives only the LED. GPIO15 unused. GPIO0 free for BOOT. | ✔ Pass — 3 corrections applied |
| 13 | **Floating pins** | MOSFET gate pulled down (R12). Unused op-amp U3D tied off. LoRa NSS pulled up. HC-SR04 ECHO always driven. RS-422 receiver bias resistors fitted. DHT22 data pulled up. **No floating input anywhere.** | ✔ Pass — 5 tie-offs added |
| 14 | **Missing capacitors** | 100 nF at every active device VCC; 10 µF + 100 nF at LoRa; 1000 µF at servo; 470 µF at battery input; 220 µF per 5 V rail; 100 µF at ESP32 VIN. | ✔ Pass |
| 15 | **Missing protection** | Polyfuse + P-MOSFET reverse protection + TVS on the payload branch. Payload faults cannot reach the flight domain. | ✔ Pass |
| 16 | **Connector polarity** | XT60 on the battery tap (polarised). Servo and sensor headers keyed 3-pin. Recommend colour-coding every flying lead: red = 5 V, orange = 3.3 V, black = GND. | ✔ Pass with assembly note |
| 17 | **Communication conflicts** | I²C, VSPI, UART2 all independent. UART0 reserved for flashing, so the payload link on UART2 does not clash with programming. | ✔ Pass |
| 18 | **Camera bandwidth** | Not applicable — the video chain is fully analog and never touches the ESP32. This was itself a correction (§1.9b). | ✔ Resolved by architecture |
| 19 | **Wireless operation** | LoRa 433 MHz and video 5.8 GHz are widely separated; the real coupling risk is VTX regulator noise, mitigated with FB1 + C10 and ≥150 mm antenna separation. Drone Wi-Fi disabled. | ✔ Pass |
| 20 | **Thermal** | AMS1117 ΔT ≈ 16 °C. Q2 dissipates 0.7 mW. MQ heaters dissipate ~1.6 W inside the pod — DHT22 relocated ≥60 mm away and placed in prop-wash. | ✔ Pass — mounting correction applied |
| 21 | **MLX90614 FOV vs. detection range** | Bare 90° BAA detects a human reliably only to ~1 m. Collimator added; range extended to ~3 m (§1.9c). | ✔ Fixed |
| 22 | **MQ-7 heater cycling** | Constant 5 V would give invalid CO readings. MOSFET driver added, duty derived as 7.84 % and cross-validated against the datasheet's 350 mW figure. | ✔ Fixed |
| 23 | **Tether integrity** | Single-ended UART replaced with RS-422; CAT5 replaced with 30 AWG twisted pairs (570 g → 45 g). | ✔ Fixed |
| 24 | **Dual-regulator hazard** | ESP32 3V3 and `+3V3_AUX` explicitly kept separate; only GND is common. Flagged on Sheets 1 and 2. | ✔ Pass |
| 25 | **Overall reliability** | Remaining single points of failure: (a) the tether is a mechanical snag risk — fly with a payout spool and slack; (b) I²C wiring runs across the moving servo horn — keep ≤150 mm, use flexible silicone 28 AWG, strain-relieve at both ends; (c) MQ elements degrade and need periodic re-zeroing in clean air. | ✔ Pass with operational notes |

**Review outcome: 25 of 25 checks pass.** Nine defects were found during validation and all nine are corrected in the schematic, not merely flagged.

---

# 10. PHYSICAL ASSEMBLY SEQUENCE

Bring the system up in stages. Do not skip to full integration — at every stage below, something can be destroyed if the previous stage was not verified.

## Stage 0 — Before assembly day
- Power all three MQ sensors from a bench 5 V supply for **24–48 h of burn-in**. This must happen days ahead; there is no shortcut.
- Fit the 433 MHz antenna to both Ra-02 modules and never remove them.
- Print/fabricate the 6 mm ID × 20 mm collimator tube; blacken the interior.

## Stage 1 — Power, unloaded
1. Bench-test BEC1 and BEC2 with no load. Confirm **5.00 V ±0.1 V** on each.
2. Assemble F1 → Q1 → D1 → C1. With a bench supply at 11.1 V, confirm VBAT_PROT ≈ 11.1 V.
3. **Reverse-polarity test:** apply 11.1 V backwards. VBAT_PROT must read 0 V and nothing should get warm. Restore correct polarity.
4. Build the AMS1117 stage. Confirm `+3V3_AUX` = **3.30 V ±0.1 V**.
5. Confirm with a meter that ESP32 3V3 and `+3V3_AUX` are **not** continuous with each other, and that all grounds are.

## Stage 2 — Controller alone
6. Power the ESP32 from `+5V_SYS` only. Flash a blink sketch on GPIO2. Confirm it boots cleanly over USB.
7. Confirm GPIO12 is unconnected.

## Stage 3 — I²C devices
8. Wire MPU6050 and MLX90614 to `+3V3_AUX` and GPIO21/22.
9. Run an I²C scanner at 100 kHz. **Expect exactly `0x68` and `0x5A`.** If either is missing, stop and fix before continuing.
10. Verify MLX90614 object and ambient temperature readings against a known warm object.

## Stage 4 — Digital sensors
11. Add DHT22 with R15. Confirm plausible temperature/humidity.
12. Build the HC-SR04 divider **on the bench first**. With the sensor powered and echoing, **measure the divider output with a meter — it must read ≈3.0 V, never 5 V.** Only then connect to GPIO13.
13. Verify distance readings against a tape measure at 0.5 m, 1 m, 2 m.

## Stage 5 — Analog chain (most failure-prone — do not rush)
14. Build all three divider + buffer chains **with the ESP32 disconnected.**
15. Power the MQ modules and the MCP6004. **Measure each buffer output with a meter. Every one must be ≤2.50 V.** If any reads above 2.6 V, a divider resistor is wrong — find it now, not after it has destroyed a GPIO.
16. Only after all three verify, connect to GPIO34/35/32.
17. Read raw ADC counts. Expect a slowly falling baseline as the sensors preheat over ~5 minutes.

## Stage 6 — MQ-7 heater driver
18. Fit Q2, R11, R12. With GPIO33 **held low**, confirm the MQ-7 heater is off (H− at 5 V, no heating).
19. Drive 7.84 % duty at 1 kHz. A DC meter on H− should read ≈**0.39 V**; a true-RMS meter reads ≈**1.40 V**. Both are expected.
20. Run one full 150 s cycle and confirm the sensor body warms during the 60 s high phase and cools during the 90 s low phase.

## Stage 7 — Servo
21. Fit C8 (1000 µF) **before** connecting the servo.
22. Connect SG90 to `+5V_SYS` and GPIO4, with its ground run directly to the star point.
23. Sweep the full range. **Watch for ESP32 resets.** If it resets, C8 is missing, too small, or the servo ground is daisy-chained — fix before proceeding.
24. Mount the MLX90614 + collimator on the horn. Route the I²C leads with strain relief and re-run the I²C scan through the full sweep to confirm the bus stays reliable while moving.

## Stage 8 — LoRa (antenna first, always)
25. **Confirm the antenna is fitted.** Then wire the Ra-02 to `+3V3_AUX` with C4/C5 within 10 mm, plus R16.
26. Run a register read (`RegVersion` should return `0x12`). If it does not, stop — do not transmit.
27. Bench range test both nodes at 1 m, then at increasing distance.

## Stage 9 — Tether
28. Build both MAX3491 stages with terminations and bias resistors.
29. Loopback test across the full 15 m cable at 115200 baud before it ever goes on the aircraft.

## Stage 10 — Video (entirely separate)
30. **Confirm the 5.8 GHz antenna is fitted.** Wire camera and VTX to `+5V_VID` through FB1 and C10.
31. Confirm video on the receiver. Verify the VTX is not thermally throttling after 5 minutes.

## Stage 11 — Integration
32. Mount everything in the pod. MPU6050 **rigidly to a frame arm**; DHT22 ≥60 mm from the MQ cluster; antennas ≥150 mm apart.
33. Colour-code and label every flying lead. Red = 5 V, orange = 3.3 V, black = GND.
34. **20-minute powered soak on the bench, props off.** Check every regulator and the MOSFET by touch. Confirm no resets, no I²C dropouts, no LoRa packet loss.
35. Measure total payload current from the pack and confirm it is ≈0.6 A typical, consistent with §8.4.

## Stage 12 — Flight
36. Props **off**. Arm, confirm the flight controller is unaffected by the payload and that motor arming does not disturb any sensor.
37. Props on. Tethered hover at 0.5 m for 60 s. Log the sensor stream and watch for vibration-induced I²C errors.
38. Verify the MPU6050 vibration baseline in hover — this becomes the noise floor the TinyML anomaly classifier is trained against.
39. Progressive flights: hover → controlled traverse → full mock-mine run.

---

## Appendix A — Net label index

| Net | Nominal | Source | Loads |
|-----|---------|--------|-------|
| `VBAT_IN` | 11.1 V | 3S LiPo tap | F1 |
| `VBAT_PROT` | 11.1 V | Q1 drain | BEC1, BEC2, D1, C1 |
| `+5V_SYS` | 5.0 V | BEC1 | ESP32 VIN, MQ-2/4/7, HC-SR04, SG90, AMS1117 |
| `+5V_VID` | 5.0 V | BEC2 (via FB1) | FPV camera, 5.8 G VTX |
| `+3V3_AUX` | 3.3 V | AMS1117 | LoRa, MPU6050, MLX90614, DHT22, MCP6004, MAX3491, pull-ups |
| `ESP32_3V3` | 3.3 V | DevKit on-board LDO | ESP32 core **only — never tied to `+3V3_AUX`** |
| `GND` | 0 V | Star at BEC1 | All |
| `I2C_SDA` / `I2C_SCL` | 3.3 V | GPIO21 / GPIO22 | MPU6050, MLX90614 |
| `SPI_SCK` / `MISO` / `MOSI` | 3.3 V | GPIO18 / 19 / 23 | SX1278 |
| `LORA_NSS` / `RST` / `DIO0` | 3.3 V | GPIO5 / 14 / 26 | SX1278 |
| `MQ2_ADC` / `MQ4_ADC` / `MQ7_ADC` | 0–2.50 V | MCP6004 outputs | GPIO34 / 35 / 32 |
| `MQ7_HEAT_PWM` | 0/3.3 V | GPIO33 | Q2 gate |
| `US_TRIG` / `US_ECHO` | 3.3 V / 3.00 V | GPIO25 / GPIO13 | HC-SR04 |
| `SERVO_PWM` | 3.3 V | GPIO4 | SG90 |
| `DHT_DATA` | 3.3 V | GPIO27 | DHT22 |
| `TETHER_TX` / `TETHER_RX` | 3.3 V | GPIO17 / GPIO16 | MAX3491 DI / RO |

## Appendix B — KiCad conversion notes

The design maps cleanly onto a KiCad project:
- **Hierarchical sheets** — use the nine sheets in §3/§6 as-is: Power, Controller, Gas, I²C, Digital, LoRa, Tether, Video, Ground Station.
- **Symbols** — ESP32-WROOM-32 (`RF_Module:ESP32-WROOM-32`), MCP6004 (`Amplifier_Operational:MCP6004`), IRLZ44N and IRF4905 (`Transistor_FET`), AMS1117 (`Regulator_Linear:AMS1117-3.3`), MAX3491 (`Interface_UART`). Sensor modules have no standard symbols — draw them as generic connectors with the pin names from §7.
- **Net labels** — Appendix A is the net list; use global labels for the six power nets so they resolve across sheets.
- **Layout priority if this becomes a PCB:** (1) star ground at the BEC return, (2) analog section (MCP6004 + dividers) physically separated from the servo and heater current paths, (3) LoRa decoupling within 10 mm of the module footprint, (4) heater and servo traces sized for 1 A.

---

*Document rev 1.0 — FAB6, SIH 2026 PS 26039. Nine design defects identified during validation; all nine corrected in this revision.*
