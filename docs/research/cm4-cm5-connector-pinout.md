# CM4 & CM5 Connector Pinout Research

## Raspberry Pi CM4 Connector Overview

The CM4 uses **two Hirose DF40C-100DS-0.4V** board-to-board connectors (100 pins each,
200 pins total). On the carrier/baseboard side, the mating connector is the
**DF40HC(3.0)-100DS-0.4V** receptacle.

- **Connector 1 (J1):** Pins 1-100 — Power, Ethernet, USB, PCIe, HDMI0/1, control
- **Connector 2 (J2):** Pins 101-200 — GPIO, SD card, Camera (CSI), Display (DSI)

### Connector Pitch & Dimensions
- Pitch: 0.4mm
- Stacking height: 1.5mm or 3.0mm (board uses 1.5mm)
- 2 rows × 50 pins per connector

## CM4 Connector 1 (J1) — Key Pin Assignments

| Pin | Signal | Pin | Signal |
|-----|--------|-----|--------|
| 1 | GND | 2 | GND |
| 3 | GND | 4 | GND |
| 5 | GND | 6 | GND |
| 7 | GND | 8 | GND |
| 9 | GND | 10 | GND |
| 11 | GND | 12 | GND |
| 13 | GND | 14 | GND |
| 15 | HDMI1_CK_N | 16 | HDMI0_CK_N |
| 17 | HDMI1_CK_P | 18 | HDMI0_CK_P |
| 19 | GND | 20 | GND |
| 21 | HDMI1_D0_N | 22 | HDMI0_D0_N |
| 23 | HDMI1_D0_P | 24 | HDMI0_D0_P |
| 25 | GND | 26 | GND |
| 27 | HDMI1_D1_N | 28 | HDMI0_D1_N |
| 29 | HDMI1_D1_P | 30 | HDMI0_D1_P |
| 31 | GND | 32 | GND |
| 33 | HDMI1_D2_N | 34 | HDMI0_D2_N |
| 35 | HDMI1_D2_P | 36 | HDMI0_D2_P |
| 37 | GND | 38 | GND |
| 39 | HDMI1_CEC | 40 | HDMI0_CEC |
| 41 | HDMI1_SDA | 42 | HDMI0_SDA |
| 43 | HDMI1_SCL | 44 | HDMI0_SCL |
| 45 | HDMI1_HOTPLUG | 46 | HDMI0_HOTPLUG |
| 47 | PCIE_CLK_nREQ | 48 | VCC5V0 |
| 49 | USB2_N | 50 | VCC5V0 |
| 51 | USB2_P | 52 | VCC5V0 |
| 53 | USBOTG_ID | 54 | VCC5V0 |
| 55 | GND | 56 | VCC5V0 |
| 57 | PCIE_CLK_N | 58 | VCC5V0 |
| 59 | PCIE_CLK_P | 60 | VCC5V0 |
| 61 | GND | 62 | VCC5V0 |
| 63 | PCIE_RX_N | 64 | VCC5V0 |
| 65 | PCIE_RX_P | 66 | VCC5V0 |
| 67 | GND | 68 | VCC5V0 |
| 69 | PCIE_TX_N | 70 | VCC5V0 |
| 71 | PCIE_TX_P | 72 | VCC5V0 |
| 73 | GND | 74 | VCC5V0 |
| 75 | TRD3_N | 76 | PSCIE_nRST |
| 77 | TRD3_P | 78 | nEXTRST |
| 79 | GND | 80 | PCIE_WAKE |
| 81 | TRD2_N | 82 | nPWR_LED |
| 83 | TRD2_P | 84 | BT_nDis |
| 85 | GND | 86 | WL_nDis |
| 87 | TRD1_N | 88 | GLOBAL_EN |
| 89 | TRD1_P | 90 | AIN1 |
| 91 | GND | 92 | AIN0 |
| 93 | TRD0_N | 94 | SYNC_OUT |
| 95 | TRD0_P | 96 | SYNC_IN |
| 97 | ETH_LEDY | 98 | TV_OUT |
| 99 | ETH_LEDG | 100 | RUN_PG |

### Key Signal Groups on J1:
- **Power (VCC5V0):** Pins 48, 50, 52, 54, 56, 58, 60, 62, 64, 66, 68, 70, 72, 74
- **Ethernet PHY (TRD):** Pins 75-77, 81-83, 87-89, 93-95 (transformer-side pairs)
- **PCIe Gen 2.0 x1:** Pins 57-59 (CLK), 63-65 (RX), 69-71 (TX), 47 (CLK_nREQ), 76 (nRST), 80 (WAKE)
- **USB 2.0:** Pins 49-51 (D-/D+), 53 (OTG_ID)
- **HDMI0:** Pins 16-46 (even) — data, clock, control
- **HDMI1:** Pins 15-45 (odd) — data, clock, control
- **Control:** Pin 88 (GLOBAL_EN), 100 (RUN_PG), 78 (nEXTRST)

## CM4 Connector 2 (J2) — Key Pin Assignments

| Pin Range | Signal Group |
|-----------|-------------|
| 101-114 | GND (ground pins) |
| 115-126 | Camera CSI-1 (4-lane): CAM1_D0-D3, CAM1_C |
| 127-134 | Camera CSI-0 (2-lane): CAM0_D0-D1, CAM0_C |
| 135-140 | Display DSI-1: DSI1_D0-D1, DSI1_C |
| 141-148 | Display DSI-0: DSI0_D0-D1, DSI0_C |
| 149-150 | DSI extended lanes |
| 151-152 | VCC1V8 (1.8V from CM4) |
| 153-154 | 3V3_RPi (3.3V from CM4) |
| 155-160 | GPIO (various) |
| 161-168 | SD card: SD_CLK, SD_CMD, SD_DAT0-3 |
| 169-170 | SD_PWR_ON, CAM_GPIO |
| 171-172 | EEPROM_nWP, RESERVED |
| 173-174 | SCL0, SDA0 (I2C0) |
| 175-176 | ID_SC, ID_SD (HAT EEPROM I2C) |
| 177-200 | GPIO2-GPIO27 (general purpose I/O) |

### Key Signal Groups on J2:
- **GPIO bank:** 26 GPIOs (GPIO2-GPIO27) for I2C, SPI, UART, etc.
- **SD card:** 6 data lines + power control
- **Camera/Display:** CSI/DSI lanes (not used in cluster design)
- **UART:** GPIO14 = RPi_TXD, GPIO15 = RPi_RXD (directly from GPIO pins)

## Critical: Ethernet Interface Type

The CM4 has a **Broadcom BCM54210PE integrated Gigabit Ethernet PHY**. The pins
exposed on the DF40 connectors (TRD0-3) are the **transformer-side** differential
pairs — these are the outputs AFTER the PHY, meant to connect directly to Ethernet
magnetics and then to an RJ45 jack.

The CM4 does **NOT** expose RGMII or any MAC-level Ethernet interface on its connectors.
This means:
- You CANNOT connect directly to an RGMII-only switch port
- You CAN connect through magnetics to another PHY port (PHY-to-PHY)
- The RTL8367S has 5 integrated PHYs, so this works perfectly

### Differential Pair Mapping:
| CM4 Signal | Ethernet Pair | Purpose |
|-----------|--------------|---------|
| TRD0_P/N | Pair A (1,2) | Bidirectional data |
| TRD1_P/N | Pair B (3,6) | Bidirectional data |
| TRD2_P/N | Pair C (4,5) | Bidirectional data (Gigabit) |
| TRD3_P/N | Pair D (7,8) | Bidirectional data (Gigabit) |

## Orange Pi CM5 Compatibility

The **Orange Pi CM5** uses the **Rockchip RK3588S** SoC and is designed to be
mechanically and electrically compatible with the Raspberry Pi CM4 form factor.

### Key Differences:
1. **Same DF40 connectors** — physically pin-compatible
2. **Ethernet:** The RK3588S has dual Gigabit Ethernet MACs with RGMII outputs.
   However, on the Orange Pi CM5, to maintain CM4 compatibility, one MAC is connected
   through an onboard RTL8211F PHY, and the transformer-side pairs are routed to
   the same TRD0-3 pins as the CM4. This means the same PHY-to-PHY magnetics approach
   works for the CM5 as well.
3. **PCIe:** RK3588S supports PCIe 3.0, but on the CM4-compatible connector it's
   limited to the same single-lane PCIe that the CM4 uses.
4. **USB:** Same USB 2.0 pair on the same pins.
5. **HDMI:** Same HDMI signals on compatible pins.

### Compatibility Summary:
| Feature | CM4 | Orange Pi CM5 | Compatible? |
|---------|-----|--------------|-------------|
| DF40 connectors | Yes | Yes | ✅ |
| Power (5V) | Same pins | Same pins | ✅ |
| Ethernet (TRD) | BCM54210PE PHY | RTL8211F PHY | ✅ |
| PCIe (x1) | Gen 2.0 | Gen 3.0 (backward compat) | ✅ |
| USB 2.0 | Same pins | Same pins | ✅ |
| HDMI | 2× HDMI | 1× HDMI + 1× varies | ⚠️ |
| GPIO | BCM2711 | RK3588S | ⚠️ (software) |

**Conclusion:** The Agentosaurus board will work with both CM4 and Orange Pi CM5
for the core cluster functionality (Ethernet, PCIe/NVMe, USB, power). GPIO-level
features may need different software/firmware for CM5.
