## Cocktail Audio X40 — Complete Hardware Inventory & Interconnect Map (Markdown Draft)

### 1. System Topology
- **Main Processor Board** — Sigma Designs SMP8671A-CBE3 SoC, DDR2 RAM, NAND, Ethernet PHY, SATA, USB hubs.
- **Audio/Power Chassis Board** — Analog I/O, DAC, ADC, relays, power regulation, phono preamp.
- **UX / Front-LCD Board** — Intersil TW8805 LCD controller, local MCU, IR receiver, rotary encoder inputs.
- **Breakout Boards** — Front-IO (USB, Aux, headphone, LED, IR), HDD carrier, USB-SATA bridge for optical drive.

### 2. Interconnect Architecture
- **Processor ⇄ Audio/Power Board**
  - Likely I²S/TDM audio, I²C control, GPIOs.
  - Separate analog power rails.
- **Processor ⇄ UX Board**
  - LCD parallel RGB or BT.656 video.
  - I²C for LCD/UX MCU.
  - Rotary encoders via analog pots into UX MCU.
- **Front-IO ⇄ Other Boards**
  - USB to Processor.
  - Aux input to Audio board.
  - Headphone jack to DAC board.

### 3. Per-Board BOMs

#### 3.1 DAC / Analog Out Board
**Role:** ESS ES9018K2M DAC, op-amp stages, balanced/unbalanced outputs.

**ICs:**
- ESS ES9018K2M (topmark: ES9D18K2M GLVB637186 E494)
- NE5532A dual op-amp (N5532A 4BM A8SZG4)
- AMS1117-3.3 LDO regulators (1117-33 0C06)
- Additional SOIC/TSSOP op-amps — TBD

**Relays:**
- Panasonic AGQ2 series signal relays (AGQ2004H) — coil voltage TBD

**Passives:**
- Nichicon gold-sleeve electrolytics — values TBD
- WIMA film caps — values TBD
- Various SMD resistors/capacitors — values TBD

**Connectors:**
- DAC-JST1 (CN1, large) — to Audio/Power board (balanced audio)
- DAC-JST2/3 — to Audio/Power board (channel/sense)
- DAC-JST4 — TBD
- DAC-JST5/6 — TBD

#### 3.2 Processor / Main Board
**Key Components:**
- Sigma Designs SMP8671A-CBE3 Secure Media Processor (PVS045.00)
- Samsung K4T1G084QG-BCF7 DDR2 SDRAM ×2 (1 Gb each)
- Atheros AR8035 Ethernet PHY
- FE1.1s USB hub
- Prolific PL2571 USB-SATA bridge
- Various regulators, NAND flash — TBD

#### 3.3 UX / Front-LCD Board
**Key Components:**
- Intersil TW8805 LCD controller
- Local MCU — TBD
- Rotary encoder inputs (volume/menu)
- IR receiver

#### 3.4 Front-IO Board (CA-X30F-FRONT-IO-REV1.0)
**Features:**
- Power/standby button
- Status LED (green/blue)
- 1/4" headphone jack → DAC board
- USB-A port → Processor board
- Aux input → Audio/Power board
- IR receiver → UX board

### 4. Interconnect Table (Initial)
| From | To | Cable Type | Pins | Function |
|------|----|------------|------|----------|
| PROC-SATA1 | HDD-SATA-DATA | SATA | 7 | HDD data |
| AUDIO/PSU | HDD-PWR | JST | 4 | HDD power (+12V, +5V, GND) |
| PROC-FFC-AUDIO | AUDIO-FFC-PROC | FFC | TBD | Digital audio (I²S/TDM) + I²C + power |
| PROC-FFC-UX | UX-FFC-PROC | FFC | TBD | Video + I²C/UART + power |
| PROC-JST-FUSB | FIO-USB-A | JST | 4 | USB power/data |
| FIO-JST-AUX | AUDIO-JST-AUXIN | JST | 3 | Aux analog L/R/G |
| FIO-JST-HP | DAC-JST? | JST | TBD | Headphone analog |
| UX-JST-ENC1 | POT-VOL-JST | JST | 3 | Volume pot |
| UX-JST-ENC2 | POT-MENU-JST | JST | 3 | Menu pot |
| FE1.1s-PORTx | PL2571-USB | — | — | Internal USB to ODD bridge |

### 5. To-Do
- Photograph all connectors for pin counts & silkscreen IDs.
- Identify all TBD ICs, passives, and relay coil voltages.
- Measure cables for length/gauge.
- Confirm interconnect functions with continuity tests.

