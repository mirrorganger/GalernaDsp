# CLAUDE.md

Guidance for AI agents working in this repository.

## What this is

A KiCad hardware project for **Galerna DSP**, an STM32F405-based audio effects board
(two audio channels in/out, 8 pots through a mux, footswitches/buttons, OLED, microSD).
It is a PCB design repo, not application code — there is a `cubeMx/` firmware skeleton,
but no DSP effect processing has been written yet.

Before making claims about the design, read **[docs/Design.md](docs/Design.md)** — it
is the single source of truth for requirements vs. as-built state, BOM, and circuit
rationale. Do not trust part numbers from memory or from git history; the schematic
files are ground truth if `docs/Design.md` and the `.kicad_sch` files ever disagree.

## Project structure

- `GalernaDsp.kicad_pro` — project file; lists the four schematic sheets.
- `GalernaDsp.kicad_sch` — root schematic (just sheet symbols).
- `power.kicad_sch`, `mcu.kicad_sch`, `audio_codec.kicad_sch`, `input_control.kicad_sch`
  — the actual sheets. KiCad `.kicad_sch` and `.kicad_pcb` files are S-expression text;
  they're greppable, e.g. `grep -o '(property "Value" "[^"]*"' mcu.kicad_sch` to list
  parts on a sheet, or search `lib_id "..."` to find every instance of a part.
- `GalernaDsp.kicad_pcb` — the board, 4-layer (F.Cu / In1.Cu / In2.Cu / B.Cu).
- `libraries/Galerna/` — custom symbols/footprints/3D models used across the project
  (referenced as `Galerna:<name>` in the schematics).
- `cubeMx/` — STM32CubeMX-generated firmware for STM32F405RGT6. `Galerna.ioc` is the
  CubeMX config (peripherals: ADC1, I2C2, I2S2, SDIO, SPI1, USB_OTG_FS). `Core/` is
  generated HAL init code with empty `USER CODE` blocks — no application logic exists
  yet. `build/` is a build directory, don't treat its contents as source.
- `jlcpcb/` — generated manufacturing outputs (gerbers, drill files, BOM/CPL for JLCPCB
  assembly). Treat as build output, not something to hand-edit.
- `docs/Design.md` — the design doc. Keep this updated as the single doc reflecting the
  current implementation — when the schematic changes in a way that affects
  requirements, parts, or circuit rationale, update this file rather than creating a
  new one.

## Conventions to know

- The audio codec is **ES8388**, not the WM8731 that appears in early project history —
  it was swapped during layout. If you find old notes, commit messages, or generated
  content referencing WM8731, treat it as stale.
- A headphone amplifier (TPA6110A2DGN) was added beyond the original "no amplification"
  requirement — there are 3 audio jacks (line in, line out, headphone out), not 2.
- Footprints follow a size convention: 0402 for most passives/resistors, 0603 for
  ~1–10 µF caps, 0805 for 22 µF caps and ferrite beads. Match this when adding parts.
- No analog/digital ground plane split, and no controlled-impedance routing (USB FS is
  the only differential pair and isn't critical) — don't propose adding either without
  being asked.
- `*.bak`, `*.lck`, `*.log`, `*.ini` and `*-backups/` are gitignored; KiCad backup/lock
  files you see locally are not meant to be committed.

## Working in this repo

- This is primarily a hardware (EDA) project — most "changes" happen in KiCad itself,
  not via text edits. You can read and grep the `.kicad_sch`/`.kicad_pcb` S-expression
  files, but avoid hand-editing them unless explicitly asked; KiCad's own save format
  and UUIDs are easy to corrupt by hand.
- Documentation and firmware source (`cubeMx/Core`, `docs/`) are normal text — edit
  those freely.
- If asked to update docs after a schematic change, verify part numbers by grepping the
  actual `.kicad_sch` files (see the grep pattern above) rather than assuming the old
  doc was correct.
