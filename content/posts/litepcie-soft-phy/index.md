---
title: "LitePCIe soft PHY: PCIe without the hard block, from a 2019 request to Gen3 x8"
date: 2026-11-13T10:00:00+01:00
draft: false
description: "LitePCIe has always relied on the FPGA vendors' PCIe hard blocks. A soft PHY, with the LTSSM, Physical Layer, Data Link Layer and configuration space in the fabric, has been requested since 2019. It now exists: it enumerates with the unmodified LitePCIe driver on Kintex-7 and Kintex UltraScale+, up to 8 GT/s x8, and a contributor has already brought it to the Lattice ECP5, the platform of the original request. At Gen3 x8 it also shows why the hard block still matters."
summary: "A soft PCIe PHY for LitePCIe: the requests since 2019, why it never happened, how it was built in a week, Gen1 to Gen3 and x1 to x8 on hardware, what Gen3 x8 costs in an XCKU3P (with a device view), and the ECP5 port by Dirk Eibach on the Versa board."
tags: ["litex", "litepcie", "pcie", "fpga", "soft-phy", "xilinx", "kintex-ultrascale-plus", "ecp5", "liteiclink"]
categories: ["hardware"]
showHero: false
---

*TL;DR: [LitePCIe](https://github.com/enjoy-digital/litepcie) has always used the PCIe hard blocks of
the FPGA vendors. Its DMA, MMAP and MSI cores and its Linux driver run on top of what the hard block
provides. FPGAs without such a block could not use it, and people have been asking for a soft PHY
since 2019, starting with the Lattice ECP5 and its open toolchain. The soft PHY now exists. It puts
the LTSSM, the Physical Layer, the Data Link Layer and the configuration space in the fabric, on top
of the FPGA transceivers. It enumerates with the unmodified LitePCIe driver and runs DMA from 2.5 GT/s
x1 to 8 GT/s x8 on Kintex-7 and Kintex UltraScale+ boards. A few days after it landed, Dirk Eibach
brought it to the ECP5 Versa. At Gen3 x8 it also makes a point we already knew: on large
configurations it fills a good part of an XCKU3P, where the hard block costs almost nothing.*

{{< figure src="img/device-view.png" alt="Two device views of the Kintex UltraScale+ XCKU3P side by side. On the left, the soft PHY Gen3 x8 endpoint covers most of the device: the PHY logic in 48% of the CLBs and the rest of the design in 9%. On the right, the LitePCIe endpoint on the PCIe hard block uses a small region next to the hard block and the transceivers: 6% of the CLBs for the PHY wrapper and 5% for the rest." caption="A LitePCIe Gen3 x8 endpoint on the XCKU3P, placed and routed by Vivado: on the soft PHY (left) and on the PCIe hard block (right). Generated from the placed designs." >}}

## A request since 2019

The request is [issue #20](https://github.com/enjoy-digital/litepcie/issues/20), opened in May 2019 by
Tim Ansell: support LitePCIe on the Lattice ECP5 SerDes, with the open toolchain. The ECP5 has no PCIe
hard block, only transceivers. whitequark had already shown the ECP5 SerDes locking on a PCIe
stream with [Yumewatari](https://github.com/whitequark/Yumewatari), and others started their own
attempts. Over the years, more requests followed for other targets: a Xilinx soft PHY
([#83](https://github.com/enjoy-digital/litepcie/issues/83)), GateMate
([#151](https://github.com/enjoy-digital/litepcie/issues/151)), an ECP5 contribution whose first look
estimated a "multi-quarter scope" ([#164](https://github.com/enjoy-digital/litepcie/issues/164)),
and a soft Data Link Layer for devices without a hard block
([#165](https://github.com/enjoy-digital/litepcie/issues/165)).

We closed #20 once, in 2021, as "not realistic for now". The reason was economic, not technical.
A PCIe endpoint without a hard block needs:

- **the Physical Layer**: 8b/10b, scrambling, Ordered Sets, lane deskew, clock compensation, and
  128b/130b at 8 GT/s;
- **the LTSSM**, the link training state machine: Detect, Polling, Configuration, L0, Recovery, speed
  changes and, at 8 GT/s, transmitter equalization;
- **the Data Link Layer**: sequence numbers, LCRC, ACK/NAK, a retry buffer that replays lost packets,
  and flow control credits;
- **the configuration space** that the host reads at enumeration.

Each of them follows a large specification closely, and a host is not forgiving: a wrong field or
a missed timeout ends in a link that never trains, or in a machine that hangs at boot. That is
months of work, and it only pays off if someone needs it badly enough to fund it. In January 2025 the
issue had a bounty pool of 11,400 EUR, with 2,000 EUR from Enjoy-Digital, against whitequark's estimate of 8,000 EUR, which we already
considered on the low side for the work. It was not enough to make the work happen.

Others kept working on the ECP5 side in the meantime. In April 2026, zyp published
[katsuo-pcie](https://github.com/zyp/litepcie-katsuo-poc), a proof of concept of LitePCIe on an ECP5,
built on a separate PCIe stack with ECP5 support that zyp had been working on for a while.

It is the same story as [USB3-PIPE](/posts/usb3-pipe/): a protocol stack that follows a
specification closely is a long task that never fit in the funding available. With AI in the loop,
it became tractable.

## The soft PHY

`SoftPCIEPHY` is a drop-in replacement for the hard block PHYs in LitePCIe. The endpoint, the DMA,
the MMAP bridges, MSI and the Linux driver see the same interface, so a design moves from a hard block
to the soft PHY by changing its PHY. Below that interface, everything is in the fabric:

- **Transaction Layer glue**: TLP framing, a Type 0 configuration space with the PCI Express, MSI and
  power management capabilities, completions for configuration requests, Unsupported Requests;
- **Data Link Layer**: DLLPs, LCRC, the retry buffer and flow control, at x1 and in a wide version
  that handles one DW per lane per cycle at x2 to x8;
- **LTSSM**: the training state machine, upstream and downstream roles, speed changes up to 8 GT/s
  and equalization;
- **Physical Layer**: per-lane datapaths, Training Sequences, SKP insertion and removal, striping
  and deskew;
- **transceivers**: through LiteICLink, with adapters for the Artix-7 GTP, Kintex-7 GTX and
  UltraScale+ GTY.

The LTSSM uses the Xilinx UltraScale state encoding, so the LTSSM tracer and tools that LitePCIe
already had keep working. The approach is the one of USB3-PIPE: the transceivers do the
serialization, and the protocol runs in the fabric.

## A week, simulation first

The first commit landed on 6 November, and Gen3 x8 ran on hardware on 10 November. The order matters
more than the speed:

1. **Building blocks with reference models**: the scrambler, the LCRC and DLLP CRC, Training
   Sequences, each checked against Python models (the LCRC against `zlib.crc32`).
2. **A link in simulation**: two Physical Layers in Verilator, through transceiver models with ppm
   offsets, polarity inversion and symbol shifts, training to L0 and exchanging packets. Then the
   Data Link Layers on top, with corrupted words and lost DLLPs: every TLP delivered once, in order.
3. **An endpoint against a host model** that follows the Linux driver's register sequence: CSRs over
   BAR0, DMA tables, loopback through host memory, MSIs.
4. **Hardware**, with the LTSSM disabled at reset and by PERST#, so that a design that hangs the host
   cannot do it again at the next boot.

On a YPCB-00338-1P1 (Kintex-7 XC7K480T, GTX), the first hardware run enumerated, loaded the LitePCIe
driver and passed the register, MSI and DMA tests on the same day. Gen2 and x2 to x8 followed the
next day, 8 GT/s the day after in simulation, and then the Kintex UltraScale+ GTY on the
[Alibaba Cloud XCKU3P card](/posts/usb3-pipe-gen2/).

{{< figure src="img/boards.jpg" alt="Left: the Inspur YPCB-00338 Kintex-7 card with its heatsink and blower fan, installed in the PCIe slot of a small PC. Right: the Alibaba Cloud AS02MC04 card without its shroud, with the Kintex UltraScale+ XCKU3P, two SFP28 cages and a PCIe x8 edge connector." caption="The two test boards. Left: the YPCB-00338-1P1 (Kintex-7 XC7K480T, GTX) in its host, for Gen1 and Gen2 up to x8. Right: the Alibaba Cloud XCKU3P card (Kintex UltraScale+, GTY), for Gen1 to Gen3 up to x8. Photo right: [TiferKing/as02mc04_hack](https://github.com/TiferKing/as02mc04_hack) (MIT)." >}}

The hardware found what the simulation could not. A TLP packetizer bug lost the last beat of a 4 DW
Memory Write followed by a 3 DW Completion: it was not in the soft PHY but in LitePCIe itself, and
it is fixed for every PHY. MSIs starved under sustained DMA until they got priority. On the GTX at
8 GT/s, the receiver reported Electrical Idle on a live signal and squelched the data. The eye from
the host was closed while the near-end loopback was clean, which points to the channel of that
board at 8 GT/s. The same code trained at 8 GT/s on the GTY.

## Results

On the Kintex-7 GTX (YPCB-00338-1P1), against the hard block of the same board, with the LitePCIe
driver's `dma_test` loopback over 15 minutes:

| Kintex-7 GTX | Soft 5 GT/s x1 | Soft 5 GT/s x4 | Soft 5 GT/s x8 | Hard IP 5 GT/s x8 |
|---|---:|---:|---:|---:|
| `dma_test` | 3.39 Gb/s | 11.57 Gb/s | 14.97 Gb/s | 13.3 Gb/s |
| Correctable errors at the root port | 0 | 0 | 0 | 0 |
| Link cycles (remove, retrain, rescan, test) | 20/20 | 20/20 | 20/20 | |
| Slice LUTs, whole design | 8,617¹ | 20,095 | 41,553 | 8,407 |

¹ 2.5 GT/s x1 build.

Above about 13 Gb/s, `dma_test` itself does not keep up: its software check loses whole buffers on
both PHYs. The link also negotiates down from x8 to x4, x2 and x1 when lanes are kept out.

On the Kintex UltraScale+ GTY (XCKU3P), the soft PHY trains at 2.5 GT/s, and the root port moves it to
8 GT/s with the equalization phases 0 to 3. At 8 GT/s x8, a test program on the driver's library
received 23.8 Gb/s of DMA data and checked it: 1.78M buffers in 5 seconds, none wrong. Retrain Link
from 2.5 or 5 GT/s back to 8 GT/s worked 60 times out of 60.

None of these numbers is a maximum. We have not tried to optimize the throughput yet: the goal of this
first version was a link that trains, enumerates and moves data reliably at every speed and width.
At 8 GT/s x8, 23.8 Gb/s is well below the 63 Gb/s the link can carry, and at 5 GT/s x8 we have not
identified what limits it to about 18 Gb/s. Finding and removing these limits comes next.

## Gen3 x8 and the XCKU3P

Width and speed have a cost in the fabric. At 8 GT/s x8, the link carries 64 Gb/s, which the soft PHY
handles at 125 MHz on 512-bit beats. Every lane needs its own 128b/130b blocks, descrambler and
equalization. The Data Link Layer has to find packet boundaries anywhere in the beat, compute an LCRC
over the whole beat, and repack TLPs that start in the middle of it.

The first Gen3 x8 synthesis needed **202K LUTs, 124% of the XCKU3P**. A round of area work brought it
into the device: the TLP repacker writes into banks and rotates on reads, the LCRC is computed over the
whole beat and rewound, the 8 GT/s framer works on pairs of slots, and the 8b/10b framer runs at half
the width on 8 GT/s capable links.

| XCKU3P (162,720 LUTs), Vivado 2024.1, timing met | CLB LUTs | CLBs used | Registers | Block RAM |
|---|---:|---:|---:|---:|
| Soft PHY Gen3 x8, first build | 202K (synthesis) | did not fit | | |
| Soft PHY Gen3 x8, after the first area changes | 111,153 (68%) | 78% | | |
| Soft PHY Gen3 x8, current | 82,434 (51%) | 57% | 69,610 (21%) | 57 (16%) |
| Hard IP Gen3 x8 (`PCIE40E4`), LitePCIe endpoint from the LiteX-Boards target | 13,213 (8%) | 11% | 17,287 (5%) | 45.5 (13%) |

The device view at the top of this post shows what that means. It is drawn from the placed designs:
one rectangle per CLB, blue where the PCIe PHY logic is placed. On the soft PHY, the PHY alone takes
48% of the CLBs, and the endpoint, DMA and SoC another 9%. On the hard block, the PHY wrapper takes 6%,
and the rest of the design 5%, even with the VexRiscv CPU of the LiteX-Boards target, all next to the
`PCIE40E4` and the transceivers.

Our implementation is not optimal yet. The area work so far took a design that did not fit down to
half of the LUTs in two passes, and we will probably optimize it further. But even optimized, a soft
PHY for large configurations will stay much more expensive in resources than the hard block. The hard
block does the same work in silicon, at no cost in LUTs, with its timing already closed. For large
configurations on devices that have one, the hard block stays the right choice.

The soft PHY is for everything else: FPGAs with transceivers but no PCIe block (ECP5, GateMate,
smaller Gowin parts), devices that need more PCIe ports than they have hard blocks, Gen1 and Gen2
links at x1 to x4, where the cost is moderate (about 8.5K LUTs at x1 on an Artix-7, against 4.5K
on the hard block), and work where seeing every layer of the link matters more than its area.

## The ECP5, the original request

The soft PHY is recent, and it has already been tested on the ECP5, the target of the 2019 request.
In April, Dirk Eibach wrote on #20 that it was time to dust off his Versa PCIe test rig. In November,
a few days after the soft PHY landed, he did, and in his words, "things escalated quickly". The port
could also build on the earlier ECP5 PCIe work: Yumewatari had worked out how to put the DCU in PCIe
mode and detect a receiver, and katsuo-pcie had shown LitePCIe running on the Versa.

{{< figure src="img/versa-board.jpg" alt="The Lattice ECP5 Versa development board on a wooden table, with its status LEDs lit, next to a potted red gerbera" caption="The Lattice ECP5 Versa on Dirk Eibach's testbed. Photo: Dirk Eibach, [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/), from [ecp5-pcie-core](https://github.com/eibach/ecp5-pcie-core)." >}}

The port uses our soft PHY as it is: everything above the transceiver, from the Physical Layer to
the configuration space, is the same code as on the Xilinx boards. He added an ECP5 lane adapter
below it: the DCU in PCIe mode through LiteICLink, the receiver detection sequence, clocking, a
behavioral DCU model for simulation and a Versa bench. For the DCU side, the PCIe-mode attributes and
the receiver detection sequence follow Yumewatari and Lattice's documentation, and katsuo-pcie
confirmed them. No code was taken from either. Like ours, much of the work was done with an AI agent (Claude
Code), under his supervision. On the Versa (LFE5UM-45F, the non-5G part), in a Fujitsu FUTRO S920
with the unmodified LitePCIe driver:

| Lattice ECP5 Versa, Gen1 x1, open toolchain | |
|---|---|
| Enumeration | at boot |
| MSI | OK |
| `dma_test` loopback | 1.66 Gb/s, 0 errors |
| 15 minute DMA soak | 22.8M buffers, 0 errors, no replay, no receiver error |
| BAR0 stress, 15 minutes | 106.5M writes with readback, 0 mismatches |
| Link cycles (unload, remove, retrain, rescan, load, test) | 20 of 20 |
| Resources, whole SoC with DMA and analyzer | 24,400 LUT4 (55%), 10,793 FFs, 56 of 108 DP16KD |

It is split into small pull requests:
[LiteICLink #111](https://github.com/enjoy-digital/liteiclink/pull/111) for the PCIe option of the ECP5
SerDes, then [LitePCIe #250](https://github.com/enjoy-digital/litepcie/pull/250) for the adapter, model
and bench, with two simulation PRs ([#248](https://github.com/enjoy-digital/litepcie/pull/248),
[#249](https://github.com/enjoy-digital/litepcie/pull/249)). They are under review. Everything around
them (bring-up runbook, host tools, hardware log and a pinned build recipe that regenerates the tested
design) is in [eibach/ecp5-pcie-core](https://github.com/eibach/ecp5-pcie-core), so the results can be
reproduced.

That is what we hoped for when we started: a core that others can take to their own hardware, in
days rather than quarters.

## What is not done yet

- The Artix-7 GTP, the first target on paper, has not run on hardware yet.
- No lane reversal, no L0s/L1 power management or ASPM, no MSI-X.
- 8 GT/s at the full width only: width negotiation is done at 2.5 and 5 GT/s.
- On the GTX, 8 GT/s with a host is still open on the board we used.
- Throughput: not optimized yet, see the results above.
- More area work at large widths, and other transceivers: GateMate and Gowin are on the list.

## Try it

```sh
git clone https://github.com/enjoy-digital/litepcie && cd litepcie

# Simulation: two Physical Layers at 8 GT/s x8, equalization and a Link Retrain.
python3 bench/soft/sim_wide.py --phy --gen3 --nlanes 8 --eq-preset 8
# A LitePCIe SoC with DMA against a host model.
python3 bench/soft/sim_soc.py --gen3 --nlanes 8

# Hardware: the full endpoint on the Alibaba Cloud XCKU3P card at 8 GT/s x8.
python3 bench/soft/ku3p.py --with-endpoint --speed gen3 --nlanes 8 --driver --build
```

The design, status and hardware results are in `doc/softphy.md`, and the soft PHY chapter of the
LitePCIe manual.

---

Thanks to Tim Ansell for the original request, to everyone who pledged on #20, to whitequark
(Yumewatari) and zyp (katsuo-pcie) for their ECP5 PCIe work, and to Dirk Eibach for the ECP5 port and
the Versa photo.

*Work and ideas by Enjoy-Digital; written up with AI in the loop.*
