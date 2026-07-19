# Galerna DSP — Design

Galerna DSP is a DSP effects board based on the STM32F405, intended to process two
audio channels in real time (guitar/line-level effects unit style: pots + footswitches +
OLED status display).

This document describes the board as actually implemented in the KiCad project. Where the
build diverged from the original requirements, that's called out explicitly.

## Requirements (original intent)

- Two audio channels input/output.
- 48 kHz / up to 24-bit sample width.
- 8 potentiometers, read through a multiplexer.
- 2 mechanical switches.
- 2 buttons.
- USB-C powered.
- SWD debug.
- OLED screen.
- IO extension header.

## As-built summary

- Two audio channels in/out over I2S, plus a dedicated headphone output — a headphone
  amplifier was added, which was not in the original "no audio amplification" requirement.
- 8 potentiometers wired through a CD4051 analog mux to a single ADC input.
- 2 mechanical switches + 2 buttons for footswitch-style control.
- USB-C powered, split into a digital 3.3 V buck rail and a clean analog 3.3 V LDO rail.
- SPI OLED status display.
- microSD card slot (SDIO) for storage — not in the original requirements list.
- SWD debug and IO expansion share a single 2×5, 1.27 mm JST-GH header (CN1).
- 4-layer PCB, no analog/digital ground split, no controlled impedance (USB FS is the
  only differential pair, and it isn't critical).

## Schematic sheets

The KiCad project (`GalernaDsp.kicad_sch`) is split into four sub-sheets:

| Sheet | File | Contents |
|---|---|---|
| Power | `power.kicad_sch` | USB-C input, ESD protection, buck + LDO regulators |
| MCU | `mcu.kicad_sch` | STM32F405RGT6, crystal, OLED, microSD, IO/SWD header, reset/power switches |
| Audio | `audio_codec.kicad_sch` | Audio codec, headphone amplifier, audio jacks |
| Input Control | `input_control.kicad_sch` | 8 potentiometers, mux, footswitches, buttons |

## Bill of materials (as built)

| Function | Part | Reference(s) | Notes |
|---|---|---|---|
| Microcontroller | STM32F405RGT6 (LQFP-64) | U (mcu) | Matches original plan |
| Audio codec | **ES8388** | U (audio) | **Changed from the originally planned WM8731SEDS** during layout (see commit "Codec updated to ES8388") |
| Headphone amplifier | TPA6110A2DGN | U (audio) | Added — original requirements said "no audio amplification" |
| Headphone volume pot | RK09L12D0A1W | R16 | Local pot for headphone amp gain |
| OLED display | ER-OLEDM015-SPI | — | SPI display module; **changed from the originally planned SSD1306 part** (verify controller compatibility before reusing old pin-mapping notes) |
| Effect potentiometers ×8 | PTV09A-4225F-B104 | POT_1…POT_8 | Read through the mux below |
| Analog mux | CD4051BM | — | Routes the 8 pots to a single ADC channel |
| Mechanical switches ×2 | 100SP1T1B4M2QE | SW1, SW2 | |
| Buttons ×2 | 1825910-6 | BTN1, BTN2 | |
| Boot mode switch (RUN/DFU) | SS12D10G5 | SW_1 | Selects BOOT0 state |
| Reset button | TS-1088-AR02016 | SW_2 | |
| Audio jacks ×3 | AudioJack4 (3.5 mm) | J2, J3, J4 | Line in, line out, headphone out |
| microSD slot | HRS DM3CS-SF | J1 | Right-angle SMT microSD holder, on SDIO |
| SWD / IO expansion header | JST-GH 2×5, 1.27 mm | CN1 | Shared header for debug + expansion |
| Crystal | X1E000210127000G | X1 | MCU clock |
| USB-C receptacle | USB4105-GF-A (generic KiCad symbol) | — | |
| ESD protection (USB) | USBLC6-2SC6 | — | |
| ESD protection array (MCU IO) | RCLAMP0521P-N | — | |
| Buck regulator (digital 3V3) | TLV62569DBV | — | |
| LDO regulator (analog 3V3) | SPX3819M5-L-3-3 | — | |

Full BOM with LCSC part numbers lives in `GalernaDsp.csv` at the project root, and
JLCPCB assembly data lives in `jlcpcb/production_files/`.

## Power section

Power comes in over USB-C and is split into two 3.3 V rails:

- **3V3_D** (digital) — switching buck regulator, TLV62569DBV.
- **3V3_A** (analog) — LDO, SPX3819M5-L-3-3, feeding the codec's analog domain for lower
  noise than the buck rail could provide.

### Current budget (3.3 V rails)

| Rail | Load | Estimate |
|---|---|---|
| 3V3_D | STM32F405 | 90–130 mA |
| 3V3_D | OLED | 5–25 mA |
| 3V3_D | Mux + pullups, misc | 5–15 mA |
| 3V3_D | **Budget** | **~200 mA** |
| 3V3_A | Audio codec | ~20 mA typical |
| 3V3_A | **Budget** | **~25–50 mA** |

Total ≈ 250 mA at 3.3 V (≈ 0.83 W). At 85–92% buck efficiency that's roughly 180–200 mA
drawn from 5 V USB — comfortably inside USB power limits.

### Analog rail filtering

After the LDO, a ferrite bead plus local decoupling filters the MCU's `VDDA` pin (ferrite
beads add impedance at high frequency, keeping switching noise off the analog supply):

![MCU VDDA decoupling](mcu_vdda_decoupling.png)

### Buck converter inductor selection

$$
L = \frac{V_{out} \cdot (V_{in} - V_{out})}{V_{in} \cdot f_{sw} \cdot \Delta I_L}
$$

Where $\Delta I_L$ (inductor ripple current) is targeted at 20–30% of max output current.

- $V_{out} = 3.3\text{V}$, $V_{in} = 4.5$–$5\text{V}$, $f_{sw} = 1.5\text{MHz}$, $I_{max} = 250\text{mA}$
- $\Delta I_L = 0.200\text{A} \times 0.25 = 0.0625\text{A}$
- Calculated $L \approx 12\mu H$ → used a 10 µH inductor.

### Fuse options considered

- Littelfuse 0603L050YR — 500 mA hold, 0603, easiest SMT routing.
- 1206 SMD 500 mA PPTC — larger pads, easier hand/JLC assembly.
- 1210 0.5 A resettable fuse — most robust, needs more board space.

## Audio codec (ES8388)

Use LINE IN, not MIC IN.

### Decoupling

Per supply pin: 1× 100 nF MLCC at the pin (shortest loop to GND). Per rail (shared): 1×
bulk cap (4.7–10 µF) near the codec.

- Digital: 10 µF bulk + 100 nF on DVDD + 100 nF on DBVDD.
- Analog: 10 µF bulk + 100 nF on AVDD + 100 nF on HPVDD (HPVDD unused on this board).

### Input path

```
AUDIO_IN → 1k → ● → 2.2µF → CODEC_IN
                 │
                 ├─ 200k → GND
                 └─ 220p → GND
```

- **2.2 µF AC coupling**: blocks DC, lets the codec bias the input to VMID. Forms a
  high-pass with the codec's ~20–30 kΩ input impedance: $f_c \approx 3.6\text{Hz}$, well
  below the audio band.
- **200 kΩ to GND**: discharge path for the coupling cap; sets input impedance with
  nothing plugged in.
- **220 pF / 1 kΩ RF filter**: rolls off RF picked up by the cable ($f_c \approx 723\text{kHz}$);
  the resistor also limits current during ESD events.

### Output path

Concerns between the codec and the 3.5 mm jacks: DC compatibility (codec biased to
VMID ≈ 1.65 V, jack is 0 V), RF/EMI filtering, output impedance/stability, and
protection against ESD/hot-plugging.

- **AC coupling (~1 µF)**: blocks the codec's DC bias so the jack sees a signal centered
  at 0 V. Without it: DC on external gear, pops, possible damage.
- **Series resistor (~100 Ω)**: isolates the codec from cable capacitance, improves
  driver stability, reduces pop energy, limits fault current on a shorted jack.
- **Pulldown (~100 kΩ to GND)**: defines the output with nothing connected, reduces
  plug/unplug pops, negligible effect on audio level.

## Headphone amplifier (TPA6110A2DGN)

Added beyond the original spec to drive headphones directly from a dedicated 3.5 mm
jack, with its own volume pot (R16, RK09L12D0A1W). See git history for the original
routing notes; layout follows standard TPA6110 application guidance (star ground for
the amp, short high-current loops to the output caps/jack).

## Crystal oscillator

Pierce oscillator, per ST's reference circuitry:

![Pierce oscillator circuitry](crystal_oscillator_pierce.png)

$R_{ext}$ and $C_L$ form a low-pass filter for harmonics above the crystal fundamental;
$R_{ext}$ also reduces drive strength over the crystal to cut harmonics.

- Values to choose: crystal (Q), load capacitance ($C_{L1}$/$C_{L2}$), $R_{ext}$.
- Values to account for: stray capacitance $C_s$ (typ. 3–5 pF), internal feedback
  resistor $R_F$.
- Example crystal: $f_0 = 16\text{MHz}$, $C_0 = 9\text{pF}$, ESR ≈ 80 Ω, 3225 package.

$$
C_L = 2 \cdot (C_0 - C_s) \approx 8\text{–}12\text{pF}
$$

$$
R_{ext} \approx 10\% \times \frac{1}{2\pi f_0 C_0} = \frac{1}{2\pi \times 24\text{MHz} \times 9\text{pF}} \approx 73\,\Omega \Rightarrow \text{100 }\Omega \text{ used}
$$

## Footprints used

| Part | Footprint |
|---|---|
| Caps 22 µF | 0805 (2012 metric) |
| Caps 10 µF | 0603 |
| Caps 1 µF | 0603 (1608 metric) |
| Caps 100 nF | 0402 (1005 metric) |
| Caps 1 nF / 10 pF | 0402 (1005 metric) |
| LEDs | 0603 or 0805 |
| Resistors | 0402 |
| Ferrite beads | 0805 or 0603 |

## PCB design notes

- 4-layer stackup: F.Cu / In1.Cu (plane) / In2.Cu (plane) / B.Cu.
- No controlled impedance needed — the only differential pair is USB FS, and it isn't
  critical.
- No split ground plane between analog and digital.
- Connectors for GPIO expansion use JST-GH.
- microSD placement followed the LeDsp module's layout as a reference.

## Firmware status

`cubeMx/` holds an STM32CubeMX-generated project for the STM32F405RGT6 with ADC1, I2C2
(codec control), I2S2 (audio data), SDIO (microSD), SPI1 (OLED), and USB_OTG_FS
configured. As of now it's the generated peripheral-init skeleton only — no DSP effect
processing has been implemented in application code yet.
