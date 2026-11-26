Started the devboard in KiCad 10. Placed a USB-C receptacle (the USB 2.0, 16 pin one) and wired up VBUS, GND, and the data lines. Learned USB-C is reversible because every signal shows up twice (a top row of A pins and a bottom row of B pins), so you tie A6 to B6 and A7 to B7 for D+ and D- and it works whichever way the cable goes in.

Added two 5.1k resistors, one from each CC pin to ground. Didn't know what CC was at first. Turns out USB-C chargers don't send any power until they see a device, and the way they check is by reading the voltage on CC. Your 5.1k pulldown and the charger's pullup make a voltage divider, and that in-between voltage is basically your board raising its hand saying "I'm a device, send me 5V." Each CC pin gets its own resistor since only one of them is used depending on which way the cable is plugged in. (Apparently a lot of cheap gadgets share one resistor and then don't charge on USB-C chargers.)

Added an XC6206 LDO regulator to bring USB's 5V down to 3.3V, plus a 1uF capacitor on the input and output. The capacitors act like tiny water tanks that cover sudden spikes in current, and the regulator's datasheet requires the output one or it can oscillate. This is the RC circuit stuff from AP Physics 2, which I haven't taken yet, so I'm learning it backwards.

Imported an RP2040 footprint with easyeda2kicad and dropped it on the PCB. Later learned you aren't supposed to place footprints straight onto the PCB, they're supposed to come from the schematic through "Update PCB from Schematic", otherwise KiCad doesn't know what it connects to.

NOTE: Found out a week later that I never actually saved anything. KiCad had been open since Nov 13 and my main schematic and PCB files were empty, everything was only in KiCad's autosave folder. Saved properly, then set up a GitHub repo so I can work on it from my desktop too. Also copied my easyeda2kicad library into the repo with project library tables, because the imported parts only existed on my laptop.

Learned the difference between a symbol and a footprint. The symbol is the drawing in the schematic that says what a part does, the footprint is the actual copper pads on the board that the part gets soldered onto. Assigned footprints to everything: 0805 for the resistors and capacitors (0.08 by 0.05 inches, about a grain of rice, which is supposed to be the easiest size to hand solder), SOT-23-3 for the regulator, and the imported one for the USB-C port.

Ran ERC (Inspect > Electrical Rules Checker, it's the ladybug icon, there's no hotkey for it). Big bug: the USB-C port's ground pins were on their own little island and not connected to the rest of GND. Since current only flows in a complete loop, the board would've gotten 5V with no way back and just not turned on. The wire looked connected but its end wasn't exactly on the pin. Fixed it, tied the connector shield to ground too. Added PWR_FLAG symbols on VBUS and GND, because the USB-C pins are "passive" and ERC can't tell they're actually a power source otherwise.

Tried placing an STM32F103 as the brain, then decided on what I actually want this board to do: drive a small color screen that plays GIFs and shows download progress, with the board downloading things itself over Wi-Fi. The STM32F103 only has 20KB of RAM (way too little for GIFs) and neither it nor the RP2040 has Wi-Fi. Switching to an ESP32-S3-WROOM-1 module, which already has the chip, flash, crystal, and antenna inside it, so the board actually gets simpler. Wi-Fi pulls bursts of up to around 500mA though and the XC6206 can only do about 200mA, so the regulator is getting swapped for an AP2112K-3.3 (600mA).

Swapped the regulator with "Change Symbol" and found out the Value field keeps the old text, so it still said XC6206 even though the symbol was the AP2112K. That's the name that goes on the parts list, so I would've ordered the wrong chip. Tied the AP2112K's EN pin to VBUS so it's always on, and bumped the output cap to 10uF for the Wi-Fi bursts.

Placed the ESP32-S3-WROOM-1 and messed up the power on my first try. I wired its 3V3 pin to a capacitor and its GND pins to the other side of a capacitor, which looked connected but wasn't actually on the 3.3V or GND nets. A capacitor isn't a wire. Fixed it by putting the +3V3 and GND power symbols right on the pins.

Added a reset circuit on EN (10k pullup to 3.3V, 1uF to ground so the chip waits for power to settle, and a button to ground) and a BOOT button on IO0 for download mode. Connected the USB data lines to the ESP32 with labels. Learned labels only connect to other labels with the exact same name, so you need one on both ends.

Added an 8 pin header for the screen (GND, 3.3V, then IO7 to IO12 for SPI clock, data, reset, DC, chip select, and backlight) and a 7 pin header that breaks out IO1, IO2, IO4, IO5, IO6 as spare pins, since this is a devboard and the point is plugging stuff in. Avoided the strapping pins (IO0, IO3, IO45, IO46) because the chip reads them on startup to decide how to boot. Put no-connect X's on every other unused pin.

Got ERC from 46 errors down to 0.

Next: Update PCB from Schematic, place the parts, and route the traces.
