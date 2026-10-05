---
title: "ChromatiX SDR: a spectrum analyzer from the Chromatic's ESP32"
date: 2026-10-05T08:30:00+02:00
draft: false
description: "ESPARGOS' ESP-SDR gets raw I/Q samples out of an ESP32's Wi-Fi radio. The ModRetro Chromatic has an ESP32 next to its FPGA, so we turned the console into a handheld 2.4 GHz spectrum analyzer: captures over the existing QSPI link into PSRAM, an FFT on a VexRiscv, a spectrum/waterfall UI on the LCD, tuning extended to 2150-2880 MHz, and a SoapySDR driver for gqrx and GNU Radio over USB."
summary: "The Chromatic's ESP32 as an SDR: ESP-SDR on the radio side, the FPGA and a RISC-V on the console side, and a USB link to SoapySDR. How the samples move, how the tuning was pushed from the Wi-Fi channels to 2150-2880 MHz, and what the result can and cannot do."
tags: ["litex", "fpga", "sdr", "esp32", "esp-sdr", "soapysdr", "gnuradio", "chromatix", "modretro"]
categories: ["hardware"]
showHero: false
---

*TL;DR: [ESP-SDR](https://espargos.net/espsdr/), from [ESPARGOS](https://espargos.net/), uses an
undocumented debug path in Espressif's radios to capture raw I/Q samples from the ESP32's
2.4 GHz receiver. The ModRetro Chromatic has an original ESP32 wired to its FPGA, and since
[ChromatiX](/posts/chromatix/) the FPGA is a LiteX design we can extend. On the
[`sdr` branch](https://github.com/enjoy-digital/chromatix/tree/sdr), the ESP32 runs ESP-SDR with a
small Chromatic patch, the captures travel over the console's existing QSPI link straight into
PSRAM, and a VexRiscv computes the FFT and draws a spectrum, waterfall and band scan on the LCD
at about 26 updates per second. We also pushed the tuning from the Wi-Fi channel frequencies to
**2150-2880 MHz**. Over USB, the console relays the captures to the PC at 2.4 MB/s, and a SoapySDR
module makes it a source for gqrx, CubicSDR and GNU Radio. It is a burst receiver with a 2.4 GHz
antenna, not a wideband SDR, and the post is clear about that too.*

{{< figure src="img/sdr_console.png" alt="The Chromatic LCD showing the SDR application: frequency header, spectrum around Wi-Fi channel 6 with the receiver passband visible, and a waterfall below it" caption="The console tuned to Wi-Fi channel 6, captured over its own USB video output. Spectrum on top, waterfall below." >}}

## ESP-SDR in two minutes

An ESP32's radio is built for Wi-Fi and Bluetooth. Between the RF front-end (LNA, mixer, filters)
and the CPU sits a fixed-function modem that turns the received signal into packets, and normally
software only ever sees the packets. ESP-SDR, published by ESPARGOS (Florian Euchner) at the end of
September 2026, uses an undocumented debug path that bypasses the modem and hands the ADC output,
I/Q samples, to the CPU. The firmware captures short bursts into internal memory and sends them
over UART or USB. It supports most of the ESP32 family, and its README is open about how it was
done: the path was found and generalised across chips with the help of LLMs.

On the original ESP32, which is the one in the Chromatic, ESP-SDR offers 16, 40 or 80 MS/s, 8 or
10-bit samples, manual gain from 0 to 72 or the hardware AGC, and bursts of up to 16380 samples. A
4096-sample capture at 40 MS/s covers 104 µs. That is the important limitation: these are
snapshots, not a continuous stream, so the receiver sees a few percent of the time at best. For a
spectrum display, a waterfall or hunting bursts, that is fine. For demodulating a continuous
signal, it is not.

## What the console adds

The Chromatic's ESP32 normally runs ModRetro's menu and OSD firmware. Around it, the console
provides everything a handheld SDR needs and the bare chip does not have: a screen, buttons, a
speaker, a battery, a fast USB 2.0 port, and an FPGA connected to the ESP32 by a UART and a quad SPI
link.

The FPGA side reuses the application SoC we built for [Doom](/posts/chromatix-doom/): a VexRiscv
with 8 KB caches at 67 MHz, 7.5 MB of PSRAM as main RAM behind an L2, a tear-free PSRAM frame
buffer, PCM audio and the buttons. `--with-app` builds it, with one addition: a CPU UART wired to
the ESP32's UART0. That build uses 69% of the logic and 48 of the 56 BSRAM, and meets timing at
67 MHz.

{{< figure src="img/data-path.svg" alt="Data path diagram: the ESP32 radio front-end and ADC feed ESP-SDR on the ESP32 CPU, which sends commands and capture headers over UART and capture payloads over 40 MHz quad SPI to the FPGA. In the GW5A, the QSPI slave writes the payloads as PSRAM bursts; the VexRiscv reads them from main RAM, runs the FFT and draws the LCD frame buffer, which goes through the video pipeline to the LCD and UVC. A USB CDC link relays commands and captures to the PC, where SoapyChromatic serves gqrx, GNU Radio and CubicSDR." caption="How a capture travels. The QSPI link and its PSRAM burst writer already existed for the ESP32's on-screen menu; the SDR reuses them as they are." >}}

## Moving the samples

The first test used stock ESP-SDR and the standard ChromatiX bitstream, which bridges the USB
serial port to the ESP32's UART at 2 Mbaud. It worked, at about 31 KB/s, and roughly one capture in
twelve failed its CRC: the bridge's 64-byte FIFO overflowed whenever the host did not poll fast
enough. Good enough to check the receiver works through the console's antenna, not good enough
for anything else.

The second step put the CPU on the ESP32's UART directly, with a 512-byte receive FIFO. That gave
122 KB/s and 50 clean captures out of 50, but a 4096-sample capture still took 67 ms, 41 of them
for the transfer. Stock ESP-SDR cannot go faster than 2 Mbaud on this UART.

The third step used a link that was already there. The ESP32 draws the console's on-screen menu by
writing into the FPGA's PSRAM over quad SPI at 40 MHz, and ChromatiX has a QSPI slave that turns
each 1 KB transfer into a PSRAM burst. Our ESP-SDR patch adds a `QSPI <address>` command: after
it, capture payloads go over QSPI to that PSRAM address and only the `DATA` header (length, CRC32,
duration) stays on the UART. No gateware change was needed. The CPU invalidates its caches, reads
the payload from its main RAM and checks the CRC. A full 16380-sample capture round trip went from
about 170 ms over the UART to **7.3 ms**.

## DSP on a 67 MHz softcore

The console computes the spectrum on the VexRiscv, in C. Each 4096-sample capture is split into 8
segments; each segment gets its DC offset removed and a Hann window, goes through a 512-point
radix-2 FFT, and the 8 power spectra are averaged and converted to a log scale in quarter-dB steps.
The first FFT used 32-bit arithmetic. The current one uses 16-bit samples with block scaling (the
whole block is shifted down only when a stage could overflow), which is enough precision for a
display and much faster on an rv32im without DSP instructions. Twiddle factors come from a table
generated by a Python script. We checked the result against numpy on the same captures: same peak
bins, and a correlation of 0.9999 on the dB shape.

An update takes about 37 ms, 21 of them in the DSP, so the display runs at about 26 updates per
second over QSPI, against 9.5 with the UART version.

## Tuning beyond the Wi-Fi channels

Checking the spectrum against known signals showed two problems. The first was simple: the ESP32's
spectrum is inverted, so every client conjugates the I/Q samples. The reference was the console's
own 24 MHz crystal, whose harmonics at 2400, 2424, 2448, 2472 and 2496 MHz show up as narrow lines
at every tuning, which makes them a convenient frequency ruler.

The second was bigger. ESP-SDR accepts any frequency from 100 to 6000 MHz, but on the original
ESP32 only the Wi-Fi channel frequencies actually moved the local oscillator: 2412 to 2472 MHz in
5 MHz steps, plus 2484. For other frequencies the firmware calibrates the PLL on 2412 MHz and then
writes a new divider, but the VCO's capacitor bank stays calibrated for 2412 MHz and the LO does not
follow. ESP-SDR documents its extended range as not RF-validated, so this is not a bug report; it
just meant the receiver could only look at the Wi-Fi channels.

We fixed it in two steps, in `chromatic_tune.c`, one of the files our build adds to ESP-SDR.

The first step uses a function of Espressif's PHY library, `set_chan_freq_sw_start`, which we found
by disassembling `libphy.a`. Wi-Fi and the Bluetooth PHY use it for their channels: it runs the full
calibration on a given MHz (2400 to 2484) plus a fractional offset. The offset turned out to move
the LO by 1.0546 times its nominal value, a factor we measured from the crystal harmonics and
applied. That gave **2386 to 2504 MHz in 1 kHz steps**, with the LO within ±2.6 kHz of the request,
which is the crystal tolerance between the console and the ESP32.

The second step went further. The PHY calibration reads its targets from an 85-entry PLL table, one
entry per MHz from 2400 to 2484, where each entry holds the VCO capacitor code, the divider (the LO
is 480 MHz × (2 + word/2^20), so any frequency above 960 MHz can be encoded) and a front-end tuning
word. Loading one entry with the divider for the target frequency and a capacitor code close to
the expected result, fitted from the calibration results, makes the calibration lock from
**2150 to 2880 MHz**, repeatably, with the LO within 2 kHz. Below 2150 or above 2880 MHz the
calibration never locks, whatever the starting point; that is the edge of the VCO's range.

On top of that, opening the receiver's low-pass filter at 80 MS/s gives a usable view of about
±38 MHz around the LO. The console and the PC can see roughly 2112 to 2918 MHz.

With the antenna and LNA matched for 2.4 GHz, sensitivity outside the ISM band is lower. Real
signals still come through: the mobile band 1 downlink carriers just below 2170 MHz, and a 20 MHz
LTE band 7 downlink carrier at 2680 MHz, seen at several LO settings and sample rates so it is not
an image.

{{< figure src="img/sdr_host_2655_80msps.png" alt="Host spectrum plot at 80 MS/s centred on 2655 MHz, showing a flat noise floor with crystal harmonic lines and a block-shaped 20 MHz LTE carrier occupying about 2671 to 2689 MHz" caption="An LTE band 7 downlink carrier at 2680 MHz, captured at 80 MS/s and plotted on the PC. Far outside the Wi-Fi channels the stock firmware could tune." >}}

Our ESP32 firmware is ESP-SDR pinned to a specific upstream commit, plus a small patch and two
source files (the QSPI transport and the tuning), all applied by `firmware/esp32-sdr/build.sh`. The
protocol is unchanged, so the same firmware still works as a stock ESP-SDR with ESPARGOS' viewers. We plan to propose the extended tuning to ESP-SDR upstream, since
every original-ESP32 user would benefit from it, not only the Chromatic.

## The handheld UI

The firmware on the console became a small instrument. The header shows the frequency, span,
tuning step and the level at the peak or at the cursor. Under it, a spectrum with a frequency axis,
known bands, Wi-Fi channel numbers, the three BLE advertising channels and an optional peak hold,
then a waterfall.

{{< figure src="img/sdr-ui.png" alt="Three 160x144 screens of the console SDR application: the menu with band presets and settings, a band scan of the 2.4 GHz ISM band, and the spectrum view on the LTE band 7 downlink preset with an 80 MHz span" caption="Menu with band presets, a band scan of the 2.4 GHz ISM band, and the LTE B7 downlink preset at 80 MHz span." >}}

Select switches between three views: spectrum with waterfall, waterfall only, and a band scan that
sweeps a range in 64 MHz steps with wide 80 MS/s captures, from the 2.4 GHz ISM band to the full
2150-2880 MHz range in 12 steps. Left and Right tune (faster when held), Up and Down change the
tuning step from 10 kHz to 20 MHz, A changes the span between 16, 40 and 80 MHz, B brings up a
cursor that A tunes to. The menu has band presets (Wi-Fi channels 1, 6 and 11, Bluetooth, and the
LTE bands in range), gain, reference level, waterfall settings and help pages.

The RSSI tone is our favourite feature. It plays a tone through the speaker, with the pitch
following the level at the cursor, so you can tune to an emitter and walk around the room to find
it without looking at the screen.

## On the PC

To use the console as a regular SDR, the USB serial link had to carry more than the debug bridge.
A small gateware block, `cdc_link`, lets the application switch the USB CDC byte stream from the
UARTBone debug bridge to its own UART for commands and a DMA reader for bulk data from main RAM.
Opening the port at 1200 baud, the same "touch" Arduino boards use, switches it back to the debug
bridge so the host tools keep working.

The console then acts as a relay. It forwards ESP-SDR commands from the PC to the ESP32, answers
the transport commands itself, and after each capture header sends the payload, already in PSRAM
from the QSPI transfer, by DMA. It still displays the relayed captures, and resumes its own
captures two seconds after the PC goes quiet.

On the PC, `scripts/chromatic_sdr.py` covers quick checks (info, benchmark, spectrum, recording to
a CS8 file). `software/SoapyChromatic` is a SoapySDR module that speaks the same protocol, so gqrx,
CubicSDR, GNU Radio and SoapySDR's Python bindings see the Chromatic as a normal device: 16, 40 or
80 MS/s, AGC or a gain element, kHz tuning through `FREQK` with an NCO for the sub-kHz rest, and
burst timestamps. ESPARGOS' own SoapySDR module targets the ESP32-S31 over Ethernet, so this one
fills the original ESP32's spot.

The relay sustains 73 captures of 16380 samples per second, 2.4 MB/s or about 1.2 MS/s of
delivered samples, with no errors over 833 captures (27 MB) in the benchmark. Through SoapySDR,
Python gets 1.0 MS/s and a GNU Radio source 1.12 MS/s. One 512-byte USB packet was lost once in
about 13,000 relayed captures; the clients detect it with the CRC and drop the capture, and the
root cause is still open.

{{< figure src="img/sdr_console_lte2680.png" alt="The console SDR display tuned to 2680 MHz showing the LTE carrier as a raised block in the spectrum and a band in the waterfall" caption="The same LTE carrier at 2680 MHz, this time on the console itself." >}}

## Limits

We want to be clear about what this is. It is a 2.4 GHz-centred receiver that sees roughly 2.1 to
2.9 GHz in bursts of about 100 µs to 400 µs, with a duty cycle around 3% at 40 MS/s. It is good for
watching Wi-Fi and Bluetooth activity, finding emitters, looking at band occupancy and LTE carriers
in range. There is no broadcast FM, no ADS-B at 1090 MHz and no GPS at 1575 MHz: they are outside
the VCO range, and we checked that no second-order or harmonic trick reaches them usefully. The
console's own 24 MHz harmonics show up as spurs.

Flashing ESP-SDR also replaces ModRetro's ESP32 firmware, so while it is installed the console has
no menu, OSD, settings or power management. Back up the ESP32 flash first (the command is below)
and restore it to get the console back. Merging the SDR transport into the ModRetro MCU firmware,
so the SDR becomes a mode rather than a replacement, is on the list.

## Try it

The ESP32 firmware is built from ESP-SDR plus our patch, with the ESP-IDF version ESP-SDR pins, and
flashed through the ChromatiX USB-to-ESP32 bridge of the standard bitstream:

```bash
git clone --recursive -b sdr https://github.com/enjoy-digital/chromatix
cd chromatix

# Back up the ModRetro ESP32 firmware first (restore: write_flash 0 esp32_backup.bin).
esptool.py --port /dev/ttyACM0 --chip esp32 --baud 460800 read_flash 0 0x400000 esp32_backup.bin

# ESP-SDR + Chromatic patch (ESP-IDF sourced), then flash it through the bridge.
./firmware/esp32-sdr/build.sh
(cd firmware/esp32-sdr/build/esp-sdr/build-esp32 && \
    python -m esptool --chip esp32 -p /dev/ttyACM0 -b 460800 write-flash @flash_args)

# Application SoC, SDR firmware.
./chromatix.py --gowin-path ~/tools/gowin_1.9.12.04/IDE --with-app --build --flash
make -C firmware/sdr BUILD_DIR=../../build
./scripts/chromatic.py --serial /dev/ttyACM0 run firmware/sdr/sdr.bin --no-verify
```

Then, on the PC, with SoapyChromatic installed from `software/SoapyChromatic`, gqrx opens it with
the device string `soapy=0,driver=chromatic`. All the measurements, checks and dead ends are in
[`doc/SDR.md`](https://github.com/enjoy-digital/chromatix/blob/sdr/doc/SDR.md).

---

*The foundation of all this is [ESP-SDR](https://espargos.net/espsdr/) by
[ESPARGOS](https://espargos.net/) (Florian Euchner, [esp-sdr](https://github.com/ESPARGOS/esp-sdr),
GPL-3.0): the raw I/Q capture of the ESP32 radio, its firmware and protocol. ChromatiX adds the
Chromatic transport and tuning patch, the console application and the PC side. Thanks to
[ModRetro](https://modretro.com/) for the open MCU and FPGA designs that made the QSPI link easy to
reuse. Built on [LiteX](https://github.com/enjoy-digital/litex) and
[SoapySDR](https://github.com/pothosware/SoapySDR).*

*Work and ideas by Enjoy-Digital; written up with AI in the loop.*
