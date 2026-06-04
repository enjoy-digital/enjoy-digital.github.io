---
title: "LiteX M2SDR: a familiar RFIC, a different approach"
date: 2026-06-03T12:00:00+02:00
draft: false
description: "Building AD9361 SDRs for clients since 2014, we made our own: an open M.2 board where the RFIC is ordinary and the SoC is where the innovation happens. How LiteX lets you reuse the same PCIe lanes for Ethernet or SATA, why it makes LiteX better, and the White Rabbit timing work with FEMTO-ST."
summary: "Our own open AD9361 SDR in an M.2 form factor, built on LiteX. Why we kept the ordinary RFIC, the transports (PCIe, Ethernet, SATA), the White Rabbit timing work, and what's next."
tags: ["litex", "sdr", "fpga", "ad9361", "litepcie", "liteeth", "white-rabbit", "hardware"]
categories: ["hardware"]
showHero: false
---

*TL;DR: building AD9361 SDRs for clients since 2014, we also built our own open one: the LiteX
M2SDR, an M.2 board with an ordinary AD9361 RFIC and a Xilinx Artix-7. The point is not the chip,
it is the SoC: with LiteX you reuse the same M.2 PCIe lanes for PCIe, Ethernet or SATA, add SerDes,
and discipline it with PTM, PTP or White Rabbit. It is commercial now, customers are happy, and it
keeps making LiteX itself better.*

{{< figure src="img/m2sdr-board.png" alt="The LiteX M2SDR board, annotated: M.2 PCIe connector, 100 MHz XO, JTAG, SI5351 clock generator, Xilinx XC7A200T FPGA, ADI AD9361 RFIC, TX/RX connectors, clock in/out and sync in/out" caption="The LiteX M2SDR: an M.2 2280 board with a Xilinx XC7A200T and an ADI AD9361, designed with Lambdaconcept, gateware and software by Enjoy-Digital." >}}

## "So, what's the RFIC?"

That is always the first question about a new SDR, so: another AD936X-based one. 😄

It is a fair thing to raise an eyebrow at. We have been designing FPGA SDRs around this chip for
clients for about ten years, and so has half the open-source SDR world. But that is also the point.
The AD9361 is proven, well understood, and nowhere near tapped out, and we kept wanting to show what
becomes possible when you stop treating the RFIC as the interesting part and put the innovation in
the SoC instead. The M2SDR is our own take on that, with no client NDA in the way: fully open
gateware and software.

## A decade of the same chip, on purpose

A lot of our contracting has been exactly this since 2014: RF systems on FPGA for a range of SDR
companies and integrators, around the AD9361 and plenty of other RFICs. You see the same pattern
every time. The RFIC is a known
quantity; the hard, valuable, differentiating work is everything around it, the data movement, the
host interface, the timing and synchronization, the on-board processing. That is where projects live
or die.

The AD9361 is the simple end of what we do, though. Over the years we have built around more
advanced RFICs too: the AD9371 (2018) and the ADRV9026/9 family (around 2022), with our
[JESD204B core](https://github.com/enjoy-digital/litejesd204b) handling the high-speed JESD links
those parts use. Some of this work has been in production for years, well before the XTRX or the
M2SDR: our LiteX and LitePCIe gateware powers Amarisoft's [Callbox](https://www.amarisoft.com/), and
Amarisoft is a long-standing LiteX user and customer.

We have also worked with the Lime LMS7002M on two fronts: the
[LimeSDR gateware](https://github.com/myriadrf/LimeSDR_GW), which we developed for Lime under
contract, and the [LiteX XTRX](https://github.com/enjoy-digital/litex_xtrx) design, which was our own
initiative. The XTRX in particular was our first full LiteX SDR SoC, with an initial SoapySDR driver,
and it is the work we reused to bootstrap the M2SDR gateware and software.

To be clear about where the M2SDR sits in that range: it is the small, open end, built on the same
LiteX foundations as the serious designs but aimed at hackers and experimenters rather than at our
partners' and customers' markets.

So with the M2SDR the effort went into the SoC, built with
[LiteX](https://github.com/enjoy-digital/litex), and the software stack on top. It is a small board
(M.2 2280) with a Xilinx XC7A200T and an AD9361, 2T2R 12-bit at up to 61.44 MSPS (122.88 with oversampling), and a
PCIe Gen2 x4 link giving roughly 14 Gbps of TX/RX bandwidth through
[LitePCIe](https://github.com/enjoy-digital/litepcie). The FPGA's base infrastructure uses only a
fraction of the part, so there is real room left for RF processing. The hardware is designed by our
partner [Lambdaconcept](https://shop.lambdaconcept.com/); we do the gateware and software.

## The different approach: innovate around the chip

Here is the part that the form factor hides. An M.2 connector carries a handful of high-speed lanes,
and because the whole thing is a LiteX SoC, what those lanes *are* is a build option. The same board
can be:

- **PCIe** up to Gen2 x4 ([LitePCIe](https://github.com/enjoy-digital/litepcie)), with MMAP and DMA
  for raw or processed IQ.
- **1G / 2.5G Ethernet** ([LiteEth](https://github.com/enjoy-digital/liteeth)) over the same lanes.
- **SATA** ([LiteSATA](https://github.com/enjoy-digital/litesata)) to record and play samples
  straight to an SSD.
- **SerDes inter-board links** ([LiteICLink](https://github.com/enjoy-digital/liteiclink)) for
  coherent multi-board setups.

{{< figure src="img/architecture.svg" alt="Diagram: the AD9361 connects to an Artix-7 LiteX SoC, which fans out to LitePCIe, LiteEth, LiteSATA and LiteICLink, with a timing-and-sync bar for PCIe PTM, Ethernet PTP and White Rabbit" caption="The RFIC is fixed and ordinary. Everything interesting is in the SoC: pick your transport, add processing, discipline the clock." >}}

In the M.2 slot you get PCIe directly. The Ethernet and SATA options come when the board is mounted
on the [LiteX Acorn Baseboard](https://github.com/enjoy-digital/litex-acorn-baseboard) (the Mini
variant), which we originally developed around the SQRL Acorn CLE215+ (another M.2 Artix-7 card).

{{< figure src="img/baseboard-mini.svg" alt="The M2SDR in the M.2 slot of the Acorn Baseboard Mini, with a PCIe x1 edge, two SFP cages, a SATA port, GPIO, and USB-C for JTAG/UART and power" caption="On the Acorn Baseboard Mini: a PCIe x1 edge to the host, two SFP cages (1G/2.5G Ethernet, and the White Rabbit / PTP input), a SATA port for an SSD, GPIO, and USB-C for JTAG/UART and for power." >}}

Add to that the LiteX debug story (host bridges and a [LiteScope](https://github.com/enjoy-digital/litescope)
logic analyzer to see inside the running SoC) and multiboot for safe remote updates over PCIe or
Ethernet, and the board stops being "an AD9361 SDR" and becomes a flexible platform that happens to
have an AD9361 on it. That reframing is the whole idea.

One combination I especially like: put the board on the baseboard, discipline its clock over PTP or
White Rabbit, and record or replay IQ straight to an SSD. SATA III (about 550 to 600 MB/s usable)
comfortably handles the AD9361 at full rate (61.44 MSPS, 2T2R, samples carried as 16-bit IQ is
roughly 490 MB/s), one direction at a time, into a disk that can hold terabytes. Recording
time-disciplined IQ straight to a local SSD, with no host in the loop and all from stock parts, is
hard to put together any other way.

## It makes LiteX better, too

There is a feedback loop here that I care about as much as the product. A real, commercial,
hardware-validated SDR is an unforgiving test of the cores underneath it. Pushing LitePCIe to
~14 Gbps, running LiteEth at 2.5G, recording to an SSD over LiteSATA, getting PCIe PTM and White
Rabbit to actually lock: every one of those exercises a LiteX core harder than most projects do, on
real hardware, against real software, in CI. The bugs and rough edges this shakes out get fixed
upstream, so the whole LiteX ecosystem gets a little more solid and better tested because this one
product exists. (It is also why the project ships a real
[debugging guide](https://github.com/enjoy-digital/litex_m2sdr/blob/main/doc/debugging-guide.md),
linked from the [AI-era post](/posts/ai-era-fpga/).)

## Timing, synchronization, and FEMTO-ST

The part that has grown the most lately is timing. The board can be disciplined from PCIe **PTM**
(Precision Time Measurement), Ethernet **PTP**, or **White Rabbit**, the CERN-born scheme for
sub-nanosecond time and frequency transfer.

{{< figure src="img/timing-sync.svg" alt="Diagram: White Rabbit, Ethernet PTP, PCIe PTM and an external 10 MHz/PPS feed into the M2SDR clock discipline (SI5351 + AD9361 reference), giving PPS/10 MHz outputs, a disciplined RFIC sample clock, and many boards on one timebase" caption="Many ways to discipline the same board: White Rabbit or PTP over the SFP, PTM over PCIe, or an external 10 MHz and PPS, all feeding the SI5351 and the AD9361 reference clock." >}}

Our White Rabbit work lives in [litex_wr_nic](https://github.com/enjoy-digital/litex_wr_nic). It
started as a project for the **SPEC A7**, a low-cost White Rabbit board developed by the Warsaw
University of Technology: we integrated White Rabbit into LiteX and made it reusable across
hardware, first on the SPEC A7 and the SQRL Acorn, and now on the M2SDR.

The M2SDR step is thanks to **FEMTO-ST**. The time-and-frequency group in Besançon, the
[oscimp / OscillatorIMP](https://github.com/oscimp) people around Jean-Michel Friedt, work exactly
at this intersection of White Rabbit, ultra-stable oscillators and SDR. They adapted the design to
drop the dedicated VCXO it used to require (their LiteX White Rabbit work is open at
[oscimp/wr_acorn](https://github.com/oscimp/wr_acorn)), and presented the result at the
[2026 White Rabbit Workshop](https://indico.cern.ch/event/1623507/contributions/7073405/attachments/3278877/5866098/FEMTO-ST.pdf).

With that in place we have locked an M2SDR as a White Rabbit *slave* to a SPEC-A7 White Rabbit
*master*: every board gets a clean PPS and 10 MHz (and from there the RFIC reference clock) off the
WR network, so a rack of M2SDRs can share one timebase for coherent work. A cheap, open,
WR-disciplinable SDR is a genuinely useful thing to have on a metrology bench. 🙂

## Where it is, and where it goes

The LiteX M2SDR is fully commercialized and on the [Enjoy-Digital shop](https://enjoy-digital-shop.myshopify.com/),
in two clocking variants, tested with the usual SoapySDR-compatible software and our own C tools and
library. Those utilities run over both PCIe and Ethernet today, with the transport chosen at
runtime; they are Linux-first, and macOS and Windows support is on the way. Early customers are
genuinely happy with it, which is the best signal we could ask for.

And it is only the start. We are actively extending it: SATA record/playback, the 1G/2.5G Ethernet
path, and tighter synchronization through the White Rabbit work and the FEMTO-ST collaboration. If
you do SDR, time-frequency, or just want a big open FPGA with an RFIC bolted on, come say hello.

---

*Code: [github.com/enjoy-digital/litex_m2sdr](https://github.com/enjoy-digital/litex_m2sdr).
Hardware by [Lambdaconcept](https://shop.lambdaconcept.com/), available from the
[Enjoy-Digital shop](https://enjoy-digital-shop.myshopify.com/).*

*Work and ideas by Enjoy-Digital (with Lambdaconcept, and the White Rabbit collaborators above);
written up with AI in the loop.*
