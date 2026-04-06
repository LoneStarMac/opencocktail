# OpenCocktail — Project Context

## What This Is

An open-source revival kit for the CocktailAudio X40 audiophile music server. We're replacing the dead Sigma Designs SMP8670 (MIPS32) processor board with a Pico 2 (RP2350) for hardware control and any *nix box for network/storage/UI.

## Architecture Rules

- **Audio never touches the host computer.** All audio flows on the original X40 boards. The XMOS module is already a USB audio device.
- **One Pico 2 controls everything.** I2C bus, GPIO outputs, rotary encoders (PIO), buttons, LCD. USB composite device (HID + CDC serial).
- **Host is architecture-agnostic.** Python daemon talks to Pico over USB serial. Runs on Pi, NUC, Mac, anything with USB and a network.
- **The goal is a downloadable kit.** Flash the Pico, clip wires to a header, install a package. Any X40 owner can do it.

## Key Hardware

| Chip | Bus | Address | Role |
|------|-----|---------|------|
| ES9018K2M | I2C | 0x92 | DAC |
| SRC4392 | I2C | 0xE0 | Sample rate converter |
| CS8422 | I2C | 0x28 | Digital input receiver |
| CS5361 | GPIO | — | ADC (phono/line) |

## Key Files

- `docs/hardware-reference.md` — Complete GPIO register map, I2C commands, signal flow
- `docs/x40-brain-transplant-spec.md` — Master project specification
- `docs/reverse-engineering-summary.md` — Firmware RE findings
- `.claude/skills/` — Agent personalities and instructions for Paperclip

## Firmware Location (Private)

The proprietary CocktailAudio firmware files live in a separate private repo (`opencocktail-firmware`). The Ghidra project (`x40.rep`) is local-only due to size (11GB).

## Current Phase

Phase 1: Reverse Engineering. Mapping the SMP8670 firmware to understand hardware control sequences. The extracted `.cfg` command files document most I2C/GPIO operations. The `euterfe` binary (7.7MB Qt/DirectFB MIPS32 ELF) needs deeper Ghidra analysis for the remaining unknowns.
