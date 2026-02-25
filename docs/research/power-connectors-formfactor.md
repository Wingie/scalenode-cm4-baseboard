# Power Supply, Connectors, and Form Factor Research

Research document for the ScaleNode CM4 Baseboard covering power supply design,
connector specifications, and Mini-ITX form factor constraints.

---

## 1. TPS54360 Buck Converter

### 1.1 Overview

The TPS54360 is a 60V-input, 3.5A step-down (buck) DC-DC converter from Texas
Instruments with an integrated high-side N-channel MOSFET. It uses fixed-frequency
peak current-mode control, which simplifies compensation and reduces output
capacitance requirements.

- **Input Voltage Range:** 4.5V to 60V (tested from 8.5V to 60V)
- **Maximum Output Current:** 3.5A continuous
- **Switching Frequency:** 100 kHz to 2500 kHz (externally adjustable)
- **Reference Voltage:** 0.8V +/-1%
- **Control Mode:** Fixed-frequency peak current mode
- **Eco-Mode:** Pulse-skip mode at light loads (146uA quiescent)
- **Shutdown Current:** 2uA (EN pulled low)

**Datasheet:** TI document SLVSB87 (Rev. G) --
https://www.ti.com/lit/ds/symlink/tps54360.pdf

### 1.2 Package and Pinout

**Package:** 8-pin HSOP (HSOIC) PowerPAD (DDA package), thermally enhanced with
exposed thermal pad.

```
        +-----------+
  BOOT 1|           |8 SW
   VIN 2|  TPS54360 |7 GND
    EN 3|           |6 COMP
RT/CLK 4|           |5 FB
        +-----+-----+
              |EP|
              +--+
         (Exposed Pad = GND)
```

| Pin | Name   | Function                                                    |
|-----|--------|-------------------------------------------------------------|
| 1   | BOOT   | Bootstrap supply for high-side gate driver                  |
| 2   | VIN    | Input supply voltage (4.5V to 60V)                          |
| 3   | EN     | Enable (internal pull-up; pull below 1.2V to disable)       |
| 4   | RT/CLK | Switching frequency set resistor / external clock input     |
| 5   | FB     | Feedback input (0.8V reference)                             |
| 6   | COMP   | Error amplifier output / compensation network               |
| 7   | GND    | Ground (must connect to exposed pad)                        |
| 8   | SW     | Switch node output (connects to inductor)                   |
| EP  | PAD    | Exposed thermal pad (GND) -- solder to PCB ground plane     |

**Package dimensions:** Approximately 5.0mm x 6.2mm (body) with exposed pad on
the underside.

### 1.3 Reference Circuit: 12V Input to 5V Output

Based on the TPS54360EVM-182 evaluation module (TI document SLVU769B), which is
designed for exactly this use case: 12V nominal input, 5.0V output at 3.5A.

#### 1.3.1 Output Voltage Setting (Feedback Divider)

The output voltage is set by a resistor divider from VOUT to the FB pin:

```
VOUT = VREF x (1 + RHS/RLS)
```

Where VREF = 0.8V.

For **5.0V output**:
- RLS (low-side, FB to GND) = **10.0 kohm** (1% tolerance)
- RHS (high-side, VOUT to FB) = **52.3 kohm** (1% tolerance)
- Calculation: 0.8V x (1 + 52.3k/10k) = 0.8V x 6.23 = 4.98V (approximately 5.0V)

#### 1.3.2 Switching Frequency

Set by a resistor from RT/CLK to GND. For the EVM reference design:
- **Target frequency:** 600 kHz
- **RT resistor:** Approximately 100 kohm (consult datasheet Table or WEBENCH)

The 600 kHz frequency is chosen to stay well below the maximum allowed for the
12V-to-5V duty cycle while providing a good balance of efficiency and component
size.

#### 1.3.3 Bootstrap Capacitor

- **Value:** 0.1uF ceramic
- **Type:** X7R or X5R dielectric
- **Voltage Rating:** 10V minimum (16V recommended)
- **Connection:** Between BOOT (pin 1) and SW (pin 8)

#### 1.3.4 Inductor Selection

For 12V to 5V at 600 kHz with 3.5A output:
- **Recommended value:** 15uH to 22uH
- **Saturation current rating:** Greater than 4.5A (peak inductor current)
- **DC resistance:** As low as practical (target < 50 mohm)
- **Type:** Shielded ferrite power inductor recommended

The inductor value is calculated to achieve approximately 30% peak-to-peak ripple
current at maximum load (a common design target for efficiency and transient
response).

#### 1.3.5 Input Capacitors

- **Minimum effective capacitance:** 3uF (after DC bias derating)
- **Recommended:** 2x 4.7uF or 2x 10uF, 50V or 100V rated, X7R ceramic
- **Additional bulk capacitor:** May be needed depending on source impedance
  (e.g., 47uF to 100uF electrolytic or polymer)

Input capacitors must be placed as close to the VIN and GND pins as possible.
This is the most critical placement in the layout.

#### 1.3.6 Output Capacitors

- **Recommended:** 2x 47uF or 3x 22uF, 10V rated, X5R/X7R ceramic
- **ESR requirement:** Low ESR for ripple and transient performance
- **Additional bulk:** 100uF to 220uF polymer or electrolytic may be added
  for improved transient response

#### 1.3.7 Catch (Freewheeling) Diode

The TPS54360 is an asynchronous converter and requires an external Schottky diode:
- **EVM part:** B560C-13-F (60V, 5A Schottky, SMC package)
- **Forward voltage:** 0.70V typical
- **Reverse voltage rating:** Must exceed VIN(max), so 60V minimum
- **Peak current rating:** Must exceed maximum inductor current (>4.5A)

#### 1.3.8 Compensation Network (Type II)

Connected between COMP (pin 6) and GND:
- Series R-C from COMP to GND sets the crossover frequency and phase margin
- Additional small capacitor from COMP to GND for high-frequency rolloff
- Error amplifier gm: 350 uA/V
- Power stage gm: 12 A/V

Typical values for 12V-to-5V at 600 kHz (consult WEBENCH or datasheet Section
8.2 for exact calculations):
- RCOMP: 10 kohm to 30 kohm
- CCOMP (series): 1nF to 10nF
- CHF (parallel rolloff): 22pF to 100pF

#### 1.3.9 UVLO / Enable

The EN pin has an internal pull-up current source. For always-on operation with
12V input, the EN pin can be left floating (connects through internal pull-up).

For programmable UVLO with hysteresis (recommended):
- Resistor divider from VIN to EN to GND
- Typical: RENA (top) = 1 Mohm, RENB (bottom) = 130 kohm
- This sets UVLO turn-on at approximately 8.5V with hysteresis

### 1.4 Architecture for 8A Total (4x CM4 at 2A Each)

The TPS54360 is rated for 3.5A maximum. To supply 8A total for four CM4 modules,
there are several approaches:

#### Option A: Separate Converter Per CM4 (Recommended)

Use **four independent TPS54360** converters, one per CM4 slot (each delivering
up to 2A with 1.5A headroom). This provides:
- Independent power domains per CM4 (fault isolation)
- Individual enable/disable control per slot
- Best thermal distribution across the PCB
- No current-sharing complexity
- Each converter runs at approximately 57% of its rated capacity

#### Option B: Parallel Converters (2x TPS54360)

Two TPS54360 converters paralleled on a shared 5V rail:
- Requires droop current sharing or active current sharing
- Use the RT/CLK pin to synchronize clocks with 180-degree phase interleaving
- Adds complexity; refer to TI Application Note SLVA389 (TPS54620 parallel
  operation) for methodology applicable to TPS54360
- Total capacity: 7A (with derating, three converters would be safer)

#### Option C: Higher-Current Alternative

Consider the **TPS54560** (60V, 5A) -- two units provide 10A with good margin.
Or use a dedicated multiphase controller for a single high-current rail.

**Recommendation:** Option A (four independent converters) is the cleanest design
for a cluster board where each CM4 should have independent power control.

### 1.5 Thermal Considerations

#### Power Dissipation Estimate (12V to 5V at 2A per converter)

At approximately 90% efficiency:
- Pin = 5V x 2A / 0.90 = 11.1W input
- Ploss = 11.1W - 10W = 1.1W per converter
- Total for 4 converters: 4.4W

Primary loss sources:
- High-side MOSFET (integrated): Conduction + switching losses
- Catch diode: Forward voltage x (1 - D) x Iout = 0.70V x 0.583 x 2A = 0.82W
- Inductor DCR: I^2 x R = 4 x 0.040 = 0.16W
- IC quiescent + gate drive

#### Thermal Management

- **Exposed pad:** Must be soldered to PCB with thermal vias to inner ground
  planes. Use an array of 5-9 small vias (0.3mm drill) under the pad.
- **Via fill:** Plugged/filled vias preferred to prevent solder wicking during
  reflow. JLCPCB provides free POFV on 6-layer boards.
- **Copper area:** Provide generous copper pour on the GND plane around the
  converter for heat spreading.
- **Airflow:** Natural convection is likely sufficient at 1.1W per converter,
  but consider placement away from the CM4 modules (which also generate heat).

#### Efficiency

The TPS54360 achieves approximately:
- **90-93% efficiency** at 12V-to-5V, 1A to 3A load (600 kHz)
- **85-88% efficiency** at light loads (100-500mA) due to Eco-mode
- Efficiency drops at higher switching frequencies and wider Vin-to-Vout ratios

### 1.6 PCB Layout Guidelines

#### Critical Current Loops

The most important layout consideration is minimizing the area of the high-
frequency switching current loops:

1. **Hot loop (highest priority):** VIN cap (+) -> VIN pin -> SW pin -> catch
   diode anode -> catch diode cathode -> VIN cap (-). This loop carries
   high di/dt current at the switching frequency. Keep it as tight as possible.

2. **Output loop:** Inductor -> output cap (+) -> output cap (-) -> catch diode
   cathode. Lower priority but still important for output ripple.

#### Component Placement Priority

1. Input ceramic capacitor -- closest to VIN and GND pins (same layer)
2. Catch diode -- close to SW and GND pins (same layer)
3. Bootstrap capacitor -- close to BOOT and SW pins
4. Inductor -- adjacent to SW pin
5. Output capacitors -- at inductor output, close to load
6. Feedback resistor divider -- close to FB pin, route from output sense point
7. Compensation network -- close to COMP pin

#### Routing Rules

- **Feedback trace:** Route away from the inductor and SW node. Never route
  under the inductor. Use Kelvin sensing from the output capacitor positive
  terminal.
- **SW node copper:** Keep the copper area connecting SW pin to the inductor
  and diode as small as practical. Excess copper acts as an EMI antenna.
- **Ground plane:** Maintain a continuous ground plane under the converter.
  Do not split the ground plane. The exposed pad thermal vias provide the
  primary ground connection.
- **Input/output capacitor ground:** Connect directly to the exposed pad ground
  area, not through long traces.

#### Thermal Via Array

Under the TPS54360 exposed pad:
- Via diameter: 0.3mm hole / 0.6mm pad
- Array: 3x3 (9 vias) or at minimum 2x3 (6 vias)
- Connect to inner ground plane(s) for heat spreading
- Fill with resin and cap with copper (POFV) if available

---

## 2. ATX 24-Pin Power Connector

### 2.1 Connector Specifications

- **Connector type:** Molex Mini-Fit Jr. series
- **Motherboard header:** Molex 44206-0007 (right-angle, through-hole)
- **Mating plug:** Molex 39-01-2240
- **Pin rating:** 6A per contact
- **Wire gauge:** 18 AWG recommended (16 AWG for 300W+)
- **Arrangement:** 2 rows of 12 pins (24 total)

### 2.2 Full 24-Pin Pinout

Viewed from the motherboard header (component side), with the latch/clip on top:

```
Pin side view (looking at motherboard header pins):

 +--------------------------------------------------+
 | 13  14  15  16  17  18  19  20  21  22  23  24   |
 |                                                    |
 | 1    2   3   4   5   6   7   8   9  10  11  12   |
 +------+                                    +-------+
        |          LATCH SIDE                |
        +------------------------------------+
```

| Pin | Signal    | Color  | Pin | Signal    | Color  |
|-----|-----------|--------|-----|-----------|--------|
| 1   | +3.3V     | Orange | 13  | +3.3V     | Orange |
| 2   | +3.3V     | Orange | 14  | -12V      | Blue   |
| 3   | GND       | Black  | 15  | GND       | Black  |
| 4   | +5V       | Red    | 16  | PS_ON#    | Green  |
| 5   | GND       | Black  | 17  | GND       | Black  |
| 6   | +5V       | Red    | 18  | GND       | Black  |
| 7   | GND       | Black  | 19  | GND       | Black  |
| 8   | PWR_OK    | Gray   | 20  | NC (-5V*) | White  |
| 9   | +5VSB     | Purple | 21  | +5V       | Red    |
| 10  | +12V      | Yellow | 22  | +5V       | Red    |
| 11  | +12V      | Yellow | 23  | +5V       | Red    |
| 12  | +3.3V     | Orange | 24  | GND       | Black  |

*Pin 20: -5V was made optional in ATX12V 1.3 (2003) and is typically NC on
modern PSUs.*

### 2.3 Voltage Rail Summary

| Rail   | Pins                          | Count | Typical Max Current |
|--------|-------------------------------|-------|---------------------|
| +3.3V  | 1, 2, 12, 13                  | 4     | 20-24A              |
| +5V    | 4, 6, 21, 22, 23             | 5     | 15-22A              |
| +12V   | 10, 11                        | 2     | 12-20A (per rail)   |
| -12V   | 14                            | 1     | 0.3-0.8A            |
| +5VSB  | 9                             | 1     | 2-3A                |
| GND    | 3, 5, 7, 15, 17, 18, 19, 24  | 8     | (return)            |

### 2.4 Signals Relevant to ScaleNode

For the ScaleNode CM4 baseboard, only the following signals are needed from the
ATX 24-pin connector:

#### +12V (Pins 10, 11)
- Main power input for the TPS54360 buck converters
- Two pins at 6A each = 12A maximum from connector
- At 12V, this provides up to 144W (far more than our ~45W requirement)

#### +5VSB (Pin 9)
- Standby 5V supply, always on when PSU is plugged into mains
- Available even when the PSU main rails are off
- Used to power the BMC (management controller) for always-on management
- Typical capacity: 2A to 3A (10-15W)
- This allows Wake-on-LAN and remote power control

#### PS_ON# (Pin 16)
- **Active-low** signal to turn on the PSU main power rails
- When PS_ON# is pulled LOW (to GND), the PSU turns on all rails
- When PS_ON# is HIGH (floating or pulled to +5VSB), the PSU is in standby
- Internal pull-up in the PSU to +5VSB (typically through ~1kohm)
- The BMC can control this via a MOSFET or open-drain GPIO:

```
  +5VSB ----[1k internal to PSU]---- PS_ON# (Pin 16)
                                        |
                                     [NMOS drain]
                                        |
  BMC GPIO ----[gate]              [NMOS source] ---- GND
```

To turn on PSU: BMC drives GPIO HIGH, NMOS conducts, PS_ON# goes LOW.
To turn off PSU: BMC drives GPIO LOW, NMOS off, PS_ON# floats HIGH.

#### PWR_OK (Pin 8)
- **Power Good** signal from PSU
- Goes HIGH (+5V, TTL level) when all PSU rails are stable and within spec
- Remains HIGH for at least 16ms after AC power loss (hold-up time)
- The BMC should monitor this signal to detect power status
- Can be connected directly to a BMC GPIO input (with appropriate level shifting
  if BMC is 3.3V logic)

#### GND (Multiple Pins)
- Connect sufficient ground pins for return current
- For 12V at 4A (48W), the return current through ground is ~4A
- Use at least 3-4 ground pins from the connector

### 2.5 Unused Pins

The following ATX pins are NOT needed for ScaleNode and can be left unconnected
on the baseboard:
- +3.3V (pins 1, 2, 12, 13) -- not needed; CM4 modules use 5V input
- +5V (pins 4, 6, 21, 22, 23) -- not needed; we generate 5V from 12V via TPS54360
- -12V (pin 14) -- legacy rail, not needed
- Pin 20 (NC/-5V) -- not connected

**Note:** Even unused pins should have the connector footprint pads present for
mechanical stability. Leaving +3.3V and +5V unconnected is safe; the PSU will
still regulate these rails but they will be unloaded (which is within spec for
modern PSUs).

### 2.6 Typical ATX Power Supply Rail Capacities

| PSU Wattage | +12V Capacity | +5V Capacity | +3.3V Capacity | +5VSB |
|-------------|---------------|--------------|----------------|-------|
| 300W        | 216W (18A)    | 60W (12A)    | 52W (16A)      | 15W   |
| 450W        | 396W (33A)    | 75W (15A)    | 66W (20A)      | 15W   |
| 550W        | 480W (40A)    | 100W (20A)   | 82W (25A)      | 15W   |
| 750W        | 660W (55A)    | 100W (20A)   | 82W (25A)      | 15W   |

Modern PSUs deliver most of their power on the +12V rail. Even a modest 300W ATX
PSU provides far more than the ~48W (12V x 4A) needed for four CM4 modules.

**Safety limit:** Per IEC 60950, each individual 12V rail is limited to 240VA
(20A) for safety. Multi-rail PSUs split the 12V into independent rails.

---

## 3. USB-C Connector (BMC Management Port)

### 3.1 Overview

The BMC management port uses a USB Type-C receptacle configured for USB 2.0 only
in device (UFP / Upstream Facing Port) mode. This provides a USB serial console
and management interface accessible from a host computer.

### 3.2 USB-C Receptacle Full Pinout (24 Pins)

```
USB-C Receptacle (viewed from mating face / plug insertion side):

  Row B: B12 B11 B10 B9 B8 B7 B6 B5 B4 B3 B2 B1
         GND RX1+RX1-VBS SB2 D-  D+ CC2 VBS TX2-TX2+GND

  Row A: A1  A2  A3  A4  A5  A6  A7  A8  A9  A10 A11 A12
         GND TX1+TX1-VBS CC1 D+  D-  SB1 VBS RX2-RX2+GND
```

| Pin  | Name  | Function                           |
|------|-------|------------------------------------|
| A1   | GND   | Ground                             |
| A2   | TX1+  | SuperSpeed TX+ (not used)          |
| A3   | TX1-  | SuperSpeed TX- (not used)          |
| A4   | VBUS  | Power                              |
| A5   | CC1   | Configuration Channel 1            |
| A6   | D+    | USB 2.0 Data+                      |
| A7   | D-    | USB 2.0 Data-                      |
| A8   | SBU1  | Sideband Use (not used)            |
| A9   | VBUS  | Power                              |
| A10  | RX2-  | SuperSpeed RX- (not used)          |
| A11  | RX2+  | SuperSpeed RX+ (not used)          |
| A12  | GND   | Ground                             |
| B1   | GND   | Ground                             |
| B2   | TX2+  | SuperSpeed TX+ (not used)          |
| B3   | TX2-  | SuperSpeed TX- (not used)          |
| B4   | VBUS  | Power                              |
| B5   | CC2   | Configuration Channel 2            |
| B6   | D+    | USB 2.0 Data+                      |
| B7   | D-    | USB 2.0 Data-                      |
| B8   | SBU2  | Sideband Use (not used)            |
| B9   | VBUS  | Power                              |
| B10  | RX1-  | SuperSpeed RX- (not used)          |
| B11  | RX1+  | SuperSpeed RX+ (not used)          |
| B12  | GND   | Ground                             |

**Receptacle physical dimensions:** 8.34mm x 2.56mm opening, 6.20mm deep.

### 3.3 USB 2.0 Device Mode Configuration

For a USB 2.0-only device (UFP), the connections are minimal:

#### CC Resistors (CRITICAL)

Each CC pin requires its own independent 5.1 kohm (+/-10%) pull-down resistor
to GND:

```
  CC1 (A5) ----[5.1k]---- GND
  CC2 (B5) ----[5.1k]---- GND
```

**WARNING: Do NOT short CC1 and CC2 together and use a single resistor.** This
was the mistake made on early Raspberry Pi 4 boards. Shorting the CC pins causes
incorrect detection with eMarked cables (which have a 1 kohm Ra on the unused CC
line). The parallel resistance (5.1k || 1k) = 836 ohm, which makes the host see
the wrong resistance and may refuse to supply VBUS power.

These 5.1 kohm resistors identify the port as a UFP (device/sink) to the
connected DFP (host/source). The host detects cable attachment by sensing current
flow through its Rp pull-up into our Rd pull-down on the CC line.

#### USB 2.0 Data Connections

In a USB-C receptacle, D+ and D- each appear on both Row A and Row B. For USB
2.0, these must be shorted together at the receptacle:

```
  D+: A6 and B6 shorted together -> route as single D+ to BMC
  D-: A7 and B7 shorted together -> route as single D- to BMC
```

This allows the connector to work regardless of cable orientation (the defining
feature of USB-C reversibility). In a cable, only one set of D+/D- wires exists.

#### VBUS Connections

All VBUS pins (A4, A9, B4, B9) are shorted together at the receptacle. For a
device that is bus-powered:

```
  A4 + A9 + B4 + B9 -> VBUS (5V from host, up to 500mA default USB 2.0)
```

For a self-powered device (the ScaleNode BMC is powered from +5VSB), VBUS may
still be connected for detection purposes but should not be used as a power
source. Add appropriate ESD protection on VBUS.

#### Ground Connections

All GND pins (A1, A12, B1, B12) shorted together to board ground. Also connect
the connector shell/shield to ground (optionally through an RC network for EMI:
1 Mohm || 4.7nF to chassis ground).

#### Unused Pins

Leave completely unconnected (no copper, no traces):
- TX1+/TX1- (A2, A3)
- TX2+/TX2- (B2, B3)
- RX1+/RX1- (B10, B11)
- RX2+/RX2- (A10, A11)
- SBU1 (A8)
- SBU2 (B8)

### 3.4 Reference Schematic (USB 2.0 Device)

```
                    USB-C Receptacle
                   +----------------+
                   |                |
    VBUS_USB <-----|A4,A9,B4,B9    |
                   |                |
                   | A5 (CC1)       |----[5.1k]---- GND
                   |                |
                   | B5 (CC2)       |----[5.1k]---- GND
                   |                |
    D+ <-----------|A6+B6 (shorted)|
                   |                |
    D- <-----------|A7+B7 (shorted)|
                   |                |
    GND <----------|A1,A12,B1,B12  |
                   |                |
    Shield --------|Shell           |---[1M]---+--- Chassis GND
                   +----------------+   [4.7nF]-+
                                              |
                                             GND

    ESD Protection (recommended):
    VBUS_USB ---[TVS diode]--- GND
    D+ ---[ESD clamp]--- GND       (e.g., TPD2E001 or USBLC6-2SC6)
    D- ---[ESD clamp]--- GND
```

### 3.5 USB 2.0 Series Resistors

The USB 2.0 specification calls for 45 ohm (+/-10%) series impedance on each
data line. Many modern USB transceivers (including those in most SoCs) integrate
these termination resistors on-die. Check the BMC/SoC datasheet:

- If integrated: No external resistors needed on D+/D-
- If not integrated: Add 22 ohm to 27 ohm series resistors near the BMC IC
  on each data line

### 3.6 Current Advertisement via CC

When the device is connected, the host advertises its current capability through
the CC pin voltage (set by its Rp value):

| CC Voltage (approx.) | Host Current Capability |
|----------------------|------------------------|
| 0.41V                | Default USB (500mA/900mA) |
| 0.92V                | 1.5A at 5V              |
| 1.68V                | 3.0A at 5V              |

For a self-powered device like ScaleNode, current advertisement is informational
only -- the BMC does not draw significant power from VBUS.

---

## 4. Mini-ITX Form Factor

### 4.1 Board Dimensions

- **Width:** 170.0mm (6.693")
- **Height:** 170.0mm (6.693")
- **Tolerance:** +/-0.5mm
- **Board thickness:** 1.6mm (standard, matching JLCPCB 6-layer)
- **Corner radius:** Optional, typically 0 (square corners per spec)

Mini-ITX was developed by VIA Technologies in 2001 as the smallest ATX-compatible
form factor. It uses a subset of the ATX mounting holes, maintaining mechanical
compatibility with ATX, Micro-ATX, and Mini-ITX cases.

### 4.2 Mounting Hole Positions

Mini-ITX uses four mounting holes that correspond to ATX hole positions C, F, H,
and J. The coordinate system origin is at the **bottom-right corner** of the board
when viewed from the component side with the rear I/O panel at the top.

Alternative convention: Using the **bottom-left corner** as origin (more common
in EDA tools), with X positive to the right and Y positive upward:

```
  Rear I/O Panel Edge (Top)
  +------+-------------------------------------------+
  |      |                                           |
  | (H)  |                                      (C)  |
  |      |                                           |
  |      |                                           |
  |      |                                           |
  |      |                                           |
  |      |          170mm x 170mm                    |
  |      |                                           |
  |      |                                           |
  |      |                                           |
  |      |                                           |
  | (J)  |                                      (F)  |
  |      |                                           |
  +------+-------------------------------------------+
  Origin (0,0) at bottom-left
```

**Mounting hole coordinates** (from bottom-left origin):

| Hole | X (from left) | Y (from bottom) | ATX ID |
|------|---------------|------------------|--------|
| C    | 159.3mm       | 159.8mm          | C      |
| F    | 159.3mm       | 15.1mm           | F      |
| H    | 10.7mm        | 159.8mm          | H      |
| J    | 10.7mm        | 15.1mm           | J      |

**Mounting hole specifications:**
- Hole diameter: 3.96mm (to accept #6-32 UNC standoffs)
- Keepout zone: 7.62mm diameter around each hole (no copper/components)
- Standoff height: 6.35mm (1/4") minimum between board and tray

### 4.3 Rear I/O Panel

The rear I/O panel aperture is standardized across ATX, Micro-ATX, and Mini-ITX:

| Parameter                              | Value                |
|----------------------------------------|----------------------|
| Aperture width                         | 158.75mm (6.250")    |
| Aperture height                        | 44.45mm (1.750")     |
| Tolerance                              | +/-0.20mm            |
| Keepout zone around aperture           | 2.54mm               |
| Board edge to rear chassis wall        | 2.108mm              |
| Bottom of aperture below board top     | 3.81mm               |
| Corner radius (max)                    | 0.99mm               |

**I/O panel position on the board:**
- The I/O aperture is located along the **top edge** (rear) of the board
- The left edge of the I/O cutout is flush with the left edge of the board
  (with the 2.108mm chassis offset)
- The I/O zone spans approximately the left 158.75mm of the rear edge
- Right side of the board (past the I/O zone) is where the ATX power connector
  conventionally sits

### 4.4 Keep-Out Zones

| Zone                        | Constraint                                     |
|-----------------------------|-------------------------------------------------|
| Mounting holes              | 7.62mm dia. clear zone (no copper/parts)       |
| I/O panel edge              | 2.54mm keepout around I/O aperture area        |
| Board edge (general)        | 1.0mm minimum from board edge to copper         |
| Component height (top)      | Varies by case; typically 25mm to 40mm max     |
| Component height (bottom)   | 6.35mm max (standoff clearance to tray)        |
| Expansion slot area         | Single slot at right side of I/O panel          |

### 4.5 ATX Connector Placement Convention

On standard Mini-ITX boards:
- The **ATX 24-pin connector** is typically placed along the **right edge** of
  the board (when viewed from the component side with I/O at top), oriented
  vertically with the latch facing outward (toward the right edge)
- This positioning allows the ATX power cable to route naturally from a
  standard ATX case power supply bay
- For ScaleNode, this convention should be followed for case compatibility

### 4.6 Board Area Budget

With 170mm x 170mm = 28,900 mm^2 of board area:

| Component                    | Estimated Area | Count | Total    |
|------------------------------|---------------|-------|----------|
| CM4 connector (2x 100-pin)  | ~30 x 50mm   | 4     | 6,000mm^2|
| TPS54360 + passives          | ~15 x 20mm   | 4     | 1,200mm^2|
| ATX 24-pin connector         | ~30 x 12mm   | 1     | 360mm^2  |
| USB-C connector              | ~9 x 7mm     | 1     | 63mm^2   |
| Ethernet (RJ45)              | ~16 x 22mm   | 4     | 1,408mm^2|
| BMC / management IC          | ~10 x 10mm   | 1     | 100mm^2  |
| I/O panel components         | Varies        | -     | ~2,000mm^2|
| **Subtotal**                 |               |       | ~11,131mm^2|
| **Remaining for routing**    |               |       | ~17,769mm^2|

This gives approximately 61% of the board area available for routing, decoupling,
and other ancillary components -- a reasonable density for a 6-layer board.

---

## 5. JLCPCB 6-Layer Stackup

### 5.1 Standard Stackup: JLC06161H-3313

JLCPCB's standard 6-layer impedance-controlled stackup for 1.6mm board thickness
with 1oz outer / 0.5oz inner copper. Material: Nan Ya Plastics NP-155F (FR-4
TG155).

**IMPORTANT: When ordering, select "FR-4 TG155" material, NOT the default
"FR4-Standard TG 135-140" for impedance-controlled boards.**

#### Layer Stack (Top to Bottom)

| Layer  | Material       | Thickness  | Er    | Notes                    |
|--------|----------------|------------|-------|--------------------------|
| L1     | Copper (1oz)   | 0.035mm    | -     | Signal (top)             |
| PP1    | Prepreg 3313x1 | 0.0994mm   | 4.05  | L1-to-L2 dielectric      |
| L2     | Copper (0.5oz) | 0.0152mm   | -     | GND plane                |
| Core1  | Core           | 0.55mm     | 4.6   | L2-to-L3 dielectric      |
| L3     | Copper (0.5oz) | 0.0152mm   | -     | Signal / Power           |
| PP2    | Prepreg 2116x1 | 0.1088mm   | 4.16  | L3-to-L4 dielectric      |
| L4     | Copper (0.5oz) | 0.0152mm   | -     | Signal / Power           |
| Core2  | Core           | 0.55mm     | 4.6   | L4-to-L5 dielectric      |
| L5     | Copper (0.5oz) | 0.0152mm   | -     | GND plane                |
| PP3    | Prepreg 3313x1 | 0.0994mm   | 4.05  | L5-to-L6 dielectric      |
| L6     | Copper (1oz)   | 0.035mm    | -     | Signal (bottom)          |

**Total thickness:** approximately 1.6mm (including solder mask)

#### Dielectric Properties (NP-155F)

| Prepreg/Core | Er (Dk)  | Loss Tangent (Df) | Thickness    |
|--------------|----------|--------------------|--------------|
| 3313x1       | 4.05     | 0.02               | 0.0994mm     |
| 2116x1       | 4.16     | 0.02               | 0.1088mm     |
| Core (H/H)   | 4.6      | 0.02               | 0.55mm       |

### 5.2 Recommended Layer Assignment

```
L1 (Top)    : Signal + Components (microstrip routing)
L2 (Inner)  : GND Plane (solid, unbroken reference)
L3 (Inner)  : Signal / Power distribution
L4 (Inner)  : Signal / Power distribution
L5 (Inner)  : GND Plane (solid, unbroken reference)
L6 (Bottom) : Signal + Components (microstrip routing)
```

This SIG/GND/SIG/SIG/GND/SIG arrangement provides:
- L1 and L6 signals reference adjacent ground planes (L2 and L5)
- L3 and L4 can be used as inner signal layers or split power planes
- Symmetric stackup prevents warping
- Good EMI shielding with ground planes as layers 2 and 5

### 5.3 Impedance Targets and Trace Dimensions

All impedance values assume the JLC06161H-3313 stackup. Use the JLCPCB online
impedance calculator (https://jlcpcb.com/impedance) for final verification.
Tolerance: +/-10% standard, +/-5% optional at extra cost.

#### 5.3.1 USB 2.0: 90 ohm Differential

- **Specification:** 90 ohm +/-15% differential impedance
- **Layer:** L1 or L6 (microstrip, referenced to L2 or L5)
- **Dielectric thickness to reference plane:** 0.0994mm (3313 prepreg)

| Parameter          | Target Value  | Notes                           |
|--------------------|---------------|---------------------------------|
| Trace width        | 5.0-6.0 mil   | Adjust per calculator           |
| Trace spacing      | 6.0-8.0 mil   | Gap between D+ and D-           |
| Diff impedance     | 90 ohm        | +/-15% per USB 2.0 spec         |
| SE impedance       | ~45 ohm       | Single-ended target              |
| Length matching     | Within 150mil | Between D+ and D- traces         |
| Max trace length   | ~4 inches     | From transceiver to connector    |

#### 5.3.2 Ethernet: 100 ohm Differential

- **Specification:** 100 ohm differential impedance
- **Layer:** L1 or L6 (microstrip)

| Parameter          | Target Value  | Notes                           |
|--------------------|---------------|---------------------------------|
| Trace width        | 4.5-5.5 mil   | Narrower than USB for higher Z  |
| Trace spacing      | 8.0-10.0 mil  | Wider spacing for higher Z      |
| Diff impedance     | 100 ohm       | Standard Ethernet requirement    |
| Length matching     | Within 50mil  | Between TX+/TX- and RX+/RX-     |
| Max trace length   | 4 inches      | PHY to magnetics/RJ45           |

#### 5.3.3 PCIe: 85 ohm Differential

- **Specification:** 85 ohm differential impedance (PCB traces only)
- **Layer:** L1 or L6 (microstrip, preferred for PCIe)

| Parameter          | Target Value  | Notes                           |
|--------------------|---------------|---------------------------------|
| Trace width        | 5.0-6.5 mil   | Between USB and Ethernet values |
| Trace spacing      | 6.0-8.0 mil   | Moderate coupling                |
| Diff impedance     | 85 ohm        | PCIe CEM specification           |
| SE impedance       | ~42.5 ohm     | Single-ended target              |
| Via transitions    | 3 max         | Per differential pair            |
| Material           | FR4 OK        | For PCIe Gen 1/2 at short length|

**Note on PCIe:** The CM4 uses PCIe Gen 2.0 (5 GT/s). At these speeds and the
short trace lengths on a Mini-ITX board (<3 inches), FR4 material is acceptable.
PCIe Gen 3+ at longer distances would require low-loss laminates.

#### 5.3.4 Single-Ended 50 ohm

For general-purpose controlled impedance (clock signals, etc.):

| Parameter          | Target Value  | Notes                           |
|--------------------|---------------|---------------------------------|
| Trace width        | 5.5-6.5 mil   | On L1/L6 over 3313 prepreg      |
| Impedance          | 50 ohm        | Standard single-ended            |

### 5.4 Via Specifications (JLCPCB 6-Layer)

| Parameter                    | Minimum       | Recommended   |
|------------------------------|---------------|---------------|
| Via hole diameter            | 0.15mm        | 0.20mm        |
| Via pad diameter (outer)     | 0.25mm        | 0.35mm+       |
| Annular ring (standard via)  | 0.075mm       | 0.13mm+       |
| Annular ring (via-in-pad)    | 0.05mm        | 0.075-0.10mm  |
| Drill size increment         | 0.05mm        | -             |
| Aspect ratio (thickness/drill)| -            | 10:1 max      |
| Via-in-pad to PTH clearance  | 0.45mm        | -             |

**Via-in-pad (POFV):** JLCPCB provides free Plated Over Filled Via (POFV) on all
6-layer and above boards. This enables direct via-in-pad for BGA fanout (useful
for the CM4 connectors) without extra cost.

#### Common Via Sizes for This Design

| Application              | Hole    | Pad     | Annular Ring |
|--------------------------|---------|---------|--------------|
| Standard signal via      | 0.20mm  | 0.45mm  | 0.125mm      |
| BGA fanout via-in-pad    | 0.20mm  | 0.35mm  | 0.075mm      |
| Thermal via (under IC)   | 0.30mm  | 0.60mm  | 0.15mm       |
| Power via (high current) | 0.40mm  | 0.70mm  | 0.15mm       |
| Ground stitching via     | 0.20mm  | 0.45mm  | 0.125mm      |

### 5.5 Trace Width for Power (DC Current)

For DC power distribution traces (non-impedance-controlled), use wider traces:

| Current | External (1oz) | Internal (0.5oz) | Notes              |
|---------|----------------|-------------------|--------------------|
| 0.5A    | 8 mil          | 15 mil            | Signal power       |
| 1.0A    | 15 mil         | 30 mil            | Per-device power   |
| 2.0A    | 30 mil         | 60 mil            | CM4 supply         |
| 3.5A    | 50 mil         | 100 mil           | TPS54360 input/out |
| 5.0A+   | Use copper pour | Use plane fill    | Main 12V / 5V bus  |

These assume 10 deg C temperature rise per IPC-2221. For higher current, use
copper pours and multiple vias.

### 5.6 Design Rule Summary for JLCPCB

| Rule                          | Value          |
|-------------------------------|----------------|
| Minimum trace width           | 3.5 mil        |
| Minimum trace spacing         | 3.5 mil        |
| Recommended trace width       | 5.0 mil+       |
| Recommended trace spacing     | 5.0 mil+       |
| Minimum via hole              | 0.15mm         |
| Recommended via hole          | 0.20mm         |
| Solder mask opening           | 0.1mm larger than pad (each side) |
| Board edge to copper          | 0.2mm (0.3mm recommended) |
| Impedance tolerance           | +/-10% standard |

---

## 6. Summary and Design Decisions

### Power Architecture

```
ATX 24-Pin Connector
       |
       +-- +12V (Pins 10, 11) ----+---- TPS54360 #1 --> 5V_CM4_1 (2A max)
       |                           +---- TPS54360 #2 --> 5V_CM4_2 (2A max)
       |                           +---- TPS54360 #3 --> 5V_CM4_3 (2A max)
       |                           +---- TPS54360 #4 --> 5V_CM4_4 (2A max)
       |
       +-- +5VSB (Pin 9) -------------- 5V_BMC (always on, 2A max)
       |
       +-- PS_ON# (Pin 16) ------------ BMC GPIO (open-drain control)
       |
       +-- PWR_OK (Pin 8) ------------- BMC GPIO (input, power status)
       |
       +-- GND (Pins 3,5,7,15,17,18,19,24)
```

### Key Component Selections

| Component          | Part / Value                    | Quantity |
|--------------------|---------------------------------|----------|
| Buck converter     | TPS54360DDA (TI)                | 4        |
| Catch diode        | B560C-13-F (60V/5A Schottky)    | 4        |
| Inductor           | 15-22uH shielded, >4.5A sat    | 4        |
| Input caps (each)  | 2x 10uF/50V X7R ceramic        | 4 sets   |
| Output caps (each) | 3x 22uF/10V X5R ceramic        | 4 sets   |
| Bootstrap cap      | 0.1uF/16V X7R ceramic          | 4        |
| FB divider (high)  | 52.3 kohm 1%                   | 4        |
| FB divider (low)   | 10.0 kohm 1%                   | 4        |
| CC1 pulldown       | 5.1 kohm 10% (USB-C)           | 1        |
| CC2 pulldown       | 5.1 kohm 10% (USB-C)           | 1        |
| ATX connector      | Molex 44206-0007 (24-pin)       | 1        |
| USB-C receptacle   | USB 2.0 compatible              | 1        |

### PCB Specifications

| Parameter              | Value                             |
|------------------------|-----------------------------------|
| Board size             | 170mm x 170mm (Mini-ITX)         |
| Layer count            | 6                                 |
| Stackup                | JLC06161H-3313                    |
| Board thickness        | 1.6mm                             |
| Outer copper           | 1 oz (35um)                       |
| Inner copper           | 0.5 oz (15.2um)                   |
| Material               | FR-4 TG155 (NP-155F)             |
| Impedance control      | Yes (USB 90R, Ethernet 100R, PCIe 85R) |
| Via-in-pad             | POFV (free on 6-layer at JLCPCB) |
| Manufacturer           | JLCPCB                            |

---

## References

- TPS54360 Datasheet (SLVSB87): https://www.ti.com/lit/ds/symlink/tps54360.pdf
- TPS54360EVM-182 User Guide (SLVU769B): https://www.mouser.com/ds/2/405/slvu769b-213927.pdf
- TI Product Page: https://www.ti.com/product/TPS54360
- TPS54620 Parallel Operation (SLVA389): https://www.ti.com/lit/pdf/slva389
- ATX Pinout Guide: https://pinoutguide.com/Power/atx_v2_pinout.shtml
- ATX Connector Details: https://www.smpspowersupply.com/connectors-pinouts.html
- USB Type-C Specification R2.0: https://www.usb.org/sites/default/files/USB%20Type-C%20Spec%20R2.0%20-%20August%202019.pdf
- USB-C Resistors and Emarkers: https://hackaday.com/2023/01/04/all-about-usb-c-resistors-and-emarkers/
- USB-C Example Circuits: https://hackaday.com/2023/08/07/all-about-usb-c-example-circuits/
- Designing with USB-C Lessons Learned: https://dubiouscreations.com/2021/04/06/designing-with-usb-c-lessons-learned/
- USB-C Pinout Guide: https://www.allaboutcircuits.com/technical-articles/introduction-to-usb-type-c-which-pins-power-delivery-data-transfer/
- Mini-ITX Addendum v1.1: https://cdn.instructables.com/ORIG/FJD/TETT/GU59XBKK/FJDTETTGU59XBKK.pdf
- Protocase Motherboard Enclosure Design Guide: https://www.protocase.com/resources/how-to-design-for-motherboards/
- Mini-ITX Wikipedia: https://en.wikipedia.org/wiki/Mini-ITX
- JLCPCB Impedance Calculator: https://jlcpcb.com/impedance
- JLCPCB 6-Layer PCB: https://jlcpcb.com/6-layer-pcb
- JLCPCB 6-Layer Stackup Guide: https://jlcpcb.com/blog/six-layer-pcb-stackup-and-buildup-guidelines
- JLCPCB Capabilities: https://jlcpcb.com/capabilities/pcb-capabilities
- JLCPCB Autogenerated Stackups (GitHub): https://github.com/gsuberland/jlcpcb_autogenerated_stackups
- JLCPCB Free POFV for 6+ Layer: https://jlcpcb.com/blog/Free-Via-in-Pad-on-6-20-Layer-PCBs-with-POFV
- USB Impedance Matching (Cadence): https://resources.pcb.cadence.com/blog/2024-impedance-matching-for-usb-interfaces-in-pcbs
- PCIe Layout Guidelines (Altium): https://resources.altium.com/p/pcie-layout-and-routing-guidelines
- PCIe 85 Ohm Discussion (Samtec/SI Journal): https://www.signalintegrityjournal.com/blogs/7-voice-of-the-experts-signal-integrity/post/1989-pci-express-is-85-ohms-really-needed
- Ethernet PCB Layout (Ka-Ro): https://karo-electronics.github.io/docs/hardware-documentation/txguide/PinAssignments/EthernetSignals.html
- ATX12V Power Supply Design Guide v1.3 (Intel): https://community.intel.com/cipcp26785/attachments/cipcp26785/intel-optane-ssd/2893/1/developer_specs_ATX12V_1_3dg.pdf
