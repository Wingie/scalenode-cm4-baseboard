# Agentosaurus Edge Node — Architecture Overview

## Design Goal

Transform the single-node Antmicro Scalenode CM4 Baseboard into a **4-node CM4/CM5
cluster board** in Mini-ITX form factor, with integrated Ethernet switching, a
BMC (Baseboard Management Controller), and ATX power input.

## Block Diagram

```
                    ATX 24-pin
                        │
                        ▼
              ┌─────────────────────┐
              │   Power Supply      │
              │   TPS54360          │
              │   12V → 5V (8A)    │
              │   + LDOs for 3.3V  │
              └────────┬────────────┘
                       │ 5V rail
         ┌─────────────┼─────────────┐
         │             │             │
    ┌────▼────┐   ┌────▼────┐   ┌───▼────┐
    │ TPS22918│   │ TPS22918│   │3.3V LDO│
    │ Slot 1  │   │ Slot 2  │   │for BMC │
    └────┬────┘   └────┬────┘   └───┬────┘
         │             │             │
    ┌────▼────┐   ┌────▼────┐   ┌───▼─────────┐
    │  CM4    │   │  CM4    │   │ ESP32-S3     │
    │ Slot 1  │   │ Slot 2  │   │ BMC          │
    │(DF40x2) │   │(DF40x2) │   │              │
    └────┬────┘   └────┬────┘   │  ┌─USB-C     │
         │             │        │  │(Mgmt)     │
    ┌────▼────┐   ┌────▼────┐   └──┴───────────┘
    │  CM4    │   │  CM4    │        │
    │ Slot 3  │   │ Slot 4  │   ┌────▼──────────┐
    │(DF40x2) │   │(DF40x2) │   │ TS3USB221 x2  │
    └────┬────┘   └────┬────┘   │ USB Mux       │
         │             │        │ (eMMC flash)   │
         │             │        └────────────────┘
         │             │
    ┌────▼─────────────▼────────────────────┐
    │           RTL8367S                     │
    │     5-Port Gigabit Switch             │
    │                                       │
    │  PHY0 ← CM4 Slot 1 (via magnetics)   │
    │  PHY1 ← CM4 Slot 2 (via magnetics)   │
    │  PHY2 ← CM4 Slot 3 (via magnetics)   │
    │  PHY3 ← CM4 Slot 4 (via magnetics)   │
    │  PHY4 → RJ45 Uplink (via magnetics)  │
    │                                       │
    └───────────────────────────────────────┘
```

## Key Architecture Decisions

### 1. Ethernet: PHY-to-PHY through Magnetics
The CM4 has an integrated Ethernet PHY (BCM54210PE) that outputs transformer-side
differential pairs (TRD0-3), NOT RGMII. The RTL8367S has 5 integrated PHYs.

**Approach:** Connect each CM4's TRD0-3 outputs through 1:1 Ethernet magnetics to
an RTL8367S PHY port. The 5th PHY port connects through magnetics to an RJ45 uplink
jack. This is the same approach used by the Turing Pi 2.

### 2. Power: ATX Input with Per-Slot Switching
- ATX 24-pin provides 12V main power and 5V standby
- TPS54360 buck converter: 12V → 5V at ~8A for all 4 CM4 slots
- TPS22918 load switches: One per CM4 slot for individual power control
- 3.3V LDO from 5V for ESP32-S3 BMC and switch IC
- Each CM4 generates its own 3.3V internally (3V3_RPi)

### 3. BMC: ESP32-S3-WROOM-1
The ESP32-S3 provides:
- **USB-C management port** (USB-OTG) for host connection
- **UART mux** to 4 CM4 serial consoles (via GPIO UART + mux IC)
- **USB mux** (TS3USB221) to flash CM4 eMMC via USB
- **Power control** via TPS22918 enable pins (4 GPIOs)
- **Status LEDs** via WS2812B addressable LEDs (1 data pin, cascaded)
- **I2C bus** for monitoring/configuration
- **WiFi** for remote management (optional)

### 4. Per-Slot Signals (from upstream analysis)
Each CM4 slot needs these signals from the upstream design:

| Signal Group | Signals | Notes |
|-------------|---------|-------|
| **Ethernet** | TRD0-3 P/N (8 lines) | Through magnetics to RTL8367S |
| **PCIe** | TX/RX P/N, CLK P/N, nREQ, WAKE, nRST (9 lines) | To M.2 NVMe slot |
| **USB 2.0** | USB2_P/N (2 lines) | To BMC USB mux |
| **UART** | RPi_TXD, RPi_RXD (2 lines) | To BMC UART mux |
| **SD Card** | CMD, CLK, DAT0-3, PWR_ON (7 lines) | To microSD slot |
| **Power** | VCC5V0 (from TPS22918), 3V3_SSD | From power supply |
| **Control** | GLOBAL_EN, RUN_PG, nEXTRST | BMC control lines |
| **HDMI** | Slot 1 only: HDMI0 full signals | To rear HDMI connector |

### 5. Signals NOT Carried Over
- HDMI1 (second display) — not needed
- CSI Camera (CAM0, CAM1) — not needed for cluster
- DSI Display (DSI0, DSI1) — not needed for cluster
- Full GPIO break-out — only BMC control GPIOs needed
- PoE circuitry — replaced by ATX power
- TV_OUT — obsolete

### 6. Form Factor: Mini-ITX (170mm × 170mm)
- Standard Mini-ITX mounting holes
- ATX 24-pin power connector at board edge
- Rear I/O: RJ45 uplink, USB-C management, HDMI (slot 1), status LEDs
- 4x CM4 slots with DF40 connectors arranged in 2×2 grid
- 4x M.2 NVMe slots adjacent to each CM4
- 4x microSD slots

### 7. PCB: JLCPCB 6-Layer
- 6-layer stackup for controlled impedance
- Signal-Ground-Signal-Signal-Ground-Signal
- 100Ω differential for Ethernet
- 85Ω differential for PCIe
- 90Ω differential for USB 2.0

## Schematic Sheet Structure (KiCad 8 Hierarchical)

```
Root (agentosaurus.kicad_sch)
├── cm4-slot.kicad_sch          × 4 instances (hierarchical)
│   ├── CM4 DF40 connectors
│   ├── Ethernet magnetics
│   ├── M.2 NVMe slot
│   ├── microSD slot
│   └── Per-slot decoupling
├── ethernet-switch.kicad_sch   × 1
│   ├── RTL8367S IC
│   ├── Uplink magnetics + RJ45
│   └── Switch decoupling
├── bmc.kicad_sch               × 1
│   ├── ESP32-S3-WROOM-1
│   ├── USB-C connector
│   ├── UART mux
│   ├── TS3USB221 USB mux × 2
│   ├── TPS22918 load switches × 4
│   └── WS2812B status LEDs × 4
├── power.kicad_sch             × 1
│   ├── ATX 24-pin connector
│   ├── TPS54360 12V→5V
│   ├── 3.3V LDO
│   └── Power sequencing
└── mechanical.kicad_sch        × 1
    ├── Mounting holes
    ├── Board outline
    └── Logos/markings
```

## Component Summary

| Component | Qty | Function |
|-----------|-----|----------|
| CM4 DF40 connector pair | 4 | Compute module sockets |
| RTL8367S | 1 | 5-port Gigabit Ethernet switch |
| ESP32-S3-WROOM-1 | 1 | BMC controller |
| TPS54360 | 1 | 12V→5V 3.5A buck converter |
| TPS22918 | 4 | Per-slot power switches |
| TS3USB221 | 2 | USB 2.0 mux (4 slots → BMC) |
| WS2812B | 4 | Per-slot status LEDs |
| M.2 Key-M connector | 4 | NVMe SSD slots |
| microSD slot | 4 | SD card per slot |
| ATX 24-pin | 1 | Power input |
| USB-C receptacle | 1 | BMC management port |
| RJ45 w/ magnetics | 1 | Ethernet uplink |
| HDMI connector | 1 | Slot 1 display output |
| Ethernet magnetics | 5 | 4 slots + 1 uplink |

## License

CERN-OHL-P v2 (Permissive) — allows commercial use without copyleft requirements.
