# Getränkespender — Automated Drink Dispenser

A **Raspberry Pi-controlled automated drink dispenser** with a custom 4-layer PCB, designed in Altium Designer.  
Built as part of a school project at HTL St. Pölten.

---

## Photos

> 📸 **Photos coming soon**

| Filename | Description |
|----------|-------------|
| `images/photos/overview.jpg` | Full device overview |
| `images/photos/pcb_assembled.jpg` | Assembled PCB |
| `images/photos/enclosure.jpg` | Complete build in enclosure |
| `images/photos/relay_closeup.jpg` | Close-up of the relay array |
| `images/photos/dispenser_action.jpg` | Dispenser in action |

---

## How the System Works

The Raspberry Pi reads a user input (button, touchscreen, or software command), selects which drink channel to activate, and triggers the corresponding relay for a set duration — opening the pump or valve for that channel.

```
User Input → Raspberry Pi GPIO → BC817 Transistor → Relay → Pump/Valve → Drink dispensed
```

### 1. Raspberry Pi 3 Model B+ (Brain)

The **Raspberry Pi 3B+** runs the control software and drives all 15 relay channels via its GPIO pins. It connects to the PCB through a **40-pin GPIO header (SBH11-PBPC-D20-RA-BK)** — a standard right-angle dual-row connector that matches the Pi's pinout exactly.

### 2. Relay Driver Circuit

Each of the 15 channels follows the same driver topology:

```
GPIO (3.3V) → 1kΩ resistor → BC817 NPN base
                                   │
                              BC817 collector
                                   │
                             Relay coil (SRD-05VDC-SL-C)
                                   │
                                  GND
                              (+ flyback diode 1N4148 across coil)
```

- The **1 kΩ base resistor** limits the GPIO current to a safe ~3 mA
- The **BC817 NPN transistor** acts as a switch — the Pi's 3.3 V GPIO is not enough to drive the relay directly (which needs 5 V / ~70 mA), so the transistor handles the load
- The **1N4148 flyback diode** in parallel with the relay coil suppresses the inductive voltage spike when the relay de-energizes — protects the transistor from breakdown

### 3. Why Relays — not MOSFETs?

MOSFETs would technically work here. We chose **mechanical relays (SRD-05VDC-SL-C) deliberately** for two reasons:

**Reliability:** Relays provide complete galvanic isolation between the low-voltage control side (Raspberry Pi, 3.3 V logic) and the load side (pumps/valves, potentially higher voltages). No risk of back-feeding into the Pi.

**User experience:** The **audible click of the relay is intentional**. When a customer selects a drink, the click gives immediate acoustic confirmation that the system has responded. It makes the interaction feel tangible and trustworthy — customers feel more confident and enjoy the experience more than if the switching happened silently. This is a deliberate UX decision, not a limitation.

### 4. Power Supply — LM2596 Buck Converters

The system is powered through 5 independent **LM2596S-3.3 buck converter** channels (one per group of relay channels). The LM2596 is a step-down switching regulator:

- **Input:** External 12 V DC
- **Output:** Stable 3.3 V / 5 V for the Raspberry Pi and relay coils
- **Why switching (not linear)?** Much higher efficiency (~75–80 % vs. ~25 % for a 7805), so no heatsink needed and less heat generated inside the enclosure

Each LM2596 circuit includes:
- **Schottky diode (1N5822)** — fast rectifier on the switching node
- **Inductor (1 µH)** — energy storage element for the buck topology
- **Capacitors (220 µF, 470 µF, 680 µF)** — input and output filtering

### 5. Schematic Sheets

| File | Contents |
|------|----------|
| `relay_driver.SchDoc` | One relay channel: BC817 driver + flyback diode + relay (×15) |
| `relay_pi_interface.SchDoc` | Raspberry Pi GPIO header, signal routing to relay drivers |
| `buck_converter.SchDoc` | LM2596 step-down supply, 5 independent channels |
| `documentation.SchDoc` | Block diagram and system overview |

---

## Bill of Materials

| Component | Designator | Qty | Description |
|-----------|-----------|-----|-------------|
| SRD-05VDC-SL-C | S1–S17 | 15 | SPDT relay, 5 V coil |
| BC817-40 | Q1–Q15 | 15 | NPN transistor, relay driver |
| 1N4148 / FDLL4148 | D1–D15 | 15 | Flyback diode |
| 1N5822 | D13–D17 | 5 | Schottky diode, buck converter |
| LM2596S-3.3 | U1–U5 | 5 | Buck converter, 3.3 V output |
| Resistor 1 kΩ | R1–R13, R15, R17, … | 20 | Base resistors |
| Resistor 3.1 kΩ | R14, R16 | 2 | Pull-up / divider |
| Resistor 4.7 kΩ | R18, R20, R22 | 3 | Pull-up |
| Capacitor 470 µF | C1, C4, C7, C8, C13 | 5 | Bulk electrolytic |
| Capacitor 220 µF | C2, C5, C9, C10, C14 | 5 | Output filter |
| Capacitor 680 µF | C3, C6, C11, C12, C15 | 5 | Input filter |
| Inductor 1 µH | L1–L5 | 5 | Buck converter choke |
| 2×20 GPIO Header | Raspberry Pi1 | 1 | 40-pin Pi connector |
| 2-pin headers | P1–P15 | 15 | Pump/valve output connectors |

Full BOM available in [`docs/BOM.xlsx`](docs/BOM.xlsx) and [`docs/BOM.pdf`](docs/BOM.pdf).

---

## Project Structure

```
Getraenkespender/
│
├── README.md
├── .gitignore
│
├── schematics/                    # Altium Designer project files
│   ├── Getraenkespender.PrjPcb    # Main Altium project file
│   ├── PCB1.PcbDoc                # PCB layout
│   ├── relay_driver.SchDoc        # Relay + BC817 driver circuit
│   ├── relay_pi_interface.SchDoc  # Raspberry Pi GPIO interface
│   ├── buck_converter.SchDoc      # LM2596 power supply
│   └── documentation.SchDoc      # System overview / block diagram
│
├── gerber/                        # Manufacturing files
│   ├── PCB1.GTL / .GBL            # Top / Bottom copper
│   ├── PCB1.GTO / .GBO            # Top / Bottom silkscreen
│   ├── PCB1.GTS / .GBS            # Top / Bottom solder mask
│   ├── PCB1.GTP / .GBP            # Top / Bottom paste
│   ├── PCB1.GM                    # Mechanical / board outline
│   ├── PCB1.G1 / .G2              # Inner copper layers
│   └── drill.txt                  # NC drill file (Excellon)
│
├── libraries/                     # Altium component libraries
│   ├── RASPBERRY_PI_3_MODEL_B+.*  # Raspberry Pi schematic + footprint
│   ├── SBH11-PBPC-D20-RA-BK.*     # 40-pin GPIO header
│   ├── LM2596S-3.3.*              # Buck converter IC
│   ├── SRD-05VDC-SL-C.IntLib      # Relay
│   ├── BC817-40.IntLib            # NPN transistor
│   ├── FDLL4148.IntLib            # Flyback diode
│   ├── 1N5822.IntLib              # Schottky diode
│   ├── VREG_LM2596S-3.3.PcbLib    # Buck converter footprint
│   └── custom_components.*        # Project-specific custom parts
│
├── docs/
│   ├── schematic_export.pdf       # Full schematic PDF export
│   ├── BOM.pdf                    # Bill of Materials (PDF)
│   ├── BOM.xlsx                   # Bill of Materials (Excel)
│   └── documentation.pdf         # Project documentation
│
└── images/
    ├── pcb/                       # PCB renders (add when available)
    └── photos/                    # Build photos (add when available)
```

---

## Tools Used

| Tool | Purpose |
|------|---------|
| **Altium Designer** | Schematic capture & PCB layout |
| **Raspberry Pi OS** | Control software runtime |
| **Python / GPIO Zero** | Relay control scripting |
| Gerber viewer ([EasyEDA](https://gerber-viewer.easyeda.com)) | View Gerber files without Altium |

---

## Author

**HTL St. Pölten — Electronics & Technical Computer Science**  
Project: Werstadt / Getränkespender · 2025
