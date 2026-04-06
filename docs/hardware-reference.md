# X40 Hardware Reference — Extracted from Firmware R0076

*Everything below was reverse-engineered from the extracted firmware filesystem. This is the Rosetta Stone for the project.*

---

## Platform Identity

The X40 is NOT MediaTek — it's built on the **Sigma Designs SMP8670** ("Tango3"), an IPTV-class media processor.

| Property | Value |
|----------|-------|
| SoC | Sigma Designs SMP8670 |
| Architecture | MIPS32 Release 2 |
| Kernel | Linux 2.6.29.6-37-sigma |
| Filesystem | UBIFS on NAND flash |
| Main App | `euterfe` — 7.7MB Qt/DirectFB application |
| Display Framework | DirectFB (hardware framebuffer, not SPI/I2C) |
| Audio Framework | Custom I2C + GPIO via `CI2C` class; ALSA for PCM routing |
| Bootloader | YAMON |

---

## Audio Chip Inventory

| Chip | I2C Address | Role | Datasheet |
|------|-------------|------|-----------|
| **ESS ES9018K2M** | `0x92` | DAC — Sabre32, up to 32-bit/384kHz, DSD128 | [ESS datasheet] |
| **Cirrus Logic SRC4392** | `0xE0` | Sample Rate Converter — also handles I2S routing, digital I/O | [Cirrus SRC4392 datasheet] |
| **Cirrus Logic CS8422** | `0x28` | Digital Input Receiver (DIR) — S/PDIF, TOSLINK decode | [Cirrus CS8422 datasheet] |
| **Cirrus Logic CS5361** | N/A (GPIO-controlled) | ADC — 24-bit/192kHz, for phono/line-in recording | [Cirrus CS5361 datasheet] |
| **Si4730** | (via tuner GPIO) | FM Tuner | [Silicon Labs Si4730] |
| **XMOS** | (GPIO-controlled) | USB audio / digital routing processor | [XMOS docs] |

---

## GPIO Register Map

These are on the SMP8670's global bus ("gbus"). In the original system, `gbus_write_uint32` and `gbus_read_uint32` utilities access them. In our replacement, these will be controlled via an MCU (RP2040/Pi) over I2C or direct GPIO.

### Power Control

| Reg | Dir | Name | Values |
|-----|-----|------|--------|
| 69 | W | Audio Power | 1=ON, 0=OFF — master power for audio subsystem |
| 56 | W | DAC ES9018 Reset | 1=ON, 0=OFF |
| 57 | W | SRC4392 / CS8422 Reset | 1=ON, 0=OFF |
| 52 | W | XMOS Reset | 1=ON, 0=OFF |
| 54 | W | Tuner Si4730 Reset | 1=ON, 0=OFF |
| 55 | W | ADC CS5361 Reset | 1=ON, 0=OFF |
| 66 | W | 35V Supply | 0=ON (for amp section) |

### Input Source Selection

| Reg | Dir | Name | Values |
|-----|-----|------|--------|
| 62 | W | 8670/XMOS Select | 1=SMP8670 audio path, 0=XMOS audio path |
| 63 | W | Audio Switch 2 | See input matrix below |
| 64 | W | Audio Switch 1 | See input matrix below |
| 58 | W | Sample Rate Select | 1=standard (48/44.1kHz), 0=alternate |
| 60 | W | ADC CS5361 HPF | High-pass filter enable |

**Input Source Matrix (GPIO 63, 64):**

| Input | SW2 (reg 63) | SW1 (reg 64) |
|-------|-------------|-------------|
| Phono In | 0 | 0 |
| Line In | 1 | 1 |
| Tuner | 0 | 1 |
| Front | 1 | 0 |

### Output Control

| Reg | Dir | Name | Values |
|-----|-----|------|--------|
| 27 | W | XLR (Balanced) Out | 1=ON, 0=OFF |
| 29 | W | RCA (Unbalanced) Out | 1=ON, 0=OFF |
| 53 | W | Headphone Out | 1=ON, 0=OFF |
| 68 | W | Coaxial Digital Out | **0=ON, 1=OFF** (inverted!) |
| 70 | W | TOSLINK Digital Out | **0=ON, 1=OFF** (inverted!) |

### Status Registers (Read-Only)

| Reg | Dir | Name | Purpose |
|-----|-----|------|---------|
| 7 | R | SRC4392 Lock | Input signal lock status |
| 8 | R | SRC4392 Interrupt | Interrupt flag |
| 9 | R | Tuner Interrupt | Tuner status |
| 21 | R | Emergency | Emergency/fault signal |
| 28 | R | DAC ES9018 Detect | DAC presence detection |
| 32 | R | ADC CS5361 Overflow | ADC clipping detection |

---

## I2C Command Reference

### Command Format (from .cfg files)

```
G W <register> <value>      # GPIO Write
G R <register>               # GPIO Read
E W <addr> <reg> <value>     # I2C Write
E R <addr> <start> <end> <count> <base>   # I2C Read range
D <microseconds>             # Delay
```

The `euterfe` binary loads these from `/etc/config/<name>.cfg` via the `CI2C` class.

### SRC4392 Register Map (I2C 0xE0)

**Page Select (always do first):**
- `E W E0 7F 00`

**Port A Control (output to DAC):**
- Reg `03` — Port A Control 1:
  - `09` = 24-bit I2S, Master, Source = Port A (ADC passthrough)
  - `19` = 24-bit I2S, Master, Source = Port B (SMP8670 playback)
  - `29` = 24-bit I2S, Master, Source = DIR (digital input passthrough)
- Reg `04` — Port A Control 2:
  - `00` = MCLK/128, Source = MCLK
  - `03` = MCLK/512, Source = MCLK
  - `08` = MCLK/128, Source = RXCKO (lock to incoming digital clock)

**Port B Control (input from SMP8670):**
- Reg `05` — Port B Control 1:
  - `41` = 24-bit I2S, Slave, Source = loopback, Muted
- Reg `06` — Port B Control 2:
  - `00` = MCLK/128
  - `03` = MCLK/512

**Digital Transmitter (DIT — S/PDIF/TOSLINK output):**
- Reg `07` — DIT Control 1:
  - `78` = Source = SRC, MCLK/512
  - `F8` = Source = SRC, MCLK/512 (alternate)
- Reg `08` — DIT Control 2:
  - `00` = TX enabled, unmuted, AES out enabled
  - `05` = TX enabled, muted

**Digital Receiver (DIR — S/PDIF/TOSLINK input):**
- Reg `0D` — DIR Control 1:
  - `00` = Input = RX1 (TOSLINK)
  - `01` = Input = RX2 (Coaxial)
  - `03` = Input = RX4
- Reg `0E` — DIR Control 2:
  - `00` = RXCKO output enabled
  - `01` = RXCKO output disabled
- Reg `0F-11` — PLL Config: `22 00 00` = 24.576 MHz

**Sample Rate Converter:**
- Reg `2D` — SRC Control 1:
  - `00` = Source = Port A, Tracking OFF
  - `01` = Source = Port B, Tracking OFF
  - `02` = Source = DIR, Tracking OFF
  - `40` = Source = Port A, Tracking ON
  - `41` = Source = Port B, Tracking ON
- Reg `2E` — SRC Control 2: `00` = 64 samples, decimation filter
- Reg `2F` — SRC Control 3: `00` = 24-bit output
- Reg `30/31` — L/R attenuation: `00` = no attenuation

**Soft Reset (always last):**
- `E W E0 01 3F`

### ES9018K2M Register Map (I2C 0x92)

- Reg `0F` — Left channel volume (default: `04`)
- Reg `10` — Right channel volume (default: `04`)
- Reg `01` — Soft reset
- Full register range: `00-14` (general), `40-5D` (DSP)

---

## Audio Signal Flow

```
                    ┌──────────────┐
                    │ PHONO PREAMP │
    Turntable ──►   │  (MM stage)  │──►┐
                    └──────────────┘   │
                                       ▼
    Line In ────────────────────►  ANALOG MUX  ◄──── Tuner (Si4730)
    (RCA)                        (GPIO 63/64)  ◄──── Front In
                                       │
                                       ▼
                                 ┌──────────┐
                                 │ CS5361   │
                                 │ ADC      │──► I2S ──►┐
                                 │ 24/192   │           │
                                 └──────────┘           │
                                                        ▼
    TOSLINK In ──► RX1 ──►┐                      ┌──────────┐
                          ├──► DIR ──►           │ SRC4392  │
    Coaxial In ──► RX2 ──►┘          │           │          │──► Port A (I2S Master)
                                     ▼           │  Port A  │         │
                              ┌──────────┐       │  Port B  │         ▼
                              │ CS8422   │       │  DIR     │    ┌──────────┐
                              │ (spare)  │       │  DIT     │    │ ES9018   │
                              └──────────┘       │  SRC     │    │ DAC      │
                                                 └──────────┘    │ Sabre32  │
    SMP8670 I2S ──► Port B ──►──────┘    │           │           └────┬─────┘
    (playback)                           │           │                │
                                   DIT ◄─┘           │                ▼
                                    │                │          ┌──────────┐
                                    ▼                │          │ Output   │
                              TOSLINK Out            │          │ Relays   │
                              Coaxial Out            │          │ (GPIO)   │
                                                     │          └──┬───┬──┘
                              ┌───────────┐          │             │   │
                              │ XMOS      │◄─────────┘             │   │
                              │ (USB DAC  │                        ▼   ▼
                              │  mode)    │                  XLR  RCA  HP
                              └───────────┘
```

---

## Boot / Power-On Sequence

From `POWER_ON.cfg` and `S99sigma` init script:

1. Load kernel modules: `llad.ko`, `em8xxx.ko`
2. Load DSP microcode via `xkc` (audio, video, demux processors)
3. Start `euterfe` Qt application
4. `euterfe` runs POWER_ON.cfg:
   - `G W 69 1` — Audio power on
   - `G W 58 1` — Set sample rate mode
   - `G W 62 0` then `G W 62 1` — Toggle XMOS select (reset)
   - `G W 63 1; G W 64 1` — Set input to Line In (default)
   - `G W 52 0` — XMOS off
   - `G W 57 1` — SRC4392 on
   - Configure SRC4392 via I2C (Port B slave, DIT, DIR, PLL)
   - `E W 92 0F 04; E W 92 10 04` — Set DAC volume

---

## Key Binary: euterfe

| Property | Value |
|----------|-------|
| Path | `/tango3/qtdfb/bin/euterfe` |
| Size | 7.7 MB |
| Format | ELF 32-bit MIPS, dynamically linked |
| Framework | Qt (signals/slots visible in strings) |
| Display | DirectFB |
| Fonts | `/tango3/qtdfb/lib/fonts`, `/mnt/hdd1/Font` |

**Key classes found in strings:**
- `CI2C` / `CI2C.cpp` — I2C communication class
- `CAudioFile` — Audio file handling
- `QwCdRipDlg` / `QThreadCdRip` — CD ripping UI and thread
- `QwVolumePopup` — Volume control popup
- `QwInputMenu` — Input selection menu
- `QwReserveRecordSetDlg` — Recording setup
- `QThreadRecordAlbumArt` — Album art download during recording

**Key functions:**
- `SpPlaybackUpdateVolume` — Spotify volume
- `g_fnSetXMosPlay` — XMOS playback mode switch
- `onCommandVolume(float)` / `onCommandVolume(int)` — Volume handlers
- `slotStopPlayback(bool)` — Playback control
- `GetVolume`, `DesiredVolume`, `CurrentVolume` — Volume state

**Config loading pattern:**
- Loads `.cfg` files from `/etc/config/%1.cfg` where `%1` is the config name (e.g., `POWER_ON`, `A_PHONOIN`, `XMOS`)

---

## Available .cfg Files (Complete List)

| File | Purpose |
|------|---------|
| `POWER_ON.cfg` | Full system initialization |
| `POWER_OFF.cfg` | Shutdown sequence |
| `A_PHONOIN.cfg` | Select phono input, configure ADC + SRC for recording |
| `A_LINEIN.cfg` | Select line input |
| `A_TUNER.cfg` | Select FM tuner input |
| `A_FRONT.cfg` | Select front input |
| `D_COAXIAL.cfg` | Select coaxial S/PDIF digital input |
| `D_TOSLINK.cfg` | Select TOSLINK optical digital input |
| `O_LINEOUT_ON.cfg` | Enable XLR + RCA outputs |
| `O_LINEOUT_OFF.cfg` | Disable XLR + RCA outputs |
| `O_DIGITAL_ON.cfg` | Enable coaxial + TOSLINK outputs |
| `O_DIGITAL_OFF.cfg` | Mute coaxial + TOSLINK outputs |
| `O_HP_ON.cfg` | Enable headphone output |
| `O_HP_OFF.cfg` | Disable headphone output |
| `O_AMP_ON.cfg` | Enable amplifier (actually disables — see note) |
| `O_AMP_OFF.cfg` | Disable amplifier |
| `XMOS.cfg` | Switch to XMOS audio path (USB DAC mode) |
| `8670.cfg` | Switch to SMP8670 audio path (internal playback) |
| `DIR_RESET.cfg` | Reset digital input receiver |
| `R_All.cfg` | Read all GPIO registers (diagnostic) |
