# Sprinkler-8 — Board Specification

8-Zone ESP32 WiFi Relay Controller  
Rev 0.2 (current) — May 2026

---

## Overview

The Sprinkler-8 is a custom PCB that switches eight 24 VAC irrigation solenoid zones using relay contacts. An ESP32-WROOM-32E module provides WiFi connectivity and runs the zone control logic. The board accepts 24 VAC directly from a standard irrigation transformer, converts it internally to 5 V and 3.3 V, and exposes eight screw-terminal zone outputs plus a common terminal.

---

## Power Architecture

| Stage | Input | Output | Device |
|-------|-------|--------|--------|
| AC rectification | 24 VAC | ~33 V DC | MB10F bridge rectifier (D1) + 470 µF bulk cap |
| 5 V buck converter | ~33 V DC | 5 V / 3 A | LM2596S-5.0 (U4) |
| 3.3 V LDO | 5 V | 3.3 V | AMS1117-3.3 (U3) |
| USB power (optional) | 5 V USB | 5 V rail | Schottky bypass via F2 polyfuse |

**Primary input:** 24 VAC via 2.1 mm barrel jack (J2) or 3.5 mm screw terminal (J3). A 3-pin jumper (JP1) selects the active input.  
**USB input:** USB-C (J1) powers the logic at 5 V for programming only — does not drive relay coils.  
**Input protection:** 5 A PTC polyfuse (F1) on the AC line.

---

## Microcontroller

| Item | Detail |
|------|--------|
| Module | ESP32-WROOM-32E (U1) |
| Flash | 4 MB |
| WiFi | 802.11 b/g/n 2.4 GHz (built-in antenna) |
| Logic voltage | 3.3 V |
| Programming | USB-C via CH340C UART (U2) with auto-reset circuit |

**Antenna keep-out:** 15 mm clearance around the ESP32 module antenna edge.

---

## GPIO Assignment

| GPIO | Function | Notes |
|------|----------|-------|
| 32 | Relay 1 (Zone 1) | Active HIGH |
| 33 | Relay 2 (Zone 2) | Active HIGH |
| 25 | Relay 3 (Zone 3) | Active HIGH |
| 26 | Relay 4 (Zone 4) | Active HIGH |
| 27 | Relay 5 (Zone 5) | Active HIGH |
| 14 | Relay 6 (Zone 6) | Active HIGH |
| 12 | Relay 7 (Zone 7) | Strapping pin — 10 kΩ pull-down to GND required; must be LOW at boot |
| 13 | Relay 8 (Zone 8) | Active HIGH |
| 23 | Status LED | 330 Ω series resistor to green LED (LED9) |
| 0  | BOOT button (SW1) | 10 kΩ pull-up; IO0 = LOW at boot enters download mode |
| TX0 / RX0 | UART0 | CH340C (USB serial) |
| EN | Reset | RST button (SW2) pulls EN low |

---

## Relay Circuit

Each of the 8 zones uses the same circuit:

```
ESP32 GPIO ──► ULN2803A input (U5)
                  ULN2803A output ──► Relay coil− (K1–K8)
                                      Relay coil+ ── +5V
                  ULN2803A COM (pin 10) ── +5V  ← flyback return, CRITICAL
Zone LED: +5V ── 470 Ω ── LED anode ── LED cathode ── ULN2803A output net
MOV snubber: across each relay NO contact and COM (47 V disc)
```

- **Relay:** HF46F-G/5-HS1, SPST-NO, 5 V coil, 5 A / 250 VAC contacts (LCSC C165255)
- **Driver:** ULN2803A Darlington array (U5) — 8 channels, internal flyback diodes
- **Contact rating:** 5 A / 250 VAC (well within 24 VAC solenoid loads ~0.3 A each)
- **Relay logic:** Active HIGH — GPIO HIGH = relay energised = zone ON

---

## Connectors

| Ref | Type | Label | Notes |
|-----|------|-------|-------|
| J1 | USB-C 16-pin | USB | Programming / 5 V logic power |
| J2 | 2.1 mm / 5.5 mm barrel | 24VAC | Primary AC input |
| J3 | 3.5 mm screw terminal (2-pin) | AC IN | Alternate AC input |
| J4–J11 | 3.5 mm screw terminal (2-pin each) | Z1–Z8 | Zone relay outputs |
| J12 | 3.5 mm screw terminal (2-pin) | COM | 24 VAC common return |
| JP1 | 3-pin 2.54 mm header | PWR-SEL | Pins 1-2 = barrel; pins 2-3 = screw terminal |
| SW1 | 6 mm tact switch | BOOT | IO0 to GND |
| SW2 | 6 mm tact switch | RST | EN to GND |

---

## Key ICs — LCSC Part Numbers

| Ref | Part | LCSC |
|-----|------|------|
| U1 | ESP32-WROOM-32E | C701341 |
| U2 | CH340C USB-UART | C84681 |
| U3 | AMS1117-3.3 LDO | C6186 |
| U4 | LM2596S-5.0 buck | C2621 |
| U5 | ULN2803A Darlington | C56026 |
| D1 | MB10F bridge rectifier | C441951 |
| D3 | USBLC6-2SC6 ESD protection | C2827693 |
| K1–K8 | HF46F-G/5-HS1 relay | C165255 |
| J1 | USB-C receptacle 16-pin | C165948 |

Full BOM with quantities and passives: `hardware/bom/SprinKlr8_BOM_JLCPCB.csv`

---

## Critical Design Notes

1. **GPIO12 pull-down (R5, 10 kΩ to GND):** GPIO12 is an ESP32 strapping pin. If it reads HIGH at boot the module selects a 1.8 V flash voltage and will not start. The pull-down is mandatory.

2. **ULN2803A COM pin (U5 pin 10) to +5V:** The ULN2803A uses +5V as the flyback clamp reference for its internal suppression diodes. Leaving it unconnected causes relay coil voltage spikes that can damage the device.

3. **LM2596 switching loop:** Keep U4 OUTPUT → D2 → L1 → C2 physically tight on the PCB. Do not route sensitive signal traces through this area.

4. **USB-C CC resistors (R1, R2, 5.1 kΩ to GND):** Required for USB-C charger compatibility. Without them many USB-C supplies will not deliver power.

5. **Antenna keep-out:** No copper (any layer), vias, or components within 15 mm of the ESP32 module's antenna end.

---

## Revision History

| Rev | Date | Changes |
|-----|------|---------|
| 0.1 | 2026-03 | Initial layout |
| 0.2 | 2026-05 | Current production revision — minor routing fixes |
