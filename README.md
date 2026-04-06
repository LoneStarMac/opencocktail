# OpenCocktail

**An open-source revival kit for the CocktailAudio X40.**

The CocktailAudio X40 is a beautiful piece of hi-fi hardware — audiophile DAC, phono preamp, CD ripper, network streamer — all in one chassis. But its brain (a Sigma Designs SMP8670 MIPS processor) is long dead, and CocktailAudio has moved on.

This project replaces the dead processor board with a Raspberry Pi Pico 2 (RP2350) for hardware control and any Linux box (Pi 5, NUC, old laptop) for network/storage. All 10 original audio PCBs stay in place. No soldering on the analog path. The audio stays pure.

## Architecture

```
┌─────────────────────────────────────────┐
│  Host (*nix box)                        │
│  ┌──────────┐  ┌──────────┐  ┌───────┐ │
│  │ Web UI   │  │ Ripping  │  │ MPD / │ │
│  │ (Flask)  │  │ Pipeline │  │ Stream│ │
│  └────┬─────┘  └────┬─────┘  └───┬───┘ │
│       └──────┬───────┘            │     │
│         USB Serial            USB Audio │
│              │                    │     │
└──────────────┼────────────────────┼─────┘
               │                    │
┌──────────────┴──┐    ┌───────────┴─────┐
│  Pico 2 (RP2350)│    │  XMOS USB Audio │
│  I2C, GPIO, PIO │    │  Module (exists) │
│  Encoders, LCD  │    │                  │
└────────┬────────┘    └──────────────────┘
         │
    ┌────┴────────────────────────┐
    │  Original X40 Audio Boards  │
    │  ES9018K2M · SRC4392        │
    │  CS8422 · CS5361 · Relays   │
    └─────────────────────────────┘
```

**Pico 2** handles all hardware control: I2C to the DAC/SRC/DIR chips, GPIO for input mux/output relays/power, PIO for rotary encoders. Presents as a USB composite device (HID + CDC serial).

**Host** handles networking, storage, CD/vinyl ripping, music playback, and the web UI. Talks to the Pico over USB serial. Receives audio from the XMOS module over USB.

**Audio never touches the computer.** It flows entirely on the original boards.

## Project Status

🔬 **Phase 1: Reverse Engineering** — in progress

We're mapping the original SMP8670 firmware to understand exactly how it controlled the hardware. Key chips, I2C addresses, and GPIO registers are documented in [`docs/hardware-reference.md`](docs/hardware-reference.md).

## What's Here

```
docs/               Hardware docs, RE findings, project spec
firmware/pico/      Pico 2 firmware (C, TinyUSB) — TBD
host/daemon/        Python host daemon — TBD
assets/             Fonts, reference photos
.claude/skills/     AI agent skill definitions (Paperclip/Claude)
```

## What You Need

- A CocktailAudio X40 (working audio boards, dead processor is fine)
- A Raspberry Pi Pico 2 (RP2350)
- Any Linux box (Pi 5, Intel NUC, old laptop)
- A Bus Pirate or similar for initial hardware probing
- Patience and a multimeter

## Contributing

This is early days. If you own an X40 and want to help probe hardware, test firmware, or write docs, open an issue.

## License

MIT
