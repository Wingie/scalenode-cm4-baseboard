# Research: KiCad 8 Hierarchical Design, CERN-OHL-P v2, and Reference Cluster Boards

**Project:** Scalenode CM4 Baseboard (4-Slot CM4 Cluster Board)
**Date:** 2026-02-25
**Purpose:** Technical research to inform the design of an open-source 4-slot CM4 cluster board using KiCad 8 with hierarchical multi-instance sheets, licensed under CERN-OHL-P v2.

---

## Table of Contents

1. [KiCad 8 Hierarchical Multi-Instance Sheets](#1-kicad-8-hierarchical-multi-instance-sheets)
2. [KiCad 8 Project Structure](#2-kicad-8-project-structure)
3. [CERN-OHL-P v2 License](#3-cern-ohl-p-v2-license)
4. [Reference Cluster Board Designs](#4-reference-cluster-board-designs)
5. [Recommendations for This Project](#5-recommendations-for-this-project)

---

## 1. KiCad 8 Hierarchical Multi-Instance Sheets

### 1.1 Core Concepts

KiCad supports three types of multi-sheet schematic organization:

| Type | Description | Use Case |
|------|-------------|----------|
| **Flat hierarchy** | Sheets are not explicitly connected in a master diagram | Simple multi-page schematics |
| **Simple hierarchy** | Each sub-sheet is used only once | Typical modular designs |
| **Complex hierarchy** | Some sub-sheets are used multiple times (multi-instance) | Repeated subcircuits like CM4 slots |

For a 4-slot CM4 cluster board, the **complex hierarchy** is the correct approach. A single sub-sheet defines the per-slot circuitry (CM4 connector, power regulation, NVMe slot, Ethernet PHY), and that sheet is instantiated four times on the root sheet.

### 1.2 Creating a Sub-Sheet and Instantiating It Multiple Times

**Step-by-step workflow:**

1. **Create the sub-sheet:** In the root schematic, use the Hierarchical Sheet tool (shortcut: `S`). Draw a rectangle and name it (e.g., `CM4_Slot_1`). Assign a filename (e.g., `cm4-slot.kicad_sch`).

2. **Design the sub-sheet:** Open the sub-sheet and design the per-slot circuitry. Add hierarchical labels for all signals that need to connect to the parent sheet (e.g., `ETH_TXP`, `ETH_TXN`, `PCIE_TX`, `PCIE_RX`, `I2C_SDA`, `I2C_SCL`, `SLOT_PWR_EN`, `SLOT_3V3`, `SLOT_5V`).

3. **Create additional instances:** Back on the root sheet, duplicate the hierarchical sheet symbol (Ctrl+D or Ctrl+C/Ctrl+V). Each copy must have:
   - A **unique sheet name** (e.g., `CM4_Slot_1`, `CM4_Slot_2`, `CM4_Slot_3`, `CM4_Slot_4`)
   - The **same filename** (e.g., `cm4-slot.kicad_sch`)

4. **Connect differently per instance:** Wire different nets to the hierarchical sheet pins on each instance in the parent sheet.

**Key rule:** The sheet name must be unique because it forms part of the full net path. A net labeled `DATA` inside the sub-sheet becomes `/CM4_Slot_1/DATA` in slot 1 and `/CM4_Slot_2/DATA` in slot 2 -- they are electrically separate.

### 1.3 Hierarchical Pins/Labels vs Global Labels

KiCad provides three label types with distinct scoping rules:

| Label Type | Scope | When to Use |
|------------|-------|-------------|
| **Net Label** (local) | Single sheet only | Internal connections within a sub-sheet |
| **Hierarchical Label** | Connects child sheet to parent via sheet pins | Signals that differ per slot instance |
| **Global Label** | Every sheet in the entire design | Shared buses: power rails, I2C bus, SPI bus, reset signals |

**For a 4-slot CM4 cluster board:**

- **Use hierarchical labels** for signals unique to each slot: PCIe lanes, per-slot Ethernet pairs, per-slot USB, per-slot NVMe, slot-specific power enable, slot-specific status LEDs.
- **Use global labels** for signals shared across all slots: main power rails (`+5V`, `+3V3`, `GND`), shared I2C management bus, shared SPI bus to BMC, global reset.
- **Use local net labels** for internal connections within the sub-sheet that do not exit (e.g., internal decoupling networks, local voltage dividers).

**How hierarchical pin connections work:**

Inside the sub-sheet, you place a hierarchical label (e.g., `ETH_PAIR_A+`). On the parent sheet, this appears as a pin on the sheet symbol. You wire a net to that pin. For Slot 1, you might wire `ETH1_A+`; for Slot 2, `ETH2_A+`. The sub-sheet design is identical, but each instance connects to different parent-level nets.

### 1.4 Passing Different Net Names to Each Instance

This is the core mechanism for making each CM4 slot unique while sharing one sub-sheet design:

```
Root Sheet
+--------------------------------------------------+
|                                                    |
|  +--[CM4_Slot_1]--+    +--[CM4_Slot_2]--+        |
|  | cm4-slot.kicad  |    | cm4-slot.kicad  |        |
|  |                 |    |                 |        |
|  | ETH_TXP >--ETH1_TXP | ETH_TXP >--ETH2_TXP    |
|  | ETH_TXN >--ETH1_TXN | ETH_TXN >--ETH2_TXN    |
|  | PCIE_TX >--PCIE1_TX  | PCIE_TX >--PCIE2_TX    |
|  | PCIE_RX >--PCIE1_RX  | PCIE_RX >--PCIE2_RX    |
|  | SLOT_ID >--"00"       | SLOT_ID >--"01"         |
|  +-----------------+    +-----------------+        |
|                                                    |
|  +--[CM4_Slot_3]--+    +--[CM4_Slot_4]--+        |
|  | cm4-slot.kicad  |    | cm4-slot.kicad  |        |
|  |                 |    |                 |        |
|  | ETH_TXP >--ETH3_TXP | ETH_TXP >--ETH4_TXP    |
|  | ETH_TXN >--ETH3_TXN | ETH_TXN >--ETH4_TXN    |
|  | PCIE_TX >--PCIE3_TX  | PCIE_TX >--PCIE4_TX    |
|  | PCIE_RX >--PCIE3_RX  | PCIE_RX >--PCIE4_RX    |
|  | SLOT_ID >--"10"       | SLOT_ID >--"11"         |
|  +-----------------+    +-----------------+        |
+--------------------------------------------------+
```

**Internal nets are automatically separated:** Any local net label inside the sub-sheet (e.g., `VREG_OUT`) becomes `/CM4_Slot_1/VREG_OUT`, `/CM4_Slot_2/VREG_OUT`, etc. No action is needed to separate these -- KiCad does it automatically via the sheet path.

**Shared signals use global labels:** Power rails like `+5V`, `+3V3`, `GND`, or a shared I2C bus that the BMC uses to talk to all slots, should be global labels inside the sub-sheet. They will be the same net across all instances.

### 1.5 Best Practices for a 4-Slot CM4 Cluster Board

1. **Design the sub-sheet for the most complex slot first.** If one slot has extra features (e.g., HDMI output, debug UART), include them all in the sub-sheet and use DNP (Do Not Place) markers or configuration resistors for slots that do not use them.

2. **Use buses for grouped signals.** Group related hierarchical labels into buses where possible (e.g., `ETH{TXP,TXN,RXP,RXN}`) to reduce clutter on the root sheet.

3. **Prefix naming convention.** On the root sheet, adopt a consistent prefix: `SLOT1_`, `SLOT2_`, `SLOT3_`, `SLOT4_` for all slot-specific nets.

4. **Separate power per slot.** Even if all slots share the same voltage rails, use per-slot enable signals (`SLOT1_PWR_EN`, etc.) and per-slot current limiting so the BMC can power-cycle individual nodes.

5. **Keep the Ethernet switch and BMC on the root sheet** (or in their own non-replicated sub-sheets). These are singleton subsystems, not per-slot.

6. **Annotation strategy:** Use the "Annotate by sheet number x100" or "x1000" option. This makes reference designators predictable:
   - Slot 1: U101, R101, C101...
   - Slot 2: U201, R201, C201...
   - Slot 3: U301, R301, C301...
   - Slot 4: U401, R401, C401...
   This greatly simplifies debugging and assembly.

7. **PCB layout replication:** Use the **HierarchicalPcb plugin** (KiCad 8 compatible) or the **ReplicateLayout plugin** to lay out one slot and automatically replicate the layout to the other three. This saves enormous time and ensures consistent routing.

### 1.6 BOM and Netlist Handling with Multi-Instance Sheets

- **Each instance generates separate components.** Even though the sub-sheet is shared, KiCad treats each instance independently for BOM and netlist purposes. Slot 1's U101 and Slot 2's U201 are separate BOM entries with the same part but unique reference designators.

- **BOM grouping:** KiCad 8's built-in BOM exporter can group identical parts, so four instances of the same LDO regulator (U101, U201, U301, U401) show as qty 4 of one part number.

- **Netlist correctness:** The netlist correctly handles both the shared nets (global labels) and the per-instance nets (hierarchical labels and local labels scoped by sheet path).

- **KiCad 8 built-in BOM exporter:** New in KiCad 8, this replaces the old external-script approach. It allows selecting and reordering columns and controlling formatting directly from the symbol fields table dialog.

### 1.7 KiCad 8 Specific Features vs KiCad 7

| Feature | KiCad 7 | KiCad 8 |
|---------|---------|---------|
| **Net navigator** | Not available | Shows net path across hierarchical sheets |
| **Properties panel (schematic)** | Not available | Fast editing of selected item properties |
| **Search panel (schematic)** | Not available | Quick search across large schematics |
| **Power symbol net naming** | Pin name (required symbol editing) | Value field (editable in schematic) |
| **Built-in BOM exporter** | External scripts required | Built-in with column selection and formatting |
| **Graphical comparison** | Not available | Compare symbols/footprints between libraries and embedded copies |
| **Drag footprints with tracks** | Limited (single footprint) | Multiple footprints at once |
| **Length tuning patterns** | Basic | Full objects (selectable, modifiable, removable) |
| **Import from other EDA** | Limited | EasyEDA, CADSTAR, Solidworks PCB, Altium, EAGLE |
| **HierarchicalPcb plugin** | Not compatible | Requires KiCad 8+ |
| **Git integration** | Not available | Built-in project manager Git support |

---

## 2. KiCad 8 Project Structure

### 2.1 Recommended File Organization

```
scalenode-cm4-baseboard/
|-- LICENSE                          # CERN-OHL-P v2 license text
|-- README.md                        # Project overview
|-- CHANGES.md                       # Modification log (required by CERN-OHL)
|-- .gitignore                       # KiCad-specific ignores
|-- .gitattributes                   # Filter rules for KiCad files
|
|-- scalenode-cm4-baseboard.kicad_pro  # Project file
|-- scalenode-cm4-baseboard.kicad_sch  # Root schematic (top-level)
|-- scalenode-cm4-baseboard.kicad_pcb  # PCB layout
|-- sym-lib-table                      # Symbol library table (project-local)
|-- fp-lib-table                       # Footprint library table (project-local)
|
|-- cm4-slot.kicad_sch               # Hierarchical sub-sheet: per-slot circuitry
|-- power-supply.kicad_sch           # Sub-sheet: main power supply
|-- ethernet-switch.kicad_sch        # Sub-sheet: Ethernet switch + PHYs
|-- bmc.kicad_sch                    # Sub-sheet: Board Management Controller
|-- pcie-routing.kicad_sch           # Sub-sheet: PCIe lane routing/switching
|
|-- lib/                             # Project-specific libraries
|   |-- scalenode.kicad_sym          # Custom symbols
|   |-- scalenode-footprints/        # Custom footprints (directory-based library)
|   |   |-- CM4_Connector.kicad_mod
|   |   |-- RTL8370N.kicad_mod
|   |   |-- ...
|   |-- 3d-models/                   # 3D models for components
|       |-- CM4_Module.step
|       |-- ...
|
|-- doc/                             # Generated documentation
|   |-- scalenode-cm4-baseboard.pdf  # Exported schematic PDF
|   |-- bom/                         # Generated BOM files
|   |-- ibom/                        # Interactive BOM output
|
|-- production/                      # Manufacturing output
|   |-- gerbers/                     # Gerber files for fabrication
|   |-- drill/                       # Drill files
|   |-- assembly/                    # Pick-and-place, assembly drawings
|   |-- bom-production.csv           # Production BOM
|
|-- docs/                            # Design documentation and research
|   |-- research/                    # Research notes (this file)
|   |-- block-diagrams/              # System block diagrams
|   |-- design-decisions/            # ADRs (Architecture Decision Records)
|
|-- img/                             # Images for README
|-- assets/                          # Visual assets for hardware portals
```

### 2.2 Symbol and Footprint Library Organization

**Project-local libraries (recommended for open-source projects):**

- Store all custom symbols in a single `.kicad_sym` file within `lib/`.
- Store all custom footprints in a directory-based footprint library within `lib/`.
- Reference libraries using `${KIPRJMOD}` variable in `sym-lib-table` and `fp-lib-table` for portability.

**Example sym-lib-table:**
```
(sym_lib_table
  (lib (name "scalenode")(type "KiCad")(uri "${KIPRJMOD}/lib/scalenode.kicad_sym")(options "")(descr "Project-specific symbols"))
)
```

**Example fp-lib-table:**
```
(fp_lib_table
  (lib (name "scalenode-footprints")(type "KiCad")(uri "${KIPRJMOD}/lib/scalenode-footprints/")(options "")(descr "Project-specific footprints"))
)
```

**Benefits of project-local libraries:**
- Anyone cloning the repo gets all required libraries automatically.
- No dependency on system-installed KiCad libraries for custom parts.
- Standard KiCad library parts are referenced from the KiCad installation (they ship with KiCad and are not project-specific).

**When to use git submodules for libraries:**
- If you maintain a shared component library across multiple projects.
- Link to a specific commit so library changes do not unexpectedly break the project.

### 2.3 Version Control Best Practices with KiCad Files

**Essential .gitignore for KiCad:**

```gitignore
# KiCad backup and temporary files
*.bak
*.bck
*.kicad_pcb-bak
*.kicad_sch-bak
*-backups/
*.kicad_prl
*~
_autosave-*
*.tmp
*-save.pro
*-save.kicad_pcb
fp-info-cache

# Generated/exported files (regenerate from source)
*.net
*.dsn
*.ses
```

**Recommended .gitattributes:**

```gitattributes
# Treat KiCad files as binary for merge purposes (prevent broken merges)
*.kicad_pcb binary
*.kicad_sch binary
*.kicad_sym binary

# Or use diff drivers for human-readable diffs (optional, advanced)
# *.kicad_sch diff=kicad_sch
# *.kicad_pcb diff=kicad_pcb
```

**Key practices:**

1. **Commit logically related changes together.** Do not mix schematic and unrelated PCB changes in one commit.

2. **Tag important milestones:**
   - `schematic-review-v1` -- schematic complete for review
   - `gerbers-v1.0` -- first fabrication release
   - `bom-v1.0` -- BOM finalized for ordering

3. **Branch strategy:**
   - `main` -- stable, reviewed designs only
   - `dev` -- active development
   - Feature branches for experimental changes (e.g., `feature/add-poe-support`)

4. **Avoid simultaneous editing of the same .kicad_sch or .kicad_pcb file.** KiCad files are structured text but merging is impractical. Use file locking (Git LFS lock or convention) if collaborating.

5. **KiCad 8 built-in Git support:** The KiCad Project Manager now integrates with Git directly, showing file status icons and branch names. It can commit, push, pull, and switch branches from within the GUI.

6. **Do not track the `.kicad_prl` file.** It contains per-user local preferences (last zoom level, open panels) and will cause constant merge conflicts.

7. **Export schematic PDFs and interactive BOMs for review.** Commit them to `doc/` so reviewers without KiCad can inspect the design.

---

## 3. CERN-OHL-P v2 License

### 3.1 Overview and Full License Location

The CERN Open Hardware Licence Version 2 -- Permissive (CERN-OHL-P v2) was published in March 2020 by CERN. It is one of three variants in the CERN OHL v2 family, all approved by the Open Source Initiative (OSI).

**Full license text available at:**
- CERN official: https://cern-ohl.web.cern.ch/
- Plain text: https://gitlab.com/ohwr/project/cernohl/-/wikis/uploads/3eff4154d05e7a0459f3ddbf0674cae4/cern_ohl_p_v2.txt
- OSI: https://opensource.org/license/cern-ohl-p
- SPDX identifier: `CERN-OHL-P-2.0`
- Choose a License: https://choosealicense.com/licenses/cern-ohl-p-2.0/

**How-to guide (PDF):**
- https://www.openhardware.io/dl/604a1182274106482b41bc4c/design/cern_ohl_p_v2_howto.pdf

### 3.2 What "Permissive" Means (vs CERN-OHL-S and CERN-OHL-W)

The three CERN OHL v2 variants mirror the spectrum of open-source software licenses:

| Variant | Type | Software Analog | Derivative Works Requirement |
|---------|------|-----------------|------------------------------|
| **CERN-OHL-P** | Permissive | Apache 2.0 / MIT | May be distributed under **any** license terms |
| **CERN-OHL-W** | Weakly Reciprocal | LGPL / MPL 2.0 | Modifications to covered source must stay open; larger works may be proprietary |
| **CERN-OHL-S** | Strongly Reciprocal (Copyleft) | GPL v3 | All derivative works must use the same license |

**CERN-OHL-P (Permissive) specifically:**

- **Permissions:** Commercial use, modification, distribution, patent use, private use.
- **Conditions:** Preserve copyright/license notices; document changes with date and description when modifying.
- **Limitations:** No warranty; no liability.
- **Key characteristic:** Derivative works can be made proprietary. Someone can take your open-source CM4 baseboard design, modify it, and sell it without releasing their modifications. They must preserve your original notices and document that changes were made.

**Why choose CERN-OHL-P for this project:**

1. Maximizes adoption -- companies can use and modify the design without fear of copyleft obligations.
2. Analogous to Apache 2.0 (what the original Antmicro Scalenode uses), but purpose-built for hardware.
3. Includes explicit patent grant (Section 6), which Apache 2.0 also provides but MIT does not.
4. Covers physical products ("Making" and "Conveying Products"), which software licenses do not address.

### 3.3 Key License Sections Summary

**Section 1 -- Definitions:**
- "Source" = design materials enabling Product creation (schematics, PCB layouts, firmware, documentation).
- "Covered Source" = Source expressly released under this license.
- "Product" = any device or tangible object resulting from Covered Source.
- "Make" = manufacture, assemble, compile, load, or apply Covered Source.
- "Convey" = communicate to the public or distribute.

**Section 2 -- Applicability:**
- Governs use, copying, modification, Conveying of Covered Source, and Making of Products.
- License is granted directly, worldwide, and without limitation in time.
- Does not restrict fair use rights.

**Section 3 -- Copying, Modifying and Conveying Covered Source:**
- You may copy and Convey verbatim copies, keeping all Notices.
- You may modify Covered Source but must maintain Notices and add a modification Notice with date and summary.
- You may Convey modified Covered Source under **different license terms** (this is what makes it "permissive"), provided you follow the notice requirements and include a copy of the CERN-OHL-P v2.

**Section 4 -- Making and Conveying Products:**
- You may Make and Convey Products, provided recipients can access applicable Notices.

**Section 5 -- Disclaimer and Liability:**
- Covered Source and Products are provided "as is" with no warranties.
- Licensor has no liability for damages.
- You indemnify the Licensor.

**Section 6 -- Patents:**
- Each Licensor grants a perpetual, worldwide, non-exclusive, royalty-free patent license.
- Patent litigation against Covered Source terminates your license rights.

### 3.4 How to Apply CERN-OHL-P v2 to a KiCad Project

**Step 1: Add the LICENSE file**

Place a file named `LICENSE` in the project root containing the full text of the CERN-OHL-P v2 license. The plain text version is available at the GitLab link above.

**Step 2: Add copyright and license headers to source files**

For KiCad schematic files, add the notice in the title block (which is editable in KiCad's schematic editor under File > Page Settings). Recommended text:

```
Title: Scalenode CM4 Baseboard
Copyright: Copyright <Your Name / Organization> <Year>
Comment 1: This source describes Open Hardware and is licensed under the CERN-OHL-P v2.
Comment 2: You may redistribute and modify this source and make products using it
Comment 3: under the terms of the CERN-OHL-P v2 (https://ohwr.org/cern_ohl_p_v2.txt).
```

For KiCad PCB files, you can add the same notice as text on the silkscreen or fabrication layer.

**Step 3: Add notice to the PCB silkscreen**

On the manufactured board, include:
```
(C) <Year> <Your Name>
CERN-OHL-P v2
Source: <URL to repository>
```

This satisfies the requirement in Section 4 that Product recipients can access applicable Notices.

**Step 4: Create a CHANGES file (if modifying an existing design)**

If you are modifying the existing Antmicro Scalenode design, the CERN-OHL-P v2 requires you to document changes. Create a `CHANGES.md` file:

```markdown
# Changes

## [Date] - [Your Name]
- Redesigned for 4-slot CM4 cluster configuration
- Added Ethernet switch (RTL8370N)
- Added Board Management Controller (STM32)
- Changed license from Apache-2.0 to CERN-OHL-P v2
- Reorganized schematic into hierarchical multi-instance sheets
```

**Step 5: Add source location to documentation**

Include the repository URL in the README, in the schematic title block, and on the PCB silkscreen so that anyone who encounters the design or a manufactured product can find the source.

### 3.5 Compatibility with Other Open-Source Licenses

| License Combination | Compatible? | Notes |
|---------------------|-------------|-------|
| CERN-OHL-P + Apache 2.0 (software) | Yes | Both permissive; similar philosophy |
| CERN-OHL-P + MIT (software) | Yes | Both permissive |
| CERN-OHL-P + GPL v3 (software/firmware) | Yes (with care) | CERN-OHL-P for hardware, GPL for firmware; dual-license recommended for firmware |
| CERN-OHL-P + CERN-OHL-S (hardware) | No direct mixing | Cannot re-license CERN-OHL-S portions under CERN-OHL-P |
| CERN-OHL-P + proprietary | Yes | Permissive license allows proprietary derivatives |

**For this project specifically:**
- Hardware design files (schematics, PCB, symbols, footprints): **CERN-OHL-P v2**
- Firmware (BMC firmware, FPGA bitstreams): Consider **Apache 2.0** or **MIT** for maximum compatibility, or **GPL v3** if copyleft is desired for firmware.
- Documentation: Consider **CC-BY-4.0** or keep under **CERN-OHL-P v2** (which covers documentation as part of "Source").

**Note on the existing Antmicro Scalenode license:** The original Scalenode CM4 Baseboard is licensed under Apache 2.0. Since Apache 2.0 is a permissive license, you may create derivative works and re-license them. However, you must:
- Retain the original copyright notice ("Copyright (c) 2020-2023 Antmicro")
- Note that files were changed
- Include a copy of the Apache 2.0 license for the portions derived from Antmicro's work

If creating a substantially new design inspired by but not directly derived from the Scalenode, CERN-OHL-P v2 can be used cleanly for the new work.

---

## 4. Reference Cluster Board Designs

### 4.1 Turing Pi 2

**Overview:**
- Form factor: Mini-ITX (170 x 170 mm)
- Slots: 4x compute module slots (Jetson SO-DIMM pinout; CM4 via adapter)
- Compatible modules: Raspberry Pi CM4, NVIDIA Jetson (Nano, TX2 NX, Orin), Turing RK1

**Networking:**
- Switch IC: **Realtek RTL8370MB-CG+** (8-port Gigabit Ethernet switch)
  - 10/100/1000 Mbps, Full/Half Duplex
  - IEEE 802.1Q VLAN support
  - 4096-entry MAC address table
  - Port-based and tag-based VLAN
  - QoS with 8 priority queues per port
- Ports: 2x RJ45 external (bridged), 4x internal (one per node), 1x BMC (100 Mbps)
- Architecture: L2 managed switch with VLAN support

**PCIe Architecture:**
- No shared PCIe switch (too expensive at $50+ per chip, no suitable multi-host concurrent endpoint solutions found).
- Each slot's single PCIe Gen 2 x1 lane is dedicated to a specific peripheral:
  - Node 1: Mini PCIe slot + SIM tray (for 4G/5G modems)
  - Node 2: Mini PCIe slot
  - Node 3: ASMedia 2-port SATA III controller
  - Node 4: VL805 USB 3.0 controller (2x USB 3.0 + front panel header)
- Turing Pi 2.5 added 4x NVMe slots (one per node, bottom-mounted)

**BMC (Board Management Controller):**
- MCU: **STM32** (specific model not publicly documented)
- Capabilities:
  - Power control per node (hot-plug support)
  - UART console access from all four nodes
  - eMMC image flashing via software
  - Authentication and serial console over LAN
  - OTA firmware updates
  - Self-testing
- Firmware: Open source
- Interface: Web UI, CLI, and API
- Turing Pi 2.5 adds: updated UART converter, 2x onboard storage, FEL recovery button

**Power:**
- Input: ATX 24-pin power connector
- Consumption: ~15W idle, ~25W full load (4x CM4 8GB + SSD)

**Lessons for our design:**
1. The decision to skip a PCIe switch and dedicate each lane to a fixed peripheral is pragmatic but inflexible. Consider whether a PCIe switch (even an inexpensive 2-port one) adds value for NVMe support.
2. The STM32-based BMC providing UART, power control, and flashing is a well-proven approach.
3. Using the Jetson SO-DIMM pinout with CM4 adapters adds module flexibility but increases cost and complexity. Direct CM4 connectors (2x 100-pin Hirose) are simpler for a CM4-only design.
4. The RTL8370N/MB is a good choice for an 8-port GbE switch at reasonable cost.

### 4.2 DeskPi Super6C

**Overview:**
- Form factor: Mini-ITX (standard)
- Slots: 6x CM4 modules
- Target: Homelab, Kubernetes clusters, distributed computing

**Per-Node Features:**
- M.2 2280 slot (PCIe Gen 2 x1) for NVMe SSD
- TF (microSD) card slot
- 5V FAN header
- Micro USB 2.0 port (for eMMC flashing)

**Networking:**
- Switch: 8-port Gigabit Ethernet switch (specific IC not publicly documented)
- 2x RJ45 external uplinks
- Each node gets 1 Gbps internal connection

**Power:**
- Input: ATX 4-pin CPU power OR DC barrel jack (19V-24V)
- Included PSU: 89.87W
- 3x 12V fan headers

**I/O (from Node 1):**
- 2x USB-A ports
- 2x HDMI ports (including HDMI 2.0)
- Note: USB ports disabled by default; require `dtoverlay=dwc2,dr_mode=host` in config.txt

**Boot Options:** eMMC, SD card, netboot, USB, NVMe

**No BMC:** Unlike the Turing Pi 2, the DeskPi Super6C does not have a dedicated BMC. Management is more manual.

**Lessons for our design:**
1. 6 slots in Mini-ITX is ambitious -- power delivery and thermal management become critical.
2. Per-node M.2 NVMe + microSD provides good storage flexibility.
3. The lack of a BMC is a significant limitation for remote/headless operation. A BMC (even a simple one) greatly improves manageability.
4. The Micro USB for eMMC flashing is a simple, low-cost approach but requires physical access.

### 4.3 Antmicro Scalenode (This Project's Ancestor)

**Overview:**
- Form factor: Slim single-node baseboard optimized for 1U rack mount
- Slots: 1x CM4 per board (up to 18 boards per 1U rack)
- License: Apache 2.0 (KiCad source files on GitHub)
- Part of the larger "Scalerunner" open-source compute cluster platform

**Key Features:**
- Gigabit Ethernet with integrated PoE circuitry
- On-board M.2 (key-M) slot for NVMe SSDs
- Slim PCB outline for 1U chassis
- Expansion connector for USB peripherals
- Expansion connector for HDMI adapters
- Designed in KiCad (currently using KiCad 7.x file format)

**Architecture:**
- Each Scalenode is a complete single-node baseboard.
- Multiple Scalenodes connect via Ethernet (external switch).
- No on-board BMC; relies on external infrastructure for management.
- PoE powers each node, eliminating per-node power cables.

**Lessons for our design:**
1. The Scalenode's per-board approach simplifies design but requires an external switch and more cabling.
2. PoE is elegant for rack-mount deployments -- consider it as an option or primary power method.
3. The existing KiCad libraries and footprints (CM4 connector, M.2 slot, PoE circuitry) can be reused.
4. The M.2 NVMe-per-node approach is well-validated.

### 4.4 Other Notable Designs

**Wiretrustee CM4 Quad SATA Board (Open Source):**
- 1x CM4 + 4x SATA ports (NAS-focused, not a cluster)
- Licensed under **CERN-OHL-P v2** -- a direct precedent for our license choice
- Design files include Allegro schematics, Gerbers, 3D models
- Discontinued product with designs fully open-sourced

**Sipeed NanoCluster:**
- Palm-sized cluster board with 7 SOM slots
- Supports CM4, CM5, and Sipeed-specific modules
- Onboard open-source RISC-V Gigabit Ethernet switch
- 60W PD/PoE power support
- Very compact form factor (~$49 for bare board)

**Will Whang's Miniature CM4 Cluster:**
- Custom hobbyist design with detailed documentation
- Uses **RP2040** as management controller (BMC equivalent)
- **RTL8397N** Gigabit Ethernet switch
- **USB2517** USB hub for management
- Good reference for a minimal BMC implementation

**Cerebro Clusterboard (Kickstarter, not yet shipping):**
- 4x SOM slots (Jetson, CM4, CM5, Radxa CM5)
- 3x M.2 sockets per node
- Built-in BMC with KVM support
- Dual Ethernet, 10 Gbps USB 3.2
- Ambitious feature set; no working prototype as of early 2026

### 4.5 Comparative Summary

| Feature | Turing Pi 2 | DeskPi Super6C | Scalenode | Our Target |
|---------|-------------|----------------|-----------|------------|
| **Nodes** | 4 | 6 | 1 (x18 per rack) | 4 |
| **Form Factor** | Mini-ITX | Mini-ITX | Slim 1U | TBD |
| **NVMe per Node** | Yes (v2.5) | Yes (M.2 2280) | Yes (M.2 key-M) | Yes |
| **Ethernet Switch** | RTL8370MB-CG+ | 8-port GbE | External | RTL8370N or similar |
| **BMC** | STM32 | None | External | STM32 or RP2040 |
| **PCIe Switch** | None (dedicated) | None (dedicated) | N/A | TBD |
| **PoE** | No | No (pins present but disabled) | Yes | Preferred |
| **Open Source HW** | No | No | Yes (Apache 2.0) | Yes (CERN-OHL-P v2) |
| **CM4 Interface** | Via adapter (Jetson pinout) | Direct CM4 | Direct CM4 | Direct CM4 |
| **Price** | ~$200+ | ~$200+ | N/A (DIY) | N/A (DIY) |

---

## 5. Recommendations for This Project

### 5.1 Schematic Architecture

```
Root Sheet (scalenode-cm4-baseboard.kicad_sch)
|
|-- [CM4_Slot_1] cm4-slot.kicad_sch  (instance 1 of 4)
|-- [CM4_Slot_2] cm4-slot.kicad_sch  (instance 2 of 4)
|-- [CM4_Slot_3] cm4-slot.kicad_sch  (instance 3 of 4)
|-- [CM4_Slot_4] cm4-slot.kicad_sch  (instance 4 of 4)
|
|-- [Power_Supply] power-supply.kicad_sch    (single instance)
|-- [Ethernet_Switch] ethernet-switch.kicad_sch  (single instance)
|-- [BMC] bmc.kicad_sch                      (single instance)
```

The `cm4-slot.kicad_sch` sub-sheet contains:
- CM4 connector (2x Hirose DF40C-100DS-0.4V)
- Per-slot power regulation (5V and 3.3V LDOs or DC-DC)
- M.2 Key-M connector (PCIe Gen 2 x1 for NVMe)
- Ethernet PHY interface (to switch on parent sheet)
- USB interface
- Status LEDs
- Power enable / reset circuitry
- All slot-specific decoupling and filtering

### 5.2 License Implementation

1. Replace the current Apache-2.0 `LICENSE` file with the CERN-OHL-P v2 full text.
2. Add a `CHANGES.md` file documenting the redesign.
3. Update the README to reference the new license.
4. Add license headers to all KiCad title blocks.
5. Add the license notice and source URL to the PCB silkscreen.
6. Retain the Antmicro copyright notice for any portions derived from the original Scalenode.

### 5.3 Key Component Decisions to Research Further

1. **Ethernet switch IC:** RTL8370N-VB-CG (8-port GbE, well-proven in Turing Pi 2) vs alternatives.
2. **BMC MCU:** STM32 (proven in Turing Pi 2) vs RP2040 (cheaper, community-friendly, used in Will Whang's design).
3. **PCIe architecture:** Dedicated per-slot NVMe (simplest) vs PCIe switch (flexible but expensive).
4. **Power input:** ATX, DC barrel jack, PoE, or combination.
5. **Form factor:** Mini-ITX (standard mounting), custom rack-optimized, or other.

---

## Sources

### KiCad 8 Hierarchical Design
- [KiCad 8.0 Schematic Editor Documentation](https://docs.kicad.org/8.0/en/eeschema/eeschema.html)
- [KiCad 8.0 Introduction](https://docs.kicad.org/8.0/en/introduction/introduction.html)
- [KiCad 8 Version Release Notes](https://www.kicad.org/blog/2024/02/Version-8.0.0-Released/)
- [KiCad 8 New Features Review -- Tech Explorations](https://techexplorations.com/blog/kicad/kicad-8-new-and-updated-features-review/)
- [KiCad 8 vs 7 Discussion -- KiCad Forums](https://forum.kicad.info/t/kicad-7-vs-8/48597)
- [KiCad 8 Overview -- Hackaday](https://hackaday.com/2024/02/26/kicad-8-makes-your-life-better-without-caveats/)
- [KiCad 8 Comparison with KiCad 7 -- SaludPCB](https://saludpcb.com/kicad-8-x-tutorial-comparison-with-kicad-7/)
- [Hierarchical Sheets Tutorial -- Tech Explorations](https://techexplorations.com/guides/kicad/high-speed-pcb-design/create-the-hierarchical-sheets/)
- [KiCad Hierarchical Sheets -- Embedded Computing Design](https://embeddedcomputing.com/technology/analog-and-power/kicad-hierarchical-sheets-for-enhanced-schematics)
- [HierarchicalPcb Plugin (KiCad 8)](https://github.com/gauravmm/HierarchicalPcb)
- [ReplicateLayout Plugin Guide](https://kicad-info.s3.dualstack.us-west-2.amazonaws.com/original/3X/2/5/253a3c06dbd8774b61b2ffc3a5470889e01f0c4f.pdf)
- [Hierarchical Sheets Forum Discussion](https://forum.kicad.info/t/understanding-multi-sheet-schematics/42922)
- [Hierarchical Pin Net Name Discussion](https://forum.kicad.info/t/hierarchical-pin-net-name/57316)

### KiCad Project Structure and Version Control
- [Version Control of KiCad Projects Using Git -- LBL ATLAS Wiki](https://atlaswiki.lbl.gov/guides/kicad/git)
- [Better Manage KiCad Projects Using Git -- Medium](https://medium.com/inventhub/better-manage-kicad-projects-using-git-8d06e1310af8)
- [Using Git with KiCad -- PCBWay](https://www.pcbway.com/blog/PCB_Design_Tutorial/Using_Git_with_KiCad.html)
- [Version Control for KiCad -- CadLab](https://cadlab.io/git-version-control-for/kicad)
- [Version Control for KiCAD PCB Projects -- Michael Kafarowski](https://michael.kafarowski.com/blog/version-control-for-pcb-design/)
- [Library Structure Best Practices -- KiCad Forums](https://forum.kicad.info/t/library-structure-best-practices/22768)
- [KiCAD Project Template -- Stasis Electronics](https://github.com/stasiselectronics/KiCAD-Project-Template)

### CERN-OHL-P v2 License
- [CERN Open Hardware Licence Home](https://cern-ohl.web.cern.ch/)
- [CERN-OHL-P v2 Full Text (plain text)](https://gitlab.com/ohwr/project/cernohl/-/wikis/uploads/3eff4154d05e7a0459f3ddbf0674cae4/cern_ohl_p_v2.txt)
- [CERN-OHL-P v2 -- OSI](https://opensource.org/license/cern-ohl-p)
- [CERN-OHL-P v2 -- SPDX](https://spdx.org/licenses/CERN-OHL-P-2.0.html)
- [CERN-OHL-P v2 -- Choose a License](https://choosealicense.com/licenses/cern-ohl-p-2.0/)
- [Guide to the CERN-OHL-P v2 (PDF)](https://www.openhardware.io/dl/604a1182274106482b41bc4c/design/cern_ohl_p_v2_howto.pdf)
- [CERN Updates Open Hardware Licence](https://home.cern/news/news/knowledge-sharing/cern-updates-its-open-hardware-licence)
- [CERN Open Hardware Licence -- Wikipedia](https://en.wikipedia.org/wiki/CERN_Open_Hardware_Licence)
- [Open Hardware Licenses -- The Turing Way](https://book.the-turing-way.org/reproducible-research/licensing/licensing-hardware/)
- [Open Hardware Makers Curriculum -- OH Licenses](https://curriculum.openhardware.space/articles/06-licenses-and-standards/oh-licenses/)
- [CERN BE-CO-HT Contribution to KiCad](https://ohwr.org/projects/cern-kicad/)

### Reference Cluster Board Designs
- [Turing Pi 2 Specs and I/O Ports](https://docs.turingpi.com/docs/turing-pi2-specs-and-io-ports)
- [Turing Pi 2 Introduction](https://docs.turingpi.com/docs/turing-pi2-intro)
- [Turing Pi 2 Review -- Jeff Geerling](https://www.jeffgeerling.com/blog/2021/turing-pi-2-4-raspberry-pi-nodes-on-mini-itx-board)
- [Turing Pi 2 Review -- Hackaday](https://hackaday.com/2022/06/16/turing-pi-2-the-low-power-cluster/)
- [Turing Pi 2 Review -- CNX Software](https://www.cnx-software.com/2022/05/17/turing-pi-2-mini-itx-cluster-board-supports-rk3588-turing-rk1-raspberry-pi-cm4-and-nvidia-jetson/)
- [Turing Pi 2 Teardown -- blog.mei-home.net](https://blog.mei-home.net/posts/turing-pi-2/)
- [DeskPi Super6C Product Page](https://deskpi.com/products/deskpi-super6c-raspberry-pi-cm4-cluster-mini-itx-board-6-rpi-cm4-supported)
- [DeskPi Super6C Wiki](https://wiki.deskpi.com/super6c/)
- [DeskPi Super6C -- Jeff Geerling PCIe Database](https://pipci.jeffgeerling.com/boards_cm/deskpi-super6c.html)
- [DeskPi Super6C GitHub](https://github.com/DeskPi-Team/super6c)
- [Antmicro Scalenode CM4 Baseboard](https://github.com/antmicro/scalenode-cm4-baseboard)
- [Antmicro Scalerunner Cluster](https://antmicro.com/blog/2022/08/scalerunner-open-source-compute-cluster)
- [Wiretrustee CM4 NAS Open Source -- Hackster](https://www.hackster.io/news/wiretrustee-cancels-its-four-port-raspberry-pi-cm4-nas-plans-releases-all-designs-as-open-source-1039865d65e8)
- [Sipeed NanoCluster -- CNX Software](https://www.cnx-software.com/2025/08/05/sipeed-nanocluster-palm-sized-cluster-board-takes-up-to-7-system-on-modules/)
- [Will Whang Miniature CM4 Cluster](https://www.willwhang.dev/Miniature-CM4-Cluster/)
- [Cerebro Clusterboard -- CNX Software](https://www.cnx-software.com/2025/05/01/cerebro-clusterboard-supports-up-to-four-nvidia-jetson-raspberry-pi-cm4-cm5-or-radxa-cm5-modules/)
- [Realtek RTL8370N-VB-CG Product Page](https://www.realtek.com/en/products/communications-network-ics/item/rtl8370n-vb-cg)
- [CM4 Carrier Board Database -- Jeff Geerling](https://pipci.jeffgeerling.com/boards_cm)
