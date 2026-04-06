# CocktailAudio X40 Brain Transplant — Project Specification

**Project:** Replace the X40's dead Sigma Designs SMP8670 MIPS mainboard with a new brain, keeping all original audio boards and hardware UI intact.

**Date:** 2026-04-05
**Status:** Firmware fully extracted and analyzed — we have the complete audio control protocol.

---

## 1. What We're Working With

### The X40 (Patient)

**Major discovery from firmware analysis:** The processor is NOT MediaTek — it's a **Sigma Designs SMP8670** ("Tango3"), an IPTV-class media processor running MIPS32 Release 2 at 700MHz. The main application is `euterfe`, a 7.7MB Qt/DirectFB binary.

See `hardware-reference.md` for the complete chip register maps, I2C addresses, and GPIO control sequences extracted from the firmware.

| Component | Chip | I2C Addr | Protocol (CONFIRMED) | Role |
|-----------|------|----------|---------------------|------|
| DAC | ESS ES9018K2M | `0x92` | I2S audio data, I2C for volume (reg 0F/10) | 32-bit/384kHz DAC, DSD128 |
| Sample Rate Converter | Cirrus Logic SRC4392 | `0xE0` | I2C config, I2S Port A (master) + Port B (slave) | Audio routing hub — all signals pass through this |
| Digital Input Receiver | Cirrus Logic CS8422 | `0x28` | I2C config | S/PDIF & TOSLINK input decode |
| ADC | Cirrus Logic CS5361 | GPIO-only | GPIO enable (reg 55), I2S output to SRC4392 | 24-bit/192kHz ADC for phono/line recording |
| FM Tuner | Silicon Labs Si4730 | GPIO (reg 54) | GPIO reset + I2C control | FM radio |
| XMOS DSP | Unknown model | GPIO (reg 52) | GPIO reset, selected via reg 62 | USB audio path, DSD handling |
| Analog Mux | Hardware switch | GPIO (reg 63, 64) | 2-bit source select | Routes phono/line/tuner/front to ADC |
| Output Relays | Hardware | GPIO (27,29,53,68,70) | Direct GPIO | XLR, RCA, headphone, coax, TOSLINK enable |

### The 10 Boards (Inventory)

| # | Board | Likely Signals | Priority |
|---|-------|---------------|----------|
| 1 | Headphone/Aux/USB/Power Switch | Audio analog, USB pass-through, power logic | Medium |
| 2 | Left Rotary Encoder | Quadrature encoder signals (A/B/SW), possibly USB HID or GPIO | High (UI) |
| 3 | MIPS Processor Board (DEAD) | The hub — everything connects here | **REMOVE** |
| 4 | Audio Processing/Inputs Board | I2S, I2C, analog audio, possibly SPI | **Critical** |
| 5 | Audio Output/DAC Board | I2S from XMOS, I2C config, analog out | **Critical** |
| 6 | Power Board | DC rails, possibly I2C power management | Medium |
| 7 | SATA Board | SATA data + power connector | Easy (direct to Mac) |
| 8 | Display Controller | SPI/I2C/parallel RGB, possibly LVDS | High (UI) |
| 9 | Right Encoder/Nav Buttons | Quadrature + GPIO, possibly USB HID | High (UI) |
| 10 | Phono Preamp | Analog audio, I2C/SPI for ADC config | High (recording) |

### The New Brain (2018 Mac Mini i7)

| Feature | Spec | Relevance |
|---------|------|-----------|
| CPU | i7-8700B, 6 cores / 12 threads | Massive overkill — plenty of headroom |
| USB | 4x USB-A 3.0, 4x USB-C (Thunderbolt 3) | Key interconnect — we'll hang everything off USB |
| HDMI | 1x HDMI 2.0 | Could drive the display if we bypass the display controller |
| Ethernet | 1x Gigabit | Network streaming |
| Storage | Internal + Thunderbolt NVMe | Fast ripping target |
| Audio | 3.5mm headphone jack | Backup/testing only |

---

## 2. Architecture — The Big Picture

```
┌─────────────────────────────────────────────────┐
│                   MAC MINI                       │
│                                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │
│  │ Audio    │  │ Display  │  │ Ripping /    │  │
│  │ Engine   │  │ Server   │  │ Tagging      │  │
│  │ (ALSA/  │  │          │  │ Engine       │  │
│  │ CoreAud)│  │          │  │              │  │
│  └────┬─────┘  └─────┬────┘  └──────┬───────┘  │
│       │              │              │           │
│  USB  │         USB  │         SATA │           │
│       │              │              │           │
└───────┼──────────────┼──────────────┼───────────┘
        │              │              │
   ┌────▼────┐   ┌─────▼─────┐  ┌────▼────┐
   │ RP2040  │   │ RP2040    │  │ Direct  │
   │ Bridge  │   │ Bridge    │  │ SATA    │
   │ #1      │   │ #2        │  │         │
   │ (Audio) │   │ (Display  │  │         │
   │         │   │  + UI)    │  │         │
   └────┬────┘   └─────┬─────┘  └─────────┘
        │              │
   Ribbon cables  Ribbon cables
        │              │
   ┌────▼────┐   ┌─────▼─────┐
   │ Audio   │   │ Display   │
   │ Boards  │   │ + Encoder │
   │ (4,5,10)│   │ Boards    │
   │         │   │ (2,8,9)   │
   └─────────┘   └───────────┘
```

### The Core Idea

The MIPS board was the hub that spoke to every other board over ribbon cables using serial protocols (I2C, SPI, UART, or possibly parallel). We replace it with:

1. **RP2040 (Raspberry Pi Pico) microcontrollers** as protocol translators — they tap into the existing ribbon cables and speak the original protocols to the boards, while presenting themselves to the Mac as USB devices (audio class, HID, serial, etc.)

2. **The Mac Mini** runs the high-level software — audio playback, CD/BD ripping, metadata tagging, display UI, and network streaming.

### Why RP2040s Instead of Direct USB?

The audio boards don't speak USB — they speak I2S, I2C, and SPI. The RP2040 is perfect because:

- **PIO (Programmable I/O):** Can emulate almost any serial protocol in hardware, including I2S at audio rates
- **TinyUSB stack:** Can present as USB Audio Class device, USB HID (encoders), or USB CDC serial
- **Dual core:** One core handles USB, the other handles real-time I2S/protocol timing
- **$4 each:** Dedicate one per subsystem without guilt
- **Proven:** People have built USB audio devices, USB-to-I2S bridges, and USB-to-SPI bridges with these

---

## 3. Project Phases

### Phase 0: Documentation & Inventory (1–2 weekends)

**Goal:** Photograph, label, and catalog everything before touching a single wire.

Tasks:
- Photograph every board, both sides, with good lighting
- Label every ribbon cable connector with masking tape (e.g., "R1: MIPS→Audio Processing")
- Record ribbon cable pin counts for each connector
- Note any visible chip markings (especially MCU part numbers — these tell us what protocols they speak)
- Check for any silk-screen labels on PCBs near connectors (TX, RX, SCL, SDA, MOSI, etc.)
- Trace which ribbon cables connect to which boards (draw a connection map)
- Check power rails with multimeter — what voltages are present on each board (3.3V, 5V, 12V?)

**Deliverable:** A board map document with photos, connection diagram, chip inventory, and voltage notes. We'll build this together as a living doc in this project folder.

### Phase 1: Signal Discovery (2–4 weekends)

**Goal:** Identify the protocol on each ribbon cable using the Bus Pirate.

This is where we learn what the MIPS board was actually saying to each board. We'll work through this systematically:

#### Step 1: Visual Protocol Identification

Before probing, many protocols can be guessed from the ribbon cable pin count and chip datasheets:

| Pin Count | Likely Protocol | Clues |
|-----------|----------------|-------|
| 2 wires + GND | UART (TX/RX) | Idles high (~3.3V) |
| 2 wires + GND | I2C (SDA/SCL) | Both lines have pull-up resistors visible on board |
| 4 wires + GND | SPI (MOSI/MISO/CLK/CS) | One line is clearly a clock |
| 3-4 wires + GND | I2S (BCLK/LRCK/DATA/MCLK) | High frequency clock for audio (e.g., 3.072 MHz) |
| Many wires | Parallel bus | Rare, more likely on the display |

#### Step 2: Bus Pirate Protocol Sniffing

For each ribbon cable, we'll follow this procedure:

**A. Power up safely:** With the MIPS board removed, we need to check if each sub-board can be powered independently. Some boards may get power through the ribbon cable from the MIPS board, others from the power board directly.

**B. Voltage check (multimeter):**
- Measure each pin against ground
- Note which pins are at 0V (ground), 3.3V, 5V, or floating
- This immediately tells you power pins vs. signal pins

**C. Bus Pirate connection:**
```
Bus Pirate  →  Ribbon Cable
GND         →  GND pin (identified above)
MOSI        →  First signal pin (for probing)
CLK         →  Second signal pin (if clock suspected)
```

**D. Try each protocol mode:**
1. **UART first** (most common for debug/config):
   - Bus Pirate: `m 3` (UART mode)
   - Try common baud rates: 9600, 115200, 38400
   - If you see readable ASCII text, you've found a UART debug console

2. **I2C scan** (most common for control):
   - Bus Pirate: `m 4` (I2C mode)
   - Run address scan: `(1)` for 7-bit addresses
   - If devices respond, you've found the I2C bus and know the slave addresses

3. **SPI** (less common for inter-board, more for flash/config):
   - Bus Pirate: `m 5` (SPI mode)
   - Try reading: `[0x00 r:4]`

#### Step 3: I2S Audio Bus Identification

The audio data path is almost certainly I2S. This is harder to sniff with a Bus Pirate because it's a continuous high-speed stream. For this one:

- **Look at the XMOS datasheet** — its I2S pinout is well documented
- **Trace the ribbon cable from the XMOS** to see where BCLK, LRCK, and DATA go
- **Measure with multimeter:** I2S BCLK will show ~1.5V average (50% duty cycle square wave)
- If we need to verify, the Bus Pirate's logic analyzer mode or a cheap Saleae clone (~$15) can capture the actual I2S waveform

#### Bus Pirate Quick Reference (for you, Lach)

```
Connect: 115200 baud, 8N1
Commands:
  m     → select mode (3=UART, 4=I2C, 5=SPI)
  (1)   → I2C address scan
  [     → start condition
  ]     → stop condition
  r     → read byte
  0xFF  → write byte
  W     → power supplies ON
  w     → power supplies OFF

Example I2C read from address 0x48:
  [0x90 r:2]
  (0x90 = 0x48 << 1 with read bit)
```

**Deliverable:** Protocol map for every ribbon cable — what protocol, what addresses, what data.

### Phase 2: RP2040 Bridge Firmware (4–8 weekends)

**Goal:** Write firmware for RP2040 Picos that bridge between the original board protocols and USB.

We'll need approximately 2-4 Pico boards, each handling a subsystem:

#### Bridge 1: Audio I/O Bridge (Most Complex)

**Hardware:**
- RP2040 Pico
- Connects to: Audio Processing board (#4), Audio Output/DAC board (#5), Phono Preamp (#10)

**Firmware responsibilities:**
- Present as **USB Audio Class 2.0 device** to the Mac (input + output)
- Speak **I2S** to the XMOS/ES9018K2M for playback
- Speak **I2C** to configure the DAC registers (volume, sample rate, PCM/DSD mode)
- Speak **I2C/SPI** to the phono preamp's ADC for vinyl recording
- Handle sample rate negotiation between USB and I2S clocks

**Key challenge:** Clock synchronization. USB audio and I2S run on different clocks. The RP2040's PIO can generate I2S clocks, and TinyUSB handles USB audio adaptive rate feedback. We'll likely need an external oscillator for jitter-free audio clocks.

**Software stack:**
```
TinyUSB (USB Audio Class 2.0)
    ↕ ring buffer
PIO State Machine (I2S master)
    ↕
ES9018K2M / XMOS / ADC
    (via I2C for control, I2S for audio data)
```

#### Bridge 2: Display + UI Bridge

**Hardware:**
- RP2040 Pico
- Connects to: Display controller (#8), Left encoder (#2), Right encoder/buttons (#9)

**Firmware responsibilities:**
- Read **quadrature encoder** signals (PIO is perfect for this — hardware debouncing)
- Read **button** states (GPIO with interrupts)
- Present as **USB HID** device (encoders as scroll wheels, buttons as keys)
- **Display:** Two options depending on what we find:
  - **Option A:** If the display is a standard SPI/I2C panel, the RP2040 drives it directly and the Mac sends framebuffer data over USB CDC serial
  - **Option B:** If the display has a standard interface (HDMI/composite), we might bypass the display controller entirely and drive it from the Mac's HDMI with a small HDMI-to-whatever adapter

#### Bridge 3: Power & Misc Bridge (Optional)

**Hardware:**
- RP2040 Pico (or might not be needed)
- Connects to: Power board (#6), Headphone/Aux/USB board (#1)

**Firmware responsibilities:**
- Monitor power state, relay control
- Headphone detect, output relay switching
- Present as USB CDC serial device for simple command/response

### Phase 3: Mac Software Stack (Ongoing, parallel with Phase 2)

**Goal:** Build the software that runs on the Mac Mini and ties everything together.

#### OS Decision: Linux vs. macOS

| Factor | macOS | Linux |
|--------|-------|-------|
| USB Audio Class 2.0 | Native CoreAudio support | Native ALSA support |
| USB HID | Native IOKit/HID | Native evdev/libinput |
| CD/BD Ripping | MakeMKV, XLD | MakeMKV, abcde, cdparanoia |
| FLAC encoding | Native via afconvert or ffmpeg | ffmpeg, flac |
| Metadata tagging | MusicBrainz Picard | MusicBrainz Picard, beets |
| Display control | Framebuffer over serial or dedicated app | Direct framebuffer, fbdev, or small X11/Wayland app |
| USB gadget mode | Not supported | Supported (but not needed here — Mac is host) |
| Ease of headless operation | Possible but finicky | Excellent |
| Remote admin | SSH + Screen Sharing | SSH |

**Recommendation: Linux (Debian or Ubuntu Server).** The Mac doesn't need macOS for anything here, and Linux gives us:
- Better control over ALSA and audio routing
- Lighter weight (no GUI overhead unless we want it)
- Better support for headless operation
- Easier to script and automate everything
- Huge community for USB audio and embedded projects

#### Software Components

1. **Audio Daemon** — Routes audio between USB audio devices (the RP2040 bridges) and the internal pipeline. Options: PipeWire (modern, handles both consumer and pro audio) or bare ALSA.

2. **Ripping Engine** — CD: `cdparanoia` for extraction + `flac` for encoding + MusicBrainz for metadata. BD: `MakeMKV` for 4K disc ripping (your modded firmware drive handles the decryption).

3. **Vinyl Recording Service** — Listens on the USB audio input from the phono preamp bridge, records to FLAC, uses silence detection for automatic track splitting, queries Discogs/MusicBrainz for metadata.

4. **Display Server** — Sends framebuffer data to the RP2040 display bridge over USB serial. Could be a simple Python app using Pillow to render album art, now-playing info, menus.

5. **Input Handler** — Reads USB HID events from the encoder/button bridge, maps them to application actions (volume, track skip, menu navigation).

6. **Network Streamer** — DLNA/UPnP server (e.g., `minidlna`), Roon endpoint, or Shairplay for AirPlay.

7. **Web UI** (stretch goal) — A local web interface for library management, accessible from any device on the network.

### Phase 4: Integration & Polish (2–4 weekends)

- Physical fitting of the Mac Mini inside the X40 chassis (measure carefully!)
- Cable routing and management
- Power sequencing (Mac needs to boot when the X40 power button is pressed)
- Thermal management (the i7-8700B runs warm — verify airflow in the chassis)
- Final audio quality testing (measure noise floor, verify bit-perfect playback)

---

## 4. Hardware Shopping List

| Item | Qty | Purpose | Approx Cost |
|------|-----|---------|-------------|
| Raspberry Pi Pico (RP2040) | 4 | Protocol bridges (2-3 active + 1 spare) | $20 |
| Breadboards + jumper wires | 1 set | Prototyping connections | $15 |
| Ribbon cable breakout boards | 5-6 | Tap into existing ribbon cables without cutting | $20-40 |
| Logic analyzer (Saleae clone) | 1 | Optional but very helpful for I2S/protocol debugging | $15 |
| Audio-grade oscillator (e.g., Si5351) | 1 | Clean I2S master clock generation | $10 |
| USB hub (powered, USB 3.0) | 1 | Connect all bridges to Mac via one port | $25 |
| Custom PCBs (later) | 2-3 | Replace breadboard prototypes with permanent boards | $30-50 |

**Total estimated: ~$120-175** (not counting the Mac Mini and X40 you already have)

---

## 5. Risk Assessment

### High Risk (revised — many risks eliminated by firmware analysis)
- **GPIO register access method.** The GPIO registers (27, 29, 52-70, etc.) are on the SMP8670's internal global bus ("gbus"). When we remove the SMP8670 board, we lose direct access to these GPIOs. We need to determine: are the audio board's chip enables and relay controls wired directly to the SMP8670's pins (gone when we remove the board), or do they go through an I2C GPIO expander on one of the audio boards? **Mitigation:** Physical tracing of the ribbon cables from the SMP8670 board to the audio boards. If GPIOs are direct SMP8670 pins, we use an RP2040 or MCP23017 I2C GPIO expander to replicate them.

- **The "gbus" GPIO abstraction.** The firmware uses `gbus_write_uint32` which writes to memory-mapped registers on the SMP8670. These GPIO "registers" (numbered 7, 21, 27, etc.) are NOT standard Linux GPIO numbers — they're an abstraction layer specific to the Sigma Designs SDK. We need to figure out which physical pins they map to. **Mitigation:** Correlate gbus register numbers to physical connector pins using multimeter continuity testing while the SMP8670 board is still in place (if possible).

### Medium Risk
- **Display panel is proprietary.** If the display uses a custom parallel interface or unusual controller, driving it from an RP2040 may require significant reverse engineering. **Mitigation:** Option B — use a small HDMI display module ($20-30) as a drop-in replacement if the original is too hard to drive.

- **Physical fit.** The 2018 Mac Mini is 197mm × 197mm × 36mm. It needs to physically fit inside the X40 chassis alongside the power supply and audio boards. **Mitigation:** Measure the chassis cavity carefully in Phase 0. Worst case, the Mac Mini sits externally connected via a short cable loom.

- **Clock jitter in USB-to-I2S bridge.** USB audio uses adaptive/asynchronous transfer. If the RP2040's PIO-generated I2S clock has too much jitter, audio quality suffers. **Mitigation:** Use the Si5351 or similar programmable clock generator for the I2S master clock, with the RP2040 managing the adaptive rate feedback loop.

### Low Risk
- **SATA drive connection.** The Mac Mini has Thunderbolt 3 — a cheap SATA-to-USB or Thunderbolt-to-SATA adapter handles this trivially.
- **CD/BD drive.** The modded BD drive likely has a SATA or USB interface already. Should "just work" on Linux with `sr0` device.
- **Network streaming.** Solved problem — many mature Linux solutions exist.

---

## 6. Immediate Next Steps

1. **Open it up and photograph everything.** Every board, both sides. Close-ups of all chip markings. Post the photos and we'll identify every IC together.

2. **Measure and record ribbon cable pinouts.** For each ribbon cable: how many pins, which pins have voltage (and what voltage), which pins are ground.

3. **Look for UART debug ports.** Many embedded devices have unpopulated UART headers on the main board. Even with the MIPS board dead, the sub-boards might have their own UART test points. These are gold for understanding what the MCUs on each board are doing.

4. **Try powering individual boards.** With the MIPS board disconnected, see if the audio boards power up from the power board alone. If they do, we can start probing them immediately.

5. **Order RP2040 Picos and a Saleae clone.** Even before we finish sniffing, having the hardware on hand means we can start prototyping as soon as we identify the first protocol.

---

## 7. How We'll Work Together

I'll help you with:
- **Identifying chips** from photos/markings and pulling up their datasheets
- **Writing Bus Pirate scripts** for each protocol sniffing session
- **Interpreting captured data** (I2C transactions, SPI frames, UART dumps)
- **Writing RP2040 firmware** in C (Pico SDK + TinyUSB) or MicroPython for prototyping
- **Building the Linux software stack** (audio daemon, ripping pipeline, display server)
- **Designing the display UI**
- **Debugging** when things inevitably don't work the first time

Each session, we can pick a specific board or subsystem to focus on. I'll keep track of what we've discovered and update this spec as we go.

---

---

## 8. Reverse Engineering Strategy: Ghidra + ARM64 Comparison

### Ghidra Headless Analysis

We can run Ghidra in headless mode to auto-analyze the MIPS `euterfe` binary and extract a complete function map. This doesn't need a GUI — it runs from the command line and exports structured data.

**Setup (on the Mac Mini or any machine with Java):**
```bash
# Download Ghidra
wget https://github.com/NationalSecurityAgency/ghidra/releases/download/Ghidra_11.3_build/ghidra_11.3_20250108.zip

# Create a project and auto-analyze
ghidra/support/analyzeHeadless /tmp/ghidra_projects X40_Project \
  -import euterfe \
  -processor MIPS:LE:32:default \
  -postScript ExportFunctionList.java /tmp/x40_functions.csv

# Export decompiled C for every function
ghidra/support/analyzeHeadless /tmp/ghidra_projects X40_Project \
  -process euterfe \
  -postScript DecompileAllFunctions.java /tmp/x40_decompiled/
```

**What we'll get:**
- Complete function list with addresses, sizes, and call graphs
- Decompiled C pseudocode for every function (especially `CI2C` methods)
- String cross-references — we can trace exactly which function loads which `.cfg` file
- The `CI2C::write()` and `CI2C::read()` implementations → shows us how the app talks to the gbus/I2C hardware
- The GPIO register mapping from gbus addresses to physical operations

**Ghidra through Chrome (for interactive exploration):**
Yes — we can run Ghidra Server or just Ghidra in X11-forwarded mode, but even simpler: install Ghidra on the Mac, use VNC or screen sharing, and browse it via Chrome remote desktop. Or we use the Ghidra headless exports and build our own web viewer.

### MIPS ↔ ARM64 Firmware Comparison

This is a genuinely clever approach. If CocktailAudio released an ARM64 firmware (likely for a newer model like the X45 Pro or X50 which may use an ARM SoC), we can:

1. **Extract both binaries** and run Ghidra headless on each
2. **Match functions by signature:** Qt class names, string references, and `.cfg` file references survive across architectures. A function that references "A_PHONOIN" and calls `CI2C::write` is the phono input setup function regardless of whether it's MIPS or ARM64.
3. **Build a function map:** Create a spreadsheet mapping MIPS function → ARM64 function → purpose
4. **Write the open replacement:** With the function map in hand, we know exactly what each function needs to do. We rewrite them in Python or C for the Pi/Mac.

**The real win:** We don't even need to fully decompile the binary. The `.cfg` files already give us 80% of the audio control logic in plain text. What Ghidra reveals is the *remaining 20%*: how `euterfe` decides which `.cfg` to load, how it handles the display, how it manages the database, and how it coordinates CD ripping with audio routing.

### What the X30 Firmware Tells Us

The X30 firmware (`X30_R0149.zip`, 105MB) is likely from the X30 which is a different product tier. It might use:
- A different SoC (possibly ARM-based)
- A different audio chip set
- But the same `euterfe` application framework

If the X30 is ARM64, then comparing X40 `euterfe` (MIPS) with X30 `euterfe` (ARM64) gives us the Rosetta Stone for understanding the codebase.

### Revised Target Architecture: Pi vs Mac Mini

With the firmware now understood, the project splits into two viable paths:

**Path A: Mac Mini (your original plan)**
- Pros: Massive power, HDMI for display, plenty of USB ports, runs Linux natively
- Cons: Physical size, power draw, thermal management in the chassis
- Best for: Maximum capability, 4K BD ripping, running Ghidra locally

**Path B: Raspberry Pi 4/5 (your ARM64 idea)**
- Pros: Tiny, low power, native GPIO pins (could potentially drive the audio boards directly!), runs ARM64 Linux
- Cons: Less CPU power, no Thunderbolt, might struggle with 4K BD ripping
- Best for: Clean integration inside the X40 chassis, potentially direct hardware control without RP2040 bridges

**Path C: Hybrid** (potentially best of both)
- Pi 5 inside the X40 chassis for audio control, display, and UI
- Mac Mini external as a NAS/ripping station connected over the network
- Pi handles real-time audio control; Mac handles heavy lifting

---

## 9. Immediate Action Items (Revised)

With the firmware analysis complete, our priorities shift dramatically:

1. **Extract and analyze the X30 firmware** — is it ARM64? If so, run both through Ghidra.

2. **Run Ghidra headless on `euterfe`** — get the function map, find the `CI2C` class implementation, understand how gbus registers map to physical pins.

3. **Physical probing (now targeted):** We know exactly what to look for:
   - Which ribbon cable carries I2C (SDA/SCL) to the SRC4392 (addr 0xE0), ES9018 (addr 0x92), CS8422 (addr 0x28)?
   - Which ribbon cable carries the GPIO enable signals (regs 27, 29, 52-57, 62-64, 68-70)?
   - Is there a single I2C bus shared across all audio boards, or separate buses?
   - Use Bus Pirate to scan for I2C devices: if we find 0xE0, 0x92, and 0x28 on the same bus, that's our main audio I2C bus.

4. **Prototype the simplest thing first:** Connect a Pi or RP2040 to the audio board's I2C bus and try running the POWER_ON sequence manually:
   ```python
   import smbus
   bus = smbus.SMBus(1)
   # SRC4392 page select
   bus.write_byte_data(0x70, 0x7F, 0x00)  # 0xE0 >> 1 = 0x70
   # ... and so on
   ```

5. **Order parts:** At minimum: a Raspberry Pi Pico, jumper wires, and a ribbon cable breakout board for the main audio board connector.

---

*This is a living document. As we discover more about each board's protocol and behavior, we'll update the tables and architecture diagrams accordingly.*
