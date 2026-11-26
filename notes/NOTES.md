# Devboard notes

Short notes on everything I've learned building this board. Pictures for most of it are in [visuals.html](visuals.html).

## Electricity basics
- Voltage = pressure (V), current = flow (A), resistance = a narrow pipe (Ω)
- Current only flows in a **complete loop**. Break it anywhere and everything stops.
- **GND** = the return path and the 0V reference. "5V" means 5V higher than GND.
- Voltage divider: two resistors in series, and the middle point sits at a voltage in between (this is how USB-C CC works)
- Capacitor = a small energy tank that covers sudden spikes in demand
- AP Physics 2 / PHYS G280 cover Ohm's law, Kirchhoff, capacitors, and RC circuits. Semiconductors, MCUs, USB, and PCB design come from EE classes or building.

## Words
- **MCU**: a microcontroller, a small computer on one chip
- **Devboard**: a board that makes an MCU easy to power, program, and plug things into
- **VBUS**: USB's 5V power wire
- **D+ / D-**: USB 2.0 data pair. Mirrored signals cancel out noise (a differential pair).
- **CC**: USB-C's "who's plugged in" pins
- **Pull-up / pull-down**: a resistor from a pin to power / to GND so it sits at a known voltage
- **Host / device**: the host is in charge (laptop), the device gets plugged in

## Reading a schematic
- Reference letters: **J** connector, **R** resistor, **C** capacitor, **U** chip, **SW** switch
- Lines = wires. Same wire = same electrical point.
- **Power symbols** (GND, +3V3, VBUS): every symbol with the same name is connected
- **Labels** work the same way, like walkie-talkie channels. A label alone connects to nothing.
- A capacitor touching a pin does **not** put the pin on a power net. The pin still needs its own GND or +3V3 symbol.
- **Value** = what it is electrically (5.1k, 1uF). **Footprint** = its physical pads on the board.

## Symbols, footprints, pads
- **Symbol**: the drawing in the schematic (what it does)
- **Footprint**: the set of copper pads on the board (where it physically goes)
- **Pad**: bare copper where a leg gets soldered. Green solder mask covers everything else.
- Pin numbers must match pad numbers. Always check the footprint against the datasheet.
- **SMD pads** are flat on the surface. **Through-hole pads** are copper rings around a drilled hole.
- **0805** = 0.08 × 0.05 in (2.0 × 1.25 mm). An easy size to hand-solder.
- Parts come onto the PCB from the schematic (**Tools → Update PCB from Schematic**), not placed straight on the board

## USB-C (J1)
- Reversible: row B is row A rotated 180°, so tie **A6↔B6 (D+)** and **A7↔B7 (D-)**
- VBUS pins A4, A9, B4, B9 → all tied together
- GND pins A1, A12, B1, B12 + SHIELD → GND
- SBU1/SBU2: unused here, so no-connect X
- **CC1 and CC2 each get their own 5.1kΩ to GND** (R1, R2). The charger sees them and turns 5V on.
- Math: charger's 56k pull-up + my 5.1k pull-down → about 0.42V = "device found"
- No CC resistors = no power from most USB-C chargers

## Power (U1, C1, C2)
- ESP32 runs on **3.3V**. 5V would damage it.
- **AP2112K-3.3** LDO regulator: VIN ← 5V, VOUT → 3.3V, EN tied to VIN (always on), NC gets an X
- Swapped from the XC6206 (200 mA max) because Wi-Fi bursts pull up to ~500 mA (AP2112K does 600 mA)
- C1 = 1µF on the input, C2 = 10µF on the output. Without the output cap the regulator can oscillate.
- A part's **value** is what goes on the parts list. Changing the symbol doesn't update the value text.

## ESP32-S3-WROOM-1 (U2)
- Module = chip + flash + crystal + Wi-Fi antenna in one metal can
- Picked over RP2040 / STM32F103 because I want Wi-Fi downloads and enough RAM for GIFs (STM32F103 has only 20 KB)
- 3V3 (pin 2) → +3V3, GND (pins 1, 40, 41) → GND
- **C3 10µF + C4 100nF** right at the 3V3 pin (big cap = slow dips, small cap = fast spikes)
- **EN (pin 3)** = on switch: R3 10k to 3.3V, C5 1µF to GND (slow start), SW1 to GND = reset button
- **IO0 (pin 27)** + SW2 to GND = boot button for download mode
- USB_D+ → pin 14, USB_D- → pin 13 (USB is built in)
- **Strapping pins** IO0, IO3, IO45, IO46: read at startup, so keep sensors off them
- IO35-IO37: used for PSRAM on some module versions, so avoid them

## Screen (J2)
- 1x8 header for an ST7789 240×240 SPI screen module
- 1 GND, 2 +3V3, 3 SCK IO12, 4 MOSI IO11, 5 RES IO8, 6 DC IO9, 7 CS IO10, 8 backlight IO7
- A header instead of soldering the screen on, so screens are swappable
- Check the screen's pin order before buying
- Libraries: AnimatedGIF, TFT_eSPI / Adafruit_ST7789

## Headers (J2, J3)
- Header pins = metal posts snapped off a 40-pin strip, soldered into through-hole pads
- **Pitch 2.54 mm** (0.1 in) = breadboard spacing
- `Conn_01x07_Pin` = 1 row × 7, male. `_Socket` = female.
- `PinHeader_1x07_P2.54mm_Vertical` = 1×7, 2.54 mm pitch, pins straight up
- Male and female fit the same footprint holes, so I can choose when buying parts
- J3 (1x7): GND, 3V3, IO1, IO2, IO4, IO5, IO6 (labels F1 to F5)
- Every other unused ESP32 pin (IO13 to IO18, IO21, IO38, IO39, TXD0, RXD0, ...) has a no-connect X
- Never put 5V into an IO pin (3.3V only)

## ERC
- Open: **Inspect → Electrical Rules Checker** (ladybug icon). No hotkey.
- Pin not connected → wire it or press **Q** for a no-connect X
- Input Power pin not driven → add **PWR_FLAG** on VBUS and GND
- Label connected to only one pin → add the matching label on the other end
- Unconnected wire endpoint → drag onto a pin end or delete the stub
- A wire that *looks* connected might not be. The end has to sit exactly on the pin tip.
- PCB Editor has its own checker: **DRC**

## KiCad hotkeys
- **A** add part, **P** power symbol, **W** wire, **L** label, **Q** no-connect, **E** edit properties
- **Tools → Assign Footprints** to set footprints in bulk
- Right-click → **Change Symbol** to swap a part

## Workflow lessons
- **Cmd+S in both the Schematic and PCB editors.** KiCad sat open for a week and my main files were empty. Everything was only in the autosave (`.history`, **File → Local History**).
- Git: commit + push before switching computers, pull before starting
- KiCad files don't merge well, so don't edit on two machines without pulling
- easyeda2kicad library lives in `lib/` with project library tables, so both computers have it
- New imports: `easyeda2kicad --full --lcsc_id=<part> --output ~/Documents/devboard/lib/easyeda2kicad`

## Resources
- Raspberry Pi "Hardware design with RP2040" / Espressif ESP32-S3-WROOM-1 datasheet
- Phil's Lab and Shawn Hymel's "KiCad: Getting Started" (YouTube)
- SparkFun beginner tutorials, "USB in a NutShell" (beyondlogic.org)
- Wokwi (browser MCU simulator), Falstad (circuit simulator)
- MIT OCW 6.002 Circuits and Electronics

## Next
- ERC is at 0 errors
- Decide how the USB-A port works, add it to the schematic
- Update PCB from Schematic → layout
