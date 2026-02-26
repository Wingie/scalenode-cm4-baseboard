# Upstream Scalenode CM4 Baseboard — Signal Architecture

## Overview

The upstream Scalenode is a single-node CM4 carrier with 7 schematic sheets connected
via **global labels** (flat hierarchy, no sheet pins). All signals below were extracted
from the actual KiCad schematic files.

## Inter-Sheet Global Labels by Category

### Power Rails
| Global Label | Sheets Used | Description |
|-------------|-------------|-------------|
| `VCC5V0` | compute-module, poe, expansion-connectors, supply | Main 5V rail (from PoE or supply) |
| `3V3_RPi` | compute-module, poe, interfaces, expansion-connectors | 3.3V from CM4 module |
| `3V3_SSD` | supply, nvme-ssd, expansion-connectors | 3.3V regulated for NVMe SSD |
| `3V3_OUT` | expansion-connectors | 3.3V output on expansion connector |
| `VCC1V8` | compute-module | 1.8V from CM4 module |
| `VSS` | poe | PoE isolated ground |

### Ethernet (Transformer-side differential pairs)
| Global Label | Sheets Used | Description |
|-------------|-------------|-------------|
| `TRD0_P` / `TRD0_N` | compute-module, poe | Ethernet pair 0 (to magnetics/RJ45) |
| `TRD1_P` / `TRD1_N` | compute-module, poe | Ethernet pair 1 |
| `TRD2_P` / `TRD2_N` | compute-module, poe | Ethernet pair 2 |
| `TRD3_P` / `TRD3_N` | compute-module, poe | Ethernet pair 3 |
| `ETH_LEDG` | compute-module, poe | Ethernet green LED (link) |
| `ETH_LEDY` | compute-module, poe | Ethernet yellow LED (activity) |

**Note:** The upstream connects CM4 Ethernet directly to RJ45 magnetics in the PoE
sheet — NO switch IC. For the Agentosaurus, we intercept these signals and route
them to the RTL8367S RGMII ports instead.

### PCIe (CM4 to NVMe M.2)
| Global Label | Sheets Used | Description |
|-------------|-------------|-------------|
| `PCIE_TX_P` / `PCIE_TX_N` | compute-module, nvme-ssd | PCIe TX differential pair |
| `PCIE_RX_P` / `PCIE_RX_N` | compute-module, nvme-ssd | PCIe RX differential pair |
| `PCIE_CLK_P` / `PCIE_CLK_N` | compute-module, nvme-ssd | PCIe reference clock |
| `PCIE_CLK_nREQ` | compute-module, nvme-ssd | PCIe clock request (active low) |
| `PCIE_WAKE` | nvme-ssd | PCIe wake signal |
| `PSCIE_nRST` | compute-module, nvme-ssd | PCIe reset (active low) |

### USB
| Global Label | Sheets Used | Description |
|-------------|-------------|-------------|
| `USB2_P` / `USB2_N` | compute-module, interfaces | USB 2.0 data pair |
| `USBOTG_ID` | compute-module, interfaces | USB OTG ID pin |
| `Adapter_USB_P` / `Adapter_USB_N` | interfaces, expansion-connectors | USB to adapter/expansion |
| `Adapter_TXD` / `Adapter_RXD` | interfaces, expansion-connectors | UART via USB adapter (FTDI) |

### HDMI (dual output from CM4)
| Global Label | Sheets Used | Description |
|-------------|-------------|-------------|
| `HDMI0_D0_P/N`, `HDMI0_D1_P/N`, `HDMI0_D2_P/N` | compute-module, expansion-connectors | HDMI0 data lanes |
| `HDMI0_CK_P` / `HDMI0_CK_N` | compute-module, expansion-connectors | HDMI0 clock |
| `HDMI0_SCL` / `HDMI0_SDA` | compute-module, expansion-connectors | HDMI0 DDC (I2C) |
| `HDMI0_CEC` | compute-module, expansion-connectors | HDMI0 CEC |
| `HDMI0_HOTPLUG` | compute-module, expansion-connectors | HDMI0 hot plug detect |
| `HDMI1_D0_P/N`, `HDMI1_D1_P/N`, `HDMI1_D2_P/N` | compute-module | HDMI1 data lanes |
| `HDMI1_CK_P` / `HDMI1_CK_N` | compute-module | HDMI1 clock |
| `HDMI1_SCL` / `HDMI1_SDA` / `HDMI1_CEC` | compute-module | HDMI1 control |
| `HDMI1_HOTPLUG` | compute-module | HDMI1 hot plug detect |

### GPIO (directly from CM4 module)
| Global Label | Sheets Used | Description |
|-------------|-------------|-------------|
| `GPIO2` - `GPIO27` | compute-module, interfaces, expansion-connectors | General purpose I/O |
| `ID_SC` / `ID_SD` | compute-module | HAT ID EEPROM I2C |
| `SCL0` / `SDA0` | compute-module, expansion-connectors | I2C bus 0 |

### SD Card
| Global Label | Sheets Used | Description |
|-------------|-------------|-------------|
| `SD_CMD` | compute-module, interfaces | SD command line |
| `SD_CLK` | compute-module, interfaces | SD clock |
| `SD_DAT0` - `SD_DAT3` | compute-module, interfaces | SD data lines |
| `SD_PWR_ON` | compute-module | SD card power control |

### Camera (CSI)
| Global Label | Sheets Used | Description |
|-------------|-------------|-------------|
| `CAM0_C_P/N`, `CAM0_D0_P/N`, `CAM0_D1_P/N` | compute-module | CSI camera 0 (2-lane) |
| `CAM1_C_P/N`, `CAM1_D0_P/N` - `CAM1_D3_P/N` | compute-module | CSI camera 1 (4-lane) |
| `CAM_GPIO` | compute-module | Camera GPIO control |

### Display (DSI)
| Global Label | Sheets Used | Description |
|-------------|-------------|-------------|
| `DSI0_C_P/N`, `DSI0_D0_P/N`, `DSI0_D1_P/N` | compute-module | DSI display 0 (2-lane) |
| `DSI1_C_P/N`, `DSI1_D0_P/N`, `DSI1_D1_P/N` | compute-module | DSI display 1 (2-lane) |
| `DSI1_D3_P/N`, `DSI2_D1_P/N` | compute-module | DSI extended lanes |

### Control & Status
| Global Label | Sheets Used | Description |
|-------------|-------------|-------------|
| `GLOBAL_EN` | compute-module | Global enable (shutdown control) |
| `RUN_PG` | compute-module | Run/power good indicator |
| `nEXTRST` | compute-module | External reset (active low) |
| `nPWR_LED` | compute-module | Power LED control (active low) |
| `BT_nDis` / `WL_nDis` | compute-module | Bluetooth/WiFi disable |
| `PROG_MODE` | compute-module, interfaces | Programming mode select |
| `EEPROM_nWP` | compute-module | EEPROM write protect |
| `SYNC_IN` / `SYNC_OUT` | compute-module | Sync signals |
| `AIN0` / `AIN1` | compute-module | Analog inputs |
| `RESERVED` | compute-module | Reserved pin |
| `TV_OUT` | compute-module | Composite TV output |

## Signals to Carry Over to Agentosaurus (per CM4 slot)

### Must have (core functionality):
- **Ethernet:** TRD0-3 P/N → route to RTL8367S RGMII port (need level translation)
- **PCIe:** PCIE_TX/RX P/N, PCIE_CLK P/N, PCIE_CLK_nREQ, PCIE_WAKE, PSCIE_nRST → M.2 NVMe
- **USB:** USB2_P/N → BMC USB mux for eMMC flashing
- **Power:** VCC5V0, 3V3_RPi, 3V3_SSD
- **UART:** RPi_TXD, RPi_RXD → BMC UART mux for serial console
- **SD Card:** SD_CMD, SD_CLK, SD_DAT0-3, SD_PWR_ON → microSD slot
- **Control:** GLOBAL_EN, RUN_PG, nEXTRST

### Slot 1 only (HDMI):
- **HDMI0:** All HDMI0 signals → external HDMI connector

### Omit (not needed for cluster):
- **HDMI1:** Second HDMI output
- **Camera (CSI):** CAM0, CAM1 — not needed for cluster
- **Display (DSI):** DSI0, DSI1 — not needed for cluster
- **GPIO break-out:** Full GPIO bank → use only for BMC control
- **PoE:** Entire PoE sheet → replaced by ATX power
- **TV_OUT:** Composite video → obsolete

## Key Design Insight

The upstream CM4 Ethernet pins (TRD0-3) are the **transformer-side** differential pairs,
NOT RGMII signals. The CM4 has an integrated Ethernet PHY (BCM54210PE) that outputs
these directly to magnetics/RJ45.

For the RTL8367S switch, we need the **RGMII MAC interface** from the CM4, NOT the
transformer-side pairs. The CM4 does NOT expose RGMII — it only exposes the PHY
output. This means:

**CRITICAL:** The RTL8367S cannot connect directly to the CM4's Ethernet pins.
We have two options:
1. Use the CM4's built-in PHY output (TRD0-3) through magnetics to RTL8367S PHY ports
   (switch acts as 5-port PHY switch) — SIMPLE but uses 5 PHY ports
2. Use a separate Ethernet PHY per node and connect via RGMII to the switch — COMPLEX

**Option 1 is correct for CM4:** The RTL8367S has 5 integrated PHYs. Connect each
CM4's transformer-level outputs through magnetics directly to switch PHY ports.
This is exactly how the Turing Pi 2 does it.

For **RK3588S (Orange Pi CM5):** The RK3588S can output RGMII directly (no integrated
PHY on most configurations). Need to verify Orange Pi CM5 pinout compatibility.
