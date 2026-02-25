# ESP32-S3 BMC Design Research

## ESP32-S3-WROOM-1 Module Overview

The **ESP32-S3-WROOM-1** is a WiFi+BLE module based on the ESP32-S3 SoC. It's used
as the BMC (Baseboard Management Controller) for the Agentosaurus cluster board.

### Key Specifications

| Parameter | Value |
|-----------|-------|
| SoC | ESP32-S3 (Xtensa LX7 dual-core, 240 MHz) |
| Flash | 4/8/16 MB (on-module) |
| PSRAM | 2/8 MB (optional, on-module) |
| WiFi | 802.11 b/g/n (2.4 GHz) |
| Bluetooth | BLE 5.0 |
| USB | USB-OTG (USB 2.0 Full Speed, 12 Mbps) |
| GPIO | Up to 45 GPIOs |
| UART | 3× UART controllers |
| I2C | 2× I2C controllers |
| SPI | 4× SPI controllers |
| ADC | 2× 12-bit SAR ADC (20 channels) |
| Operating voltage | 3.3V |
| Module size | 18mm × 25.5mm × 3.1mm |
| Antenna | PCB antenna or U.FL connector (depends on variant) |

## GPIO Allocation for BMC Functions

### Required GPIO Summary

| Function | GPIOs Needed | ESP32-S3 Pins |
|----------|-------------|---------------|
| USB-OTG (D+/D-) | 2 | GPIO19 (D-), GPIO20 (D+) |
| UART TX to mux | 1 | GPIO17 (UART1_TX) |
| UART RX from mux | 1 | GPIO18 (UART1_RX) |
| UART mux select | 2 | GPIO4, GPIO5 (A0, A1 for 4:1 mux) |
| USB mux select | 2 | GPIO6, GPIO7 (SEL for 2× TS3USB221) |
| USB mux enable | 2 | GPIO8, GPIO9 (OE for 2× TS3USB221) |
| Power switch enable | 4 | GPIO10-GPIO13 (TPS22918 EN per slot) |
| WS2812B LED data | 1 | GPIO14 (data out, cascaded) |
| Switch MDIO CLK | 1 | GPIO15 (MDIO clock to RTL8367S) |
| Switch MDIO DATA | 1 | GPIO16 (MDIO data to RTL8367S) |
| CM4 GLOBAL_EN | 4 | GPIO35-GPIO38 (per-slot enable) |
| CM4 nEXTRST | 4 | GPIO39-GPIO42 (per-slot reset) |
| Boot mode strapping | 2 | GPIO0 (BOOT), GPIO46 (LOG_LEVEL) |
| **Total** | **~27** | Well within 45 GPIO limit |

### USB-OTG for Management Port

The ESP32-S3 has native USB-OTG support on GPIO19/GPIO20:
- Acts as USB device when connected to a host PC
- Provides serial console (CDC-ACM) for BMC command interface
- Can also provide USB mass storage for firmware updates

### UART Multiplexing

To provide serial console access to all 4 CM4 modules using a single UART:

```
ESP32-S3                 4:1 UART Mux              CM4 Modules
┌──────┐              ┌──────────────┐
│UART1 │──TX──────────┤ TX_IN    Y0 ├──── CM4 Slot 1 RXD
│      │──RX──────────┤ RX_OUT   Y1 ├──── CM4 Slot 2 RXD
│      │              │          Y2 ├──── CM4 Slot 3 RXD
│ GPIO4├──A0──────────┤ A0       Y3 ├──── CM4 Slot 4 RXD
│ GPIO5├──A1──────────┤ A1          │
│      │              │             │
│      │              │ RX_IN0 ◄────┤──── CM4 Slot 1 TXD
│      │              │ RX_IN1 ◄────┤──── CM4 Slot 2 TXD
│      │              │ RX_IN2 ◄────┤──── CM4 Slot 3 TXD
│      │              │ RX_IN3 ◄────┤──── CM4 Slot 4 TXD
└──────┘              └──────────────┘
```

**UART Mux IC Options:**
- **TI SN74LV4052A** — Dual 4:1 analog mux (one for TX, one for RX)
- **NXP 74HC4052** — Similar dual 4:1 mux
- Simple approach: use two 74HC4052 channels (TX and RX separately)

### Boot Mode Configuration

| GPIO | State | Mode |
|------|-------|------|
| GPIO0 | LOW | Download mode (firmware flash) |
| GPIO0 | HIGH | Normal boot (SPI flash) |
| GPIO46 | LOW | Default log level |

## Support IC: TPS22918 — Per-Slot Power Switch

### Overview
The **TI TPS22918** is a 5.5V, 2A load switch with quick output discharge.

### Key Specs
| Parameter | Value |
|-----------|-------|
| Input voltage | 1.0V to 5.5V |
| Max continuous current | 2A |
| On-resistance | 52mΩ typical |
| Package | SOT-23-5 (DBV) |
| Enable | Active HIGH |
| Quick output discharge | Yes (when OFF) |
| Rise time | ~1ms (adjustable with CT pin) |

### Pin Assignment (SOT-23-5)
| Pin | Name | Function |
|-----|------|----------|
| 1 | VIN | Input supply (5V from TPS54360) |
| 2 | GND | Ground |
| 3 | EN | Enable (HIGH = ON, from ESP32-S3 GPIO) |
| 4 | CT | Rise time control (capacitor to GND) |
| 5 | VOUT | Output supply (5V to CM4 slot) |

### Typical Circuit
```
VCC5V0 ──┬── VIN (1)
          │
          └── 100nF
              │
             GND

ESP32 GPIO ── 10kΩ ── EN (3)

CT (4) ── 10nF ── GND   (sets ~1ms rise time)

VOUT (5) ──┬── 100µF ── GND
            │
            └── To CM4 VCC5V0 pins
```

### Per-Slot Control
- ESP32-S3 drives EN pin HIGH to power on a CM4 slot
- Quick output discharge ensures clean power-off
- 4× TPS22918 needed (one per slot)

## Support IC: TS3USB221 — USB 2.0 Multiplexer

### Overview
The **TI TS3USB221** is a USB 2.0 high-speed (480 Mbps) 1:2 multiplexer/demultiplexer.

### Key Specs
| Parameter | Value |
|-----------|-------|
| Data rate | Up to 480 Mbps (USB 2.0 HS) |
| On-resistance | 6Ω typical |
| Bandwidth | 900 MHz |
| Supply | 3.0V to 3.6V |
| Package | QFN-10 (1.8×1.4mm) |
| Select | Active HIGH/LOW |

### Pin Assignment (QFN-10)
| Pin | Name | Function |
|-----|------|----------|
| 1 | D_P | Common USB D+ |
| 2 | D_N | Common USB D- |
| 3 | GND | Ground |
| 4 | S | Select (LOW = port 1, HIGH = port 2) |
| 5 | OE | Output enable (LOW = enabled) |
| 6 | VCC | 3.3V supply |
| 7 | D2_N | Port 2 USB D- |
| 8 | D2_P | Port 2 USB D+ |
| 9 | D1_N | Port 1 USB D- |
| 10 | D1_P | Port 1 USB D+ |

### USB Mux Architecture for 4 Slots

Need 2× TS3USB221 in a tree topology:

```
ESP32-S3              TS3USB221 #1           TS3USB221 #2a    TS3USB221 #2b
USB-OTG ───── D ────┤ Common              ┌──┤ Common
                     │                     │  │
              SEL1 ──┤ S        Port 1 ────┘  │ Port 1 ──── CM4 Slot 1
                     │          Port 2 ───────┤ Port 2 ──── CM4 Slot 2
                     │                        │
                     │ Port 2 ────────────────┤ TS3USB221 #2b
                     │                        │ Common
                     │                  SEL2──┤ S
                     │                        │ Port 1 ──── CM4 Slot 3
                     │                        │ Port 2 ──── CM4 Slot 4
                     └────────────────────────┘
```

Actually, simpler: Use 3× TS3USB221 in 1→2→4 tree, OR use 2× TS3USB221 with
external select logic. The simplest approach:

**Option A (2-level tree):** 1× first-level mux + 2× second-level mux = 3 ICs
**Option B (analog mux):** Use a TS3USB221 + a 4:1 USB mux like FSUSB42

For simplicity, use **3× TS3USB221** in a 1:2:4 tree.

## Support IC: WS2812B — Addressable RGB LED

### Overview
The **WS2812B** is a 5050-size (5mm × 5mm) addressable RGB LED with integrated driver.

### Key Specs
| Parameter | Value |
|-----------|-------|
| Supply voltage | 3.5V to 5.3V |
| LED current | 12mA per color (R/G/B) |
| Data rate | 800 kbps (NRZ) |
| Cascade | Unlimited (data passes through) |
| Package | 5050 SMD (5mm × 5mm) |

### Pin Assignment
| Pin | Name | Function |
|-----|------|----------|
| 1 | VDD | 5V power |
| 2 | DOUT | Data output (to next LED) |
| 3 | GND | Ground |
| 4 | DIN | Data input (from controller or previous LED) |

### Cascade Connection for 4 Status LEDs
```
ESP32-S3 GPIO14 ── DIN[1] → DOUT[1] ── DIN[2] → DOUT[2] ── DIN[3] → DOUT[3] ── DIN[4]
                    LED 1      LED 2      LED 3      LED 4
                   (Slot 1)   (Slot 2)   (Slot 3)   (Slot 4)
```

### Decoupling
- 100nF capacitor on each LED's VDD pin
- Optional 470Ω series resistor on DIN of first LED

### Level Shifting Note
WS2812B data threshold is ~0.7×VDD for logic HIGH. At 5V supply, this requires
3.5V input. The ESP32-S3 outputs 3.3V which is marginal. Solutions:
1. Power WS2812B from 3.3V instead of 5V (reduces brightness but ensures reliable data)
2. Add a level shifter (74HCT125 or single transistor) on the data line
3. Use WS2812B-V5 variant which has lower logic threshold

## Reference Schematic: ESP32-S3-WROOM-1 Minimal Circuit

```
USB-C Receptacle                          ESP32-S3-WROOM-1
┌──────────┐                             ┌─────────────────┐
│ VBUS ────┼── 5V (via Schottky) ───┐    │                 │
│ D+ ──────┼── 22Ω ── GPIO20 (D+)  │    │  3V3 ── VDD     │
│ D- ──────┼── 22Ω ── GPIO19 (D-)  │    │  GND ── GND     │
│ CC1 ─────┼── 5.1kΩ ── GND        │    │                 │
│ CC2 ─────┼── 5.1kΩ ── GND        │    │  GPIO0 ── BOOT  │
│ GND ─────┼── GND                  │    │  EN ──── RESET  │
│ SHIELD ──┼── GND (via 1MΩ)       │    │                 │
└──────────┘                        │    └─────────────────┘
                                    │
                             3.3V LDO (AMS1117-3.3)
                                    │
                                  3.3V
```

### Reset Circuit
- 10kΩ pull-up on EN pin to 3.3V
- 100nF cap on EN to GND (debounce)
- Optional reset button to GND

### Boot Button
- GPIO0 with 10kΩ pull-up to 3.3V
- Button to GND for download mode
