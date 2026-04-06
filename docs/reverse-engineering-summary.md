# CocktailAudio X40 Reverse Engineering & Troubleshooting Summary

## System Overview

**Hardware**: CocktailAudio X40 Audio Player  
**CPU**: Sigma Designs SMP8671 (MIPS 24Kc, 702MHz)  
**Issue**: Display blank on boot, audio subsystem failures  
**Root Cause**: Hardware communication failures between boards during boot sequence

---

## Multi-Board Architecture

### Board Inventory
| Board | Part Number | Function | Key Components |
|-------|-------------|----------|----------------|
| CPU | Main-REV1.0 | Main processor | SMP8671, DDR3 RAM (256MB), NAND flash |
| PSU | AF12NT-85 Rev 3A | Power supply | OB2358AP PWM controller, flyback transformer |
| UX | X30-FRONT-LCD REV1.1 | Display/UI | TI TFP401PZP, Intersil TW8835 |
| Amp/Power | | Audio processing | XMOS 8U6C5 audio processor |
| IO | | Front panel I/O | USB, headphone jack |

### Inter-Board Communication
- **Primary**: SPI (Serial Peripheral Interface) from SMP8670 spec sheet
- **Secondary**: UART (3 ports: ttyS0, ttyS1, ttyS2)
- **Data Transport**: Ribbon cables with specific pin assignments
- **Key Signal Path**: TP9 (CPU) ↔ TP13 (amp board) via ribbon cable pin 1

---

## Power Supply Board (AF12NT-85) Component Map

### ICs and Major Components
| Ref Des | Component | Value/Part | Function |
|---------|-----------|------------|----------|
| IC1 | PWM Controller | OB2358AP | Main switching controller |
| IC10 | Voltage Reference | TL431A | Feedback regulation |
| T1 | Transformer | AFI1NR-09B | Flyback isolation |
| PC1 | Optocoupler | EL817 | Isolation feedback |
| Q10 | Transistor | 2N2222A | Switching control |
| RY1 | Relay | HE-1A-5SH | Power switching |

### Resistor Values (Complete Map)
| Ref Des | Colors | Value | Function |
|---------|--------|-------|----------|
| R1 | Gold-Yellow-Green-Maroon | 4.7MΩ | HV startup |
| R2 | Gold-Yellow-Green-Maroon | 4.7MΩ | HV startup |
| R3 | Blue-Orange-Brown | 630Ω | Control |
| R4 | Gold-Black-Purple-Yellow | 1MΩ | Startup |
| R5 | Gold-Black-Black-Brown | 100Ω | Gate drive |
| R6 | Orange-Orange-Orange-Gold | 33kΩ | Feedback |
| R7 | Brown-Black-Black-Gold | 100Ω | Gate drive |
| R8 | Gold-Orange-Black-Brown | 300Ω | Control |
| R9 | RWIN 1.5ΩJ | 1.5Ω 5W | Current sense |
| R10 | Gold-Brown-Orange-Orange | 130kΩ | Feedback |
| R11 | Gold-Red-Red-Red | 222kΩ | Feedback |
| R12 | Gold-Red-Black-Maroon | 200Ω | Control |
| R13 | Maroon-Maroon-Black-Black-Maroon | 550Ω | Control (5-band) |
| R14 | Gold-Yellow-Purple-Yellow | 4.7MΩ | HV startup |
| R15 | Maroon-Maroon-Black-Black-Maroon | 550Ω | Control (5-band) |
| R18 | Gold-Red-Purple-Yellow | 2.7MΩ | HV control |
| R19 | Gold-Orange-Black-Brown | 300Ω | Control |

### Connectors
- **CN1**: AC input (110VAC)
- **CN3**: DC output (multiple rails)
- **CN10**: Control signals (P-ON, GND, +5V, GND)

---

## Test Points & Debug Interfaces

### T-Series (UART/SPI Communication)
| Point | Location | Voltage Activity | Likely Function |
|-------|----------|------------------|----------------|
| T4 | CPU board, near SMP8671 | 0-400mV patterns | SPI SCK/MOSI |
| T10 | CPU board, near CPU base | Active patterns | SPI MISO/CS |
| T11 | CPU board, with switch pad | 0-400mV patterns | SPI data line |
| T12 | CPU board | 0-400mV patterns | SPI control |

### TP-Series (Test/Monitoring Points)
| Point | Location | Connection | Function |
|-------|----------|------------|----------|
| TP1 | UX board | TI TFP401PZP pin 97 | Video processor |
| TP2 | Amp board | ZF79→R11→heatsink chip | Power management |
| TP3 | Audio board | TI audio processor pin 12 | Audio subsystem |
| TP4 | UX board | Intersil TW8835 pin 64 | Video scaler |
| TP9 | CPU board | Ribbon cable pin 1 | Inter-board comm |
| TP11 | Amp board | Ribbon cable pin 10 | Secondary control |
| TP12 | Amp board | Ground reference | System ground |
| TP13 | Amp board | Ribbon cable pin 1 | Links to TP9 |

### Missing Points (Not Yet Located)
**T-series**: T1, T2, T3, T5, T6, T7, T8, T9  
**TP-series**: TP14, TP15

---

## Boot Sequence Analysis

### Normal Boot Failure Pattern
1. **CPU board starts** ✓
2. **Basic hardware detection** ✓
3. **em8xxx driver loads** but with memory errors
4. **XMOS audio device** connects then disconnects
5. **S99sigma script fails** during microcode loading
6. **Display goes blank** after splash screen
7. **Audio/streaming functions fail**

### Memory Error Addresses
**Persistent Failure Points**:
- `0x9edfada8` - Emergency exit #0
- `0x9edb8d9c` - Emergency exit #1  
- `0x9ec93d90` - Emergency exit #2
- `0x00190005/6` - Missing device IDs

### Incremental Boot Testing Results
| Configuration | Result | em8xxx Errors |
|---------------|--------|---------------|
| CPU only | ❌ No power | N/A |
| CPU + IO + UX (no ribbon) | ✅ Clean boot | Minimal |
| CPU + IO + UX (with ribbon) | ✅ System boots | TBD |
| Full system | ❌ Hardware failures | Severe |

**Conclusion**: Problem isolates to **amp/DAC board connection**

---

## Firmware & Software

### System Details
- **Kernel**: Linux 2.6.29.6-37-sigma
- **Architecture**: MIPS 24Kc
- **Memory**: 384MB configured, 256MB physical
- **Storage**: 512MB NAND flash, UBIFS filesystem
- **Network**: 192.168.1.45 (accessible via telnet)

### Critical Scripts
- **S99sigma**: Main initialization script (loads audio/video drivers)
- **euterfe**: Main UI application
- **OB2358AP**: PWM controller for power supply

### Firmware Update System
- **Web interface**: `/mnt/hdd1/.http/htdocs/cocktailAudioUpdate/`
- **PHP scripts**: SourceGuardian encoded (not readable)
- **Recovery mode**: Menu button + power for 7-8 seconds
- **Available firmware**: `firmware.pkg` ready for update

---

## Hardware Issues Identified

### Primary Problem
**Inter-board communication failure** between CPU and amp/DAC boards causing:
- SPI signal integrity issues on ribbon cable
- XMOS audio processor instability  
- em8xxx hardware abstraction layer memory errors
- Cascade failure of audio/video subsystems

### Secondary Issues
- **Ribbon cable connection quality**
- **Power sequencing between boards**
- **Signal timing during hardware initialization**

### Working Subsystems
- ✅ **CPU board and basic system**
- ✅ **Network connectivity**
- ✅ **File system access**
- ✅ **Display subsystem** (when manually initialized)
- ✅ **Power supply** (all rails stable)

---

## Development Kit Recommendations

### High-Density Debug Connector
- **Micro D-sub panel mount** (40+ pins)
- **Removable breakout modules** instead of permanent cables
- **Different modules**: Test points, logic analyzer, scope probes
- **Rear panel integration** for clean appearance

### Test Point Access
- **Single-pin headers** for critical T/TP points
- **Color-coded documentation** for signal types
- **CAT6 cable** for multi-conductor runs (twisted pairs, good shielding)
- **Panel-mount D-sub connectors** for familiar, polarized connections

### Systematic Component Inventory
- **Complete resistor mapping** (all values confirmed)
- **IC identification** across all boards
- **Connector pinout documentation**
- **Power rail mapping**

---

## Next Steps

1. **Complete incremental testing** (add boards one by one)
2. **Firmware update** via recovery mode with firmware.pkg
3. **Ribbon cable replacement/repair** if signal integrity confirmed bad
4. **SPI signal monitoring** on T-points during boot
5. **Component-level diagnosis** of amp/DAC board if needed

---

## Tools & Techniques Used

- **Multimeter testing** of test points and power rails
- **Telnet access** for live system analysis  
- **UART debugging** via T-point monitoring
- **Systematic component identification** via visual inspection
- **Incremental hardware isolation** testing
- **dmesg log analysis** for hardware error correlation
- **Color band resistor decoding** with verification
- **PCB trace analysis** for signal path mapping

**Key Learning**: Modern embedded audio systems use complex multi-board architectures with SPI communication. Hardware issues often manifest as software errors in hardware abstraction layers. Systematic isolation testing is essential for multi-board failure diagnosis.