# TH03_ZBPS1.0 Hardware Documentation

This file documents the reverse-engineered hardware layout for the **TH03_ZBPS1.0** temperature and humidity sensor board (powered by the Telink TLSR8656 / TLSR825x series SoC).

## Hardware Components

### Microcontroller (MCU)
- **IC:** Telink TLSR825x (or TLSR8656)
- **Features:** Bluetooth Low Energy (BLE), Deep Sleep Retention

### LCD Controller
- **IC:** VKL060 / BL55028 (SSOP24 package)
- **I2C Address:** `0x7C` (8-bit) / `0x3E` (7-bit)
- **COMs/SEGs:** 4 COMs, 15 SEGs (60 segments total)
- **Driver Quirk 1 (RAM Address):** The display RAM does **NOT** start at `0x00`. According to the datasheet, the Display RAM addresses start at **`0x0B`**. Sending `0x00` as the ADSET address byte causes the controller to silently ignore all segment data.
- **Driver Quirk 2 (I2C Speed & NACKs):** The VKL060 is sensitive to I2C clock speeds. It requires a relatively fast clock (e.g., `2µs` period / 500kHz). Slower clocks (`24µs`) can cause the controller to NACK transfers.

### Sensor (Temperature & Humidity)
- **IC:** GXHTV4 (package marked GV4)
- **Details:** The GXHTV4 and its sibling GXHT40 (package marked G40) are 16-bit relative humidity and temperature sensors manufactured by GXCAS (Beijing Galaxy-CAS Technology Co., Ltd.). 
- **Compatibility:** They are pin-to-pin and I²C command clones of the Sensirion SHT4x (SHT40 / SHT41) family.

## GPIO Mapping

| Component | Pin | Active State | Notes |
| :--- | :--- | :--- | :--- |
| **Pairing Button** | `PD3` | LOW (0) | Requires internal pull-up resistor (10K). Shorts to GND when pressed. A 1.75s hold triggers C/F swap, 5-7s triggers factory reset. |
| **I2C SDA (LCD)** | `PD7` | N/A | Connected to VKL060 controller |
| **I2C SCL (LCD)** | `PA1` | N/A | Connected to VKL060 controller |
| **Sensor Power** | `PB0` | HIGH (1) | Provides power to the GXHTV4 sensor. Must be driven HIGH before reading. Also often used for VBAT measurement. |
| **I2C SDA (Sensor)** | `PA0` | N/A | Connected to GXHTV4 |
| **I2C SCL (Sensor)** | `PD4` | N/A | Connected to GXHTV4 |
| **Crash Pins** | `PC0`, `PC1` | N/A | **DO NOT DRIVE THESE PINS.** Pulling these high/low or defining them as I/O immediately crashes the board. |

## Firmware Development Notes

### Deep Sleep & Variable Retention
The TLSR825x enters a deep sleep state between BLE advertising packets to save power. During this deep sleep, the CPU shuts down and standard SRAM (`.bss` / `.data`) is completely wiped and re-initialized upon wake.
- **Any variable that needs to persist across the BLE sleep cycle MUST be marked with the `RAM` macro** (which maps to `_attribute_data_retention_`). 
- Standard `static` or global variables will constantly reset to `0` every 1-2 seconds, which can silently break state machines or loops.
