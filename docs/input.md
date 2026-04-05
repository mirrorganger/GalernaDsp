# Galerna DSP

Galerna DSP will DSP board based on STM32 to produce audio effects.

## Requirements

- Two audio channels input/output. No audio amplification.
- 48KHz / up to 24 bit sample width
- 8 potentiometers. Read through a Multiplexer
- 2 mechanical switches
- 2 buttons
- USB C powered
- SWD debug
- OLED screen SSD1306 
- IO extension header 

### Power Section 

The power of the board will be done through USB C.

There will be two power rails
 
 - 3.3V for the digital section
 - 3.3A clean analog power mostly for the Audio Codec

For that, two voltage regulators will be used.
 
 - Switching Buck regulator for digital rail
 - Low Voltage Drop (LDO) for the analog rail

## Design choices

- 4 layer board 
- No split GND for analog and digtial
- Audio Input and Output with 3.5mm Jacks.


## PARTS

IC's selected 

| Component                   | Part Name               | LCSC Part Number |
|-----------------------------|-------------------------|------------------|
| Microcontroller             | STM32F405RGT6 (LQFP-64) |                  |
| Audio Codec                 | WM8731SEDS              |                  |
| OLED Screen                 | SSD1306                 |                  |
| Multiplexer                 | CD4051BM96G4            |                  |
| Buck Switching Regulator    | TLV62569DBV             |                  |
| LDO Regulator               | SPX3819M5-L-3-3         |                  |
| USB C Receptacle            | USB4105-GF-A            |                  |
| ESD protection              | USBLC6 - 2SC6           |                  |
| Audio Jacks (x2)            | PJ-320D                 | C431535          |          



### PCB design

No impedance control needed. The Only differential pair is USB FS. Not critical.



# Audio amplifier routing prompt

I'm going to route the Headphone Amplifier (TPA6110A2DGN). There are two images for the Headphone Amplifier schematic and PCB in the context. 

I want you to give me routing recomendations.



