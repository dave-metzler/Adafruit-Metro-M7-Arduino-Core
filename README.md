[README.md](https://github.com/user-attachments/files/34704425/README.md)
# Arduino core for the Adafruit Metro M7 (i.MX RT1011) — unofficial, v0.1

Written from scratch (not derived from the Teensy cores) against NXP's register
definitions.

**Verified on a real board:** boot via TinyUF2, GPIO (D13), `millis`/`delay` timing
(1 Hz Blink), UART `Serial1` loopback and baud rate (9600, within 3 %), the CPU
clock (measured 500 MHz, already set up by the bootloader, so this core does not
touch it), `analogRead`, `attachInterrupt`, PWM, SPI (loopback), and I2C (with a real device). Everything else below is written but untested.

## Status

| Feature | State |
|---|---|
| Boot via TinyUF2 (app at 0x6000C000), vector table, FlexRAM, C++ constructors | **hardware-verified** |
| `pinMode` / `digitalWrite` / `digitalRead` | D13 verified; other pins written |
| `millis` / `micros` / `delay` / `delayMicroseconds` (GPT1, CPU-clock independent) | **hardware-verified** |
| `Serial1` on LPUART1, D0 (RX) / D1 (TX), interrupt-driven RX | **hardware-verified** (loopback + baud, via `libraries/MetroM7Examples/examples/UartSelfTest`) |
| Fault indicator: a crash makes D13 blink very fast | written |
| `analogRead` / `analogReadResolution` on A0..A5 (ADC1, 12-bit native, default 10-bit result) | **hardware-verified** (0 V / 1.65 V / 3.3 V read 0 / 50 / 100 %, `libraries/MetroM7Examples/examples/AnalogSelfTest`) |
| `attachInterrupt` / `attachInterruptParam` / `detachInterrupt` (any pin; LOW/HIGH/RISING/FALLING/CHANGE) | **hardware-verified** (RISING/FALLING/CHANGE/detach/param, `libraries/MetroM7Examples/examples/InterruptSelfTest`) |
| PWM: `analogWrite`, `analogWriteResolution`, `analogWriteFrequency` on D5, D6, D8..D13 (FlexPWM1; default 8-bit at 4 kHz; pairs D5/D6, D8/D9, D10/D11, D12/D13 share a frequency) | **hardware-verified on D12/D13** (duty 25/50/75 %, 4 kHz and 1 kHz, constant levels; PWM clock measured 125 MHz; `libraries/MetroM7Examples/examples/PwmSelfTest`). D5, D6, D8..D11 not yet exercised |
| SPI (`#include <SPI.h>`): master on LPSPI1, MISO/MOSI/SCK on the ICSP header, all 4 modes, MSB/LSB first, `transfer`/`transfer16`/buffer; chip select is a GPIO of your choice | **hardware-verified** (loopback: modes 0-3, MSB/LSB, 16-bit, buffers of 18 lengths, 12 MHz; timing at 1 and 10 MHz; LPSPI source clock measured 105.6 MHz; `libraries/MetroM7Examples/examples/SpiSelfTest`). Not yet tried with a real device |
| I2C (`#include <Wire.h>`): controller mode on LPI2C1 (SDA/SCL pins and STEMMA QT), 10 kHz to 1 MHz, repeated start, Arduino error codes 0-4. Peripheral (slave) mode **not implemented** | **hardware-verified** (bus idle, full scan, probe timing at 100/400 kHz, and a real device found at 0x6A with a 1-byte read; `libraries/MetroM7Examples/examples/WireSelfTest`) |
| `Serial.printf()` / `Serial1.printf()` (any `Print`; incl. `%f`, long lines) | **written, awaiting hardware test** (`libraries/MetroM7Examples/examples/SerialPrintf`) |
| Native USB `Serial` (CDC-ACM on the USB-C port; full speed, interrupt driven, 1200-baud touch reboots to TinyUF2) | **hardware-verified**: enumeration, echo 128 KiB both ways, device→host 1 MB/s; 1200-baud touch reboot still untested after the fix (`libraries/MetroM7Examples/examples/UsbDiag`, `libraries/MetroM7Examples/examples/UsbSerialTest`, `tools/usb_check.py`) |
| SD library support, NeoPixel | **not implemented** |
| CPU clock | 500 MHz, set by TinyUF2 and left alone (measured with `libraries/MetroM7Examples/examples/CpuSpeed`) |

## Install

1. Put `hardware/adafruit-metro-m7` in your sketchbook's `hardware/` folder
   (or in the IDE's `hardware/` folder), restart the IDE.
2. Toolchain: `arm-none-eabi-gcc` must be on your PATH (any Arduino ARM core, e.g.
   *Arduino SAMD Boards*, installs one), or set `compiler.path` in `platform.txt`.
   Python 3 is required (UF2 packing and upload).
3. Board: *Adafruit Metro M7 (iMX RT1011) with microSD*.
4. Upload: double-click RESET so the `METROM7BOOT` drive appears, then Upload.
   If auto-copy can't find the drive, use *Sketch → Export Compiled Binary* and drag
   the `.uf2` onto the drive.

Command line: `tools/build.sh hardware/adafruit-metro-m7/imxrt1011/libraries/MetroM7Examples/examples/Blink/Blink.ino` (set `ARM_BIN` if needed).

## Staged bring-up (do these in order; there is no SWD debugging)

1. **`libraries/MetroM7Examples/examples/Blink`** – D13 LED blinks at 1 Hz. Proves link address, startup, GPIO, timing.
   * LED blinking *very* fast = the CPU faulted (hard fault etc.).
   * Nothing at all = the bootloader didn't jump in, or startup hung. See "If it doesn't boot".
2. **`libraries/MetroM7Examples/examples/UartSelfTest`** – no adapter needed: jumper D1→D0 and D1→D2; result on the LED (see the sketch header).
3. **`libraries/MetroM7Examples/examples/InterruptSelfTest`** – jumper D3→D2; result on the LED.
4. **`libraries/MetroM7Examples/examples/PwmDiag`** – no wiring: reports the measured PWM clock in MHz, then breathes D13.
   **`libraries/MetroM7Examples/examples/PwmSelfTest`** – jumper D12→D2; D13 breathes, then 8 measured tests reported on the LED.
5. **`libraries/MetroM7Examples/examples/SpiSelfTest`** – jumper MOSI→MISO on the ICSP header; 7 loopback tests reported on the LED.
6. **`libraries/MetroM7Examples/examples/WireSelfTest`** – no wiring needed (attach an I2C device if you have one); LED reports.
7. **`libraries/MetroM7Examples/examples/AnalogSelfTest`** – jumper A0 to GND or 3V3; reading shown as a percentage on the LED.
8. **`libraries/MetroM7Examples/examples/SerialHello`** (UART, `Serial1`) – 3.3 V USB-serial adapter: adapter RX→D1, TX→D0, GND→GND,
   115200 baud.

### If it doesn't boot
The bootloader is not overwritten by uploads (the app lives at 0x6000C000 and up), so
you can always double-click RESET and flash a different UF2, including Adafruit's
CircuitPython UF2, to get back to a known state.

## Things I could not verify — check these first if something is off

(Resolved by hardware testing: GPT1 prescaler, UART clock, GPIO1 routing for GPIO_03.)

1. **GPIO routing.** All 28 pad-to-GPIO/bit assignments were cross-checked against
   CircuitPython's independent RT1011 pin tables (0 mismatches). D13, D2 (input) and the UART
   pads have been exercised on hardware; D4..D6 (GPIO2) have not.
2. **FlexRAM split.** Left as the bootloader has it: 32 KB ITCM, 32 KB DTCM (data, stack, USB DMA structures), 64 KB OCRAM (heap). `startup.c` does not reconfigure FlexRAM: our own splits broke USB DMA.
3. **App size limit** is 2 MB (`FLASH_LEN` in the linker script); the rest of the 8 MB is left free.
4. TinyUF2 might re-enter itself or leave state I didn't anticipate; startup masks
   and clears all NVIC IRQs and disables any enabled watchdog first for that reason.

## Layout
- `cores/imxrt1011/imxrt1011.h` – generated by `tools/gen_regs.py` from NXP's SVD
- `cores/imxrt1011/api/` – Arduino's ArduinoCore-API (LGPL-2.1+)
- `libraries/SPI/` – SPI library (LPSPI1)
- `libraries/Wire/` – Wire library (LPI2C1, controller mode)
- `variants/metro_m7/` – pin table (`variant.cpp`), pin names, linker script
- `tools/bin2uf2.py` – UF2 packer (`--selftest` re-packs Adafruit's own CircuitPython
  UF2 and checks it is byte-identical)

## Pin map (from the board's schematic)
D0 RX=GPIO_09 · D1 TX=GPIO_10 · D2=GPIO_13 · D3=GPIO_12 · D4..D6=GPIO_SD_00..02 ·
D7=GPIO_11 · D8=GPIO_08 · D9=GPIO_07 · D10=GPIO_06 · D11=GPIO_05 · D12=GPIO_04 ·
D13/LED=GPIO_03 · A0=GPIO_AD_02 · A1=GPIO_AD_01 · A2=GPIO_AD_00 · A3=GPIO_AD_05 ·
A4=GPIO_AD_10 · A5=GPIO_AD_08 · SDA=GPIO_01 · SCL=GPIO_02 · MISO/MOSI/SCK=GPIO_AD_03/04/06 ·
SD_CS=GPIO_AD_14 · SD_CD=GPIO_AD_11 · NeoPixel=GPIO_00

## Licenses and credits
- Register definitions: NXP MIMXRT1011 SVD, BSD-3-Clause.
- Pin assignments: Adafruit Metro M7 microSD board files (CC BY-SA); ADC channel numbers
  and GPIO bit numbers cross-checked against CircuitPython (MIT).
- `api/`: ArduinoCore-API, LGPL-2.1-or-later.
- Remaining code: written for this project; choose your own license (LGPL-2.1+ is the
  natural fit for an Arduino core).

## USB Serial

`Serial` is the native USB port; the D0/D1 UART is `Serial1`. USB starts before `setup()`.
Default VID:PID is 1209:0001 (pid.codes shared test ID); override with `-DUSB_VID=... -DUSB_PID=...`.
Test order: `UsbDiag` (LED shows how far enumeration got), then `UsbSerialTest` plus
`python3 tools/usb_check.py` (needs pyserial). The 1200-baud-touch reboot is untested;
double-click RESET still always works.


## Installing through the Arduino Boards Manager

1. Host the files from `dist/` on a release page (e.g. GitHub Releases) and generate the index with
   `tools/build_release.sh <base-url> "<your name>" <website> <email>`.
2. In the Arduino IDE: *File > Preferences > Additional boards manager URLs*, add the URL of
   `package_metro_m7_index.json`, then install **Adafruit Metro M7 iMX RT1011 (unofficial)**.
3. The package pulls the xPack `arm-none-eabi-gcc` 15.2.1 toolchain (~300 MB, once) and a small
   `metro-uf2` helper (UF2 conversion + upload).  No Python is needed.
4. Select the board and port; **Upload** resets the board with the 1200-baud touch (needs a sketch that
   is running USB Serial) and copies the UF2 onto the `METROM7BOOT` drive.  Otherwise double-click RESET first.

Supported hosts: Linux x86-64 and arm64, macOS Intel and Apple Silicon, Windows 64-bit.
For a manual install (copy the `imxrt1011` folder into `hardware/`), add a `platform.local.txt` that sets
`compiler.path=` (see the comment at the top of `platform.txt`) and a `tools.metro_uf2.path=` pointing at a
folder containing a `metro-uf2` binary built from `tools/metro-uf2`.

## Examples

All example and self-test sketches live in the `MetroM7Examples` library bundled with the core
(`hardware/adafruit-metro-m7/imxrt1011/libraries/MetroM7Examples/examples/`).  In the Arduino IDE they appear under
*File > Examples > MetroM7Examples* once the Metro M7 board is selected.  The SD library brings its own stock examples.

## License

LGPL-2.1-or-later (see `LICENSE`); third-party parts and the GPL-3.0 SD library are listed in `NOTICE.md`.
