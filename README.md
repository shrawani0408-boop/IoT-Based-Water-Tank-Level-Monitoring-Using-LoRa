# IoT-Based Water Tank Level Measurement System Using LoRa

## Overview

This project implements a **four-tank water-level monitoring system** using ESP32 microcontrollers, analog pressure sensors, and SX1278 LoRa modules operating at 433 MHz.

Four independent transmitter nodes measure the water levels of separate tanks and send their readings wirelessly to a central ESP32 receiver. The receiver identifies each tank using a unique Tank ID and displays its water-level percentage, raw sensor reading, and received signal strength (RSSI) through the Serial Monitor. It also detects communication timeouts when a previously communicating transmitter stops sending data.

## Objectives

- Monitor the water levels of four separate tanks.
- Convert analog pressure-sensor readings into estimated water-level percentages.
- Transmit tank-specific data wirelessly using 433 MHz LoRa.
- Identify each tank using a unique transmitter ID.
- Display received sensor readings and LoRa signal strength.
- Detect missing data using receiver-side timeout monitoring.

## System Architecture

The system consists of four transmitter nodes and one central receiver.

```text
       TANK 1                           TANK 2
  Pressure Sensor                  Pressure Sensor
        ↓                                ↓
    ESP32 + LoRa                    ESP32 + LoRa
     Tank ID 1                       Tank ID 2
         \                                /
          \                              /
           ─────── 433 MHz LoRa ─────────
                          ↓
                  Central ESP32
                   + LoRa Module
                          ↓
                   Serial Monitor
                          ↑
           ─────── 433 MHz LoRa ─────────
          /                              \
     ESP32 + LoRa                    ESP32 + LoRa
     Tank ID 3                       Tank ID 4
        ↑                                ↑
  Pressure Sensor                  Pressure Sensor
       TANK 3                           TANK 4
```

Each transmitter independently reads its pressure sensor, calculates the estimated water level, and transmits a packet. The receiver parses the packet, identifies the corresponding tank, and updates its stored reading.

## Hardware Requirements

| Component | Quantity | Purpose |
|---|---:|---|
| ESP32 development board | 5 | Four transmitters and one receiver |
| SX1278 LoRa module, 433 MHz | 5 | Wireless communication |
| Analog pressure sensor | 4 | Water-level measurement |
| Jumper wires and connectors | As required | Electrical connections |
| Suitable power supply | 5 sets | Power for the nodes |
| Laptop or computer | 1 | Serial monitoring and debugging |

## Software Requirements

- Arduino IDE
- ESP32 board support package
- Arduino LoRa library
- SPI library
- Serial Monitor

## Working Principle

1. Each pressure sensor measures the pressure associated with the water column in its tank.
2. The ESP32 reads the analog sensor output using its built-in ADC.
3. Ten ADC samples are collected, with a 10 ms delay between samples, and averaged to reduce reading fluctuations.
4. The average ADC reading is converted into an estimated water-level percentage using empty-tank and full-tank calibration values.
5. The percentage is constrained to the range of 0% to 100%.
6. Each transmitter creates a comma-separated packet containing the Tank ID, raw ADC reading, and water-level percentage.
7. The SX1278 LoRa module transmits the packet at 433 MHz.
8. The central receiver parses the received packet and validates its format and Tank ID.
9. The receiver displays the raw reading, estimated water level, and RSSI.
10. The receiver marks a tank as timed out if no new packet arrives within the configured 12-second timeout after communication has been established.

## Data Packet Format

The transmitter sends data in the following format:

```text
TankID,RawValue,WaterLevel
```

Example:

```text
1,1862,52.5
```

| Field | Example | Description |
|---|---|---|
| Tank ID | `1` | Identifies the transmitting tank |
| Raw value | `1862` | Averaged ADC reading |
| Water level | `52.5` | Estimated water level in percent |

The example readings are illustrative; actual values depend on the installed sensor and its calibration.

## Water-Level Calculation

The transmitter calculates the water-level percentage using two calibration points:

- `EMPTY_RAW`: ADC reading for the empty tank
- `FULL_RAW`: ADC reading for the full tank

The calculation is:

\[
\text{Water Level (\%)} =
\frac{\text{Current Reading}-\text{EMPTY\_RAW}}
{\text{FULL\_RAW}-\text{EMPTY\_RAW}}
\times 100
\]

The result is limited to the range of 0% to 100%.

The code currently uses these example calibration values:

```cpp
const int EMPTY_RAW = 1000;
const int FULL_RAW = 3000;
```

These values must be replaced with measurements from the actual sensor installation. The calculation assumes an approximately linear relationship between the sensor reading and water height.

## LoRa Communication

The system uses SX1278 LoRa modules configured for 433 MHz communication.

The transmitter and receiver share the following configuration:

| Parameter | Configuration |
|---|---|
| Frequency | 433 MHz |
| SCK | GPIO 18 |
| MISO | GPIO 19 |
| MOSI | GPIO 23 |
| NSS / SS | GPIO 5 |
| RESET | GPIO 14 |
| DIO0 | GPIO 26 |
| Packet CRC | Enabled |

Each transmitter uses a unique Tank ID from 1 to 4. The transmitters send readings every 8 seconds, with staggered initial transmission delays based on their IDs to reduce the likelihood of simultaneous transmissions.

Staggered startup alone does not guarantee collision-free communication. A polling mechanism, acknowledgements, and retransmission logic could improve reliability.

## Communication Timeout Detection

The receiver tracks the most recent packet received from each tank.

- Timeout threshold: 12 seconds
- When a previously received tank stops sending packets beyond the threshold, its displayed status changes to `Timeout`.
- When another valid packet arrives, the tank's status is updated and timeout reporting is reset.

The timeout indicates that fresh data has not arrived. It does not independently determine whether the cause is a power failure, transmitter fault, or wireless communication interruption.

## Sample Output

The following is illustrative output showing how the receiver displays readings from four tanks.

```text
Received packet: 1,1862,52.5
------ TANK DETAILS ------
Tank ID: 1
Raw Sensor Value: 1862
Water Level: 52.5%
Signal Strength (RSSI): -67 dBm
--------------------------

------ ALL TANK LEVELS ------
Tank 1: 52.5%
Tank 2: Waiting...
Tank 3: Waiting...
Tank 4: Waiting...
-----------------------------

Received packet: 2,2140,68.4
------ TANK DETAILS ------
Tank ID: 2
Raw Sensor Value: 2140
Water Level: 68.4%
Signal Strength (RSSI): -71 dBm
--------------------------

------ ALL TANK LEVELS ------
Tank 1: 52.5%
Tank 2: 68.4%
Tank 3: Waiting...
Tank 4: Waiting...
-----------------------------
```

As further packets arrive, the receiver updates the corresponding tank entry without replacing the readings of the other tanks.

## Repository Structure

A suggested repository structure is:

```text
LoRa-Water-Tank-Monitoring/
├── transmitter/
│   └── transmitter.ino
├── receiver/
│   └── receiver.ino
└── README.md
```

The transmitter code is shared across all four nodes. Change `TANK_ID` to `1`, `2`, `3`, or `4` before uploading it to each transmitter.

## Advantages

- Wireless monitoring reduces the need for long signal cables.
- Four tanks can be monitored from one central receiver.
- Unique IDs associate measurements with individual tanks.
- Averaging ADC samples helps reduce short-term reading fluctuations.
- RSSI provides an indication of received signal strength.
- Timeout detection identifies stale tank data.
- The design can be extended with dashboards, alerts, and automated control.

## Applications

- College and school water-storage systems
- Residential buildings and apartments
- Industrial water tanks
- Agricultural water storage and irrigation
- Remote water-distribution facilities
- Rainwater-harvesting systems

## Limitations and Future Scope

### Current limitations

- Water-level accuracy depends on pressure-sensor calibration and installation.
- The current percentage calculation assumes a linear sensor response.
- Transmitter startup staggering does not eliminate all possible packet collisions.
- The receiver uses the Serial Monitor rather than a dedicated graphical dashboard.
- Timeout detection indicates missing data but cannot identify the exact cause.
- The code does not implement acknowledgements or retransmission handling.

### Future enhancements

- Develop a web-based dashboard for live monitoring of all four tanks.
- Add low-level and overflow alerts.
- Implement automatic pump control.
- Add data logging and historical water-level graphs.
- Introduce polling, acknowledgements, and retry mechanisms.
- Add mobile notifications for communication failures and critical water levels.

## Conclusion

The IoT-Based Water Tank Level Measurement System demonstrates a distributed embedded architecture using ESP32 microcontrollers, analog pressure sensors, and 433 MHz LoRa communication. Four transmitter nodes measure and transmit tank-specific water-level data to a central receiver, which identifies each tank, displays its readings, and detects communication timeouts.

The project combines sensor interfacing, ADC processing, calibration, packet-based wireless telemetry, and embedded communication to support centralized monitoring of multiple water tanks.

## Disclaimer

The reported sample readings are illustrative and should not be interpreted as measured experimental results unless verified on the actual hardware. Reliable water-level measurement requires appropriate sensor calibration and testing under real installation conditions.
