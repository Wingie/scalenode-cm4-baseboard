# Agentosaurus Edge Node

A 4-node CM4/CM5 cluster carrier board in Mini-ITX form factor with integrated Ethernet switching and baseboard management controller (BMC).

## Key Features

- **4x CM4/CM5 slots** — DF40 high-density connectors, each with M.2 Key-M NVMe and microSD
- **RTL8367S 5-port GbE switch** — PHY-to-PHY through magnetics, RJ45 uplink
- **ESP32-S3 BMC** — USB-C management, per-slot power control, serial console mux, eMMC flashing via USB mux
- **ATX power input** — 24-pin ATX, TPS54360 12V→5V buck, TPS22918 per-slot load switches
- **Mini-ITX form factor** — 170mm x 170mm, standard mounting holes

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

## Schematic Structure

The KiCad project is in `agentosaurus/` with hierarchical sheets:

| Sheet | Description |
|-------|-------------|
| `agentosaurus.kicad_sch` | Top-level sheet with hierarchy |
| `cm4-slot.kicad_sch` | CM4 DF40 connector, M.2 NVMe slot, microSD slot |
| `ethernet-switch.kicad_sch` | RTL8367S 5-port GbE switch, magnetics, RJ45 |
| `power.kicad_sch` | ATX input, TPS54360 buck, LDOs, load switches |
| `bmc.kicad_sch` | ESP32-S3 BMC, USB-C, UART/USB mux |
| `mechanical.kicad_sch` | Mounting holes, board outline |

Custom symbols and footprints are in `agentosaurus/agentosaurus-symbols/` and `agentosaurus/agentosaurus-footprints/`.

## Project Structure

```
├── agentosaurus/          # Agentosaurus KiCad project
│   ├── *.kicad_sch        # Schematic sheets
│   ├── agentosaurus-symbols/    # Custom KiCad symbols
│   └── agentosaurus-footprints/ # Custom KiCad footprints
├── docs/                  # Research and architecture docs
├── lib/                   # Original Scalenode KiCad libraries
├── img/                   # Images
└── LICENSE
```

## Heritage

Forked from the [Antmicro Scalenode CM4 Baseboard](https://github.com/antmicro/scalenode-cm4-baseboard), a single-node CM4 carrier designed for 1U rack PoE deployments. The original design is preserved in the root KiCad project files; Agentosaurus lives in `agentosaurus/`.

## License

The original Antmicro Scalenode design is published under [Apache-2.0](LICENSE). New Agentosaurus hardware designs are published under [CERN-OHL-P v2](https://ohwr.org/cern_ohl_p_v2.txt).
