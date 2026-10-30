---
title: "TriXium, part 2: a language model on an Efinix Ti375, from power-up to chat in 20 seconds"
date: 2026-10-30T09:15:00+02:00
draft: false
description: "TriXium on the Efinix Ti375 C529 kit: the ternary Bonsai 2 2B model decoding at 14 tok/s from 1 GB of LPDDR4x, first with the weights sent over UDP from a host, then standalone with the model and its tokenizer in the eMMC. Power it up and about 20 seconds later the board answers on the network with its own chat page, on a few watts. With an int8 KV cache that keeps the speed flat up to 2,000 tokens of context."
summary: "Part 2 of TriXium: moving from PCIe cards to the small Efinix Ti375 kit. LPDDR4x at its full rate through both AXI targets, the weights over UDP, then a standalone board with the model and the tokenizer in eMMC, a chat page served by the firmware, and an int8 attention cache."
tags: ["litex", "fpga", "efinix", "titanium", "ti375", "lpddr4x", "liteeth", "llm", "ternary", "trixium"]
categories: ["hardware"]
showHero: false
---

*TL;DR: After the PCIe cards of [part 1](/posts/trixium/), TriXium moved to a small board: the
Efinix Titanium Ti375 C529 development kit, with 1 GB of LPDDR4x behind the FPGA's hard memory
controller and no host interface faster than Ethernet. The first goal was to get the core running
on the Efinix fabric and the LPDDR4x at its full rate, using both of the controller's AXI targets.
The Ternary-Bonsai-2 2B model then ran with its weights sent over UDP from a host, at **8.9 tok/s**
on the first bitstream and **14.3 tok/s** two days later. Then the host went away: the bitstream
and firmware boot from the flash, the model and its tokenizer load from the kit's eMMC, and the
firmware serves its own chat page on the network. You switch the board on, and **about 20 seconds
later** `http://trixium.local/` answers, on a board that draws a few watts. An int8 attention
cache, added last, keeps the decode speed nearly flat: **13.8 tok/s at 2,040 tokens of context**,
against 10.1 before.*

{{< video src="boot.mp4" poster="img/boot-board.jpg" controls="true" muted="true" playsinline="true" preload="metadata" >}}

*The measured power-up, cut to 42 s: the Ti375 kit configures itself from its flash, streams the 2B
model and its tokenizer from the eMMC, and serves the chat page. No GPU, no cloud, no PC.*

## Why a small board, and why this one

[Part 1](/posts/trixium/) ended on a projection. Batch-1 decode is set by the memory, so the speed
of a smaller home is easy to estimate: about 78% of its bandwidth divided by the bytes streamed per
token, the ratio measured on the YPCB-00338. That makes boards with fast memory and no PCIe card
around them interesting, and the Ti375 kit is a good one: a Titanium Ti375, a fabric that runs
designs like ours at 100 MHz with margin, and an LPDDR4x x32 device with 12.8 GB/s of peak
bandwidth behind a hard controller. That is three quarters of the YPCB-00338's two DDR3 channels, on a
development board.

{{< figure src="img/smaller-homes.jpg" alt="Slide titled Same blocks, smaller homes: the FK33, XCU1525 and YPCB-00338 measured, and projections for an Efinix Ti375 and an Artix-7 200T board" caption="Same blocks, smaller homes: once the design was memory-bound on the large cards, the speed on a smaller board could be projected from its memory alone." >}}

The model is the Ternary-Bonsai-2 2B, a ternary version of Qwen3.5-2B: the newer Bonsai
generation, a hybrid of Gated DeltaNet and attention layers. It streams about 0.41 GB of weights
per token, small enough for the 1 GB of LPDDR4x with its KV cache and recurrent state. How the
design moved from the first Bonsai models to these hybrids is [part 3](/posts/trixium-bonsai2/);
here it is enough to know that the core already ran it on the YPCB-00338.

## Getting the core onto the Efinix fabric

The first step was in the simulator, before Efinity ever ran. The Ti375's memory system is very
different from the PCIe cards': no LiteDRAM, but one hard controller with 512-bit AXI4 ports,
shared by the CPU and four engine channels. A `--board ti375` preset of the Verilator simulation
modelled exactly that, and it found a deadlock and a stall in our shared AXI port before any
hardware did (a cross-master write dependency, and head-of-line read blocking).

Then Efinity, and the first synthesis did not fit: 457k LUT4 and 410k flip-flops for a part with
about 370k logic elements, 123% of the logic. Two things explain most of it:

- **No distributed RAM.** Efinix has no LUT RAM, so the attention unit's small asynchronous-read
  operand memories became 131k flip-flops, half the design. Reading them synchronously, with the
  word select registered alongside (the read data was registered in the next cycle anyway),
  removed them with the same timing.
- **RAM port forms.** Efinity maps a true dual-port RAM only when the write is nested in the read
  enable, a form Migen does not emit. The Hadamard unit's banks got a port form Efinity maps,
  checked bit-exact.

With the lite profile of units, four engine channels and the DeltaNet stream at half width, the
design placed at 75% of the logic and met timing at 100 MHz. As the work went on it grew to about
95% of the logic with every useful unit back in, and still closed at 100 MHz with 105 to 110 MHz
achievable.

The kit then surfaced the things no model knew about:

- **The hard controller's AXI IDs are 6 bits**, while LiteX declared 8. The shared port put the
  master index in bit 6, which was dropped, and the CPU's first read came back routed to engine
  channel 0: fixed in LiteX ([#2685](https://github.com/enjoy-digital/litex/pull/2685)) and
  litex-boards ([#931](https://github.com/litex-hub/litex-boards/pull/931)).
- **The controller waits for WVALID before AWREADY**, which AXI allows, and our port sent the write
  data only after the address was accepted: the BIOS stopped in the middle of its banner. The port
  now sends a buffered burst's data first, and a test runs every slave write order.
- **Reads can come back out of order** across IDs, so each channel reader uses one ID.

On 21 October, the 2B gave the reference tokens on the kit, at **8.86 tok/s**, with everything at
100 MHz and the weights uploaded over Ethernet.

## LPDDR4x at its full rate

That first bitstream used one of the controller's two 512-bit AXI targets. At 100 MHz, one target
carries 6.4 GB/s, half of the LPDDR4x's 12.8 GB/s, and the weight streams used 92% of it: two thirds
of each token waited on that port. The controller has a second target, so LiteX's `EfinixDDR`
gained the option to use both ([#2688](https://github.com/enjoy-digital/litex/pull/2688)), with
two engine channels and the CPU on the first and two channels on the second. And LiteX's
`AXIArbiter` gained an interleaving mode
([#2689](https://github.com/enjoy-digital/litex/pull/2689)): it used to hold a master until all
its requests were answered, which serialised the channel readers on a shared port.

The streams went from 0.46 to 0.75 beats per channel per clock: **11.94 tok/s**. Then a smaller
effect that every multi-stream LPDDR design should know about. The four channels advance in step,
and with windows a power of two apart they opened rows in the same LPDDR4x banks. Starting each
channel's window 8 KB further than the previous one broke the aliasing (anything under 16 KB
works; 8 KB is the middle of the plateau): 0.78 beats per channel per clock, about 10 GB/s of
weights out of the 12.8 GB/s peak.

The rest of the climb was the usual measure-and-move work, each step a build on the kit:

| build | change | 2B tok/s at 64 / 256 / 1,024 tokens |
| --- | --- | --- |
| v7 (21 Oct) | one AXI target, first golden tokens | 8.86 / 8.05 / 5.41 |
| v8 | both AXI targets | 11.94 / 10.46 / - |
| v9 | DeltaNet stream at full width, embedding restore on the Hadamard unit | 13.28 / 11.45 / 6.75 |
| v11 | channel windows skewed by 8 KB across the banks | 13.63 / 11.70 / 6.84 |
| v15 | the attention softmax unit (the CPU's softmax was half the attention at 1,024) | 14.08 / 13.20 / 9.31 |
| v18 | attention stream at 8 lanes: both KV heads in one command | 14.33 / 13.97 / 10.93 |
| v19 | attention stream to 2,048 positions, timing margin | 14.32 / 13.96 / 11.97 |

{{< figure src="img/demo-dashboard.jpg" alt="Screen recording: on the left the trixium_chat terminal with a reply being generated in French and the trixium_smi status below it showing 6.14 GB/s of 12.8 GB/s port peak on four channels; on the right the TriXium live dashboard in a browser showing 13.7 tok/s decode, 77% engine busy, 6.06 GB/s DDR read, a per-stage breakdown of a token's cycles and a grid of the ternary engine's activity" caption="The kit at work with a host: trixium_chat and trixium_smi on the left, the live dashboard on the right (decode rate, DDR read rate against the port peak, where a token's cycles go)." >}}

## Weights over UDP, then a host that only watches

Without PCIe, the weights first came over Ethernet. A small gateware block, `UDPUpload`, takes the
datagrams of a LiteEth UDP port and forwards one to the weight loader only when its sequence number
is the expected one; gaps, duplicates and partial datagrams are dropped and counted, and the host
tool resends from the expected number. Each transfer is checked against a CRC-32 as it lands. The
2B image, 452 MB, uploads in **4.8 s**, about 95 MB/s over the kit's gigabit port.

So the first demo was: load the bitstream over JTAG, upload the weights, and chat from the host.
`trixium deploy --board ti375` does all of it in about a minute, `trixium chat`, `console` and
`smi` work against the kit (the console over its UART, the status over Etherbone), and
`trixium dashboard` serves a live page with the decode rate, the DDR traffic and where a token's
cycles go.

At that point the host was still doing one real job: the tokenizer.

## The tokenizer moves onto the board

A language model works on token ids, not text. The host's job was to turn the prompt into ids and
the ids back into text, and the 2B's Qwen3.5 tokenizer is not small: a 248,320-token vocabulary
with its merge rules. To make the board standalone, the tokenizer had to move into the firmware.

We wrote one encoder in C, `trixium_tok`, shared by the host's `trixium_chat` and the firmware, and
checked it against the reference encoder's ids on 614 paragraphs (our docs, multilingual text and
edge cases). Its tables (5.8 MB for the 2B) travel with the model; at boot the firmware copies them
to the top of main RAM, builds its index (14.9 MB in all), and checks their hash against the
model's. From then on, any serial terminal is a complete client: `chat` on the console, type text,
read text, the conversation continuing from the KV cache turn by turn.

## Standalone: the model in the eMMC

The weights still came over Ethernet at each power-up. The kit has a microSD slot and an 8 GB eMMC,
so the next step was to store the model on the board.

The storage path uses LiteSDCard's PHY and core, with the card's read stream widened into the same
weight loader as the UDP upload: the firmware arms the loader for a channel window, a multi-block
read lands directly in the LPDDR4x, and the data never goes through the CPU. A CRC-32 of each entry
is computed in the gateware on the bytes going to the loader and checked against the image's table,
so a corrupted block stops the boot instead of producing a confused model.

The microSD slot worked first, but only in 1-bit mode on our kit (one data line never goes low,
somewhere between the FPGA ball and the card): 436 MB in 74 s. The eMMC then took over: LiteX's
SD card library learned eMMC (CMD1 init, 8-bit bus, high speed,
[#2695](https://github.com/enjoy-digital/litex/pull/2695)), and the kit's litex-boards target got
it as its SD card ([#933](https://github.com/litex-hub/litex-boards/pull/933)). The 2B model and
its tokenizer now load in **9.6 s**, 46 MB/s, close to the eMMC's cap at 8 bits and high speed. The
firmware writes the eMMC itself from a UDP upload (`ti375 model <img> --persist`), and the eMMC
holds up to seven models in slots, one of them selected for power-up.

{{< figure src="img/board-ti375.png" alt="Block diagram of TriXium on the Efinix Ti375 C529, standalone: QSPI flash for the bitstream and firmware, eMMC for the model and tokenizer, LiteEth with the firmware's TCP/IP and HTTP chat server, the VexiiRiscv CPU, the TriXium core with sequencer, matvec engine, units and scratchpad, an interleaving AXI arbiter, and the hard LPDDR4x controller with two 512-bit AXI targets to 1 GB of LPDDR4x" caption="TriXium on the Ti375, standalone: bitstream and firmware from the flash, model and tokenizer from the eMMC, LiteEth and the firmware's own HTTP server, four engine channels and the CPU on the hard LPDDR4x controller's two AXI targets." >}}

## A chat page served by the firmware

The last piece was the network. The SoC keeps LiteX's hardware Etherbone stack for the debug tools
and adds a CPU MAC beside it (unicasts for the hardware stack stay in hardware, broadcasts reach
both). On the CPU side, the firmware has a small TCP/IP stack of its own, written for this: ARP,
ICMP, DHCP with a static fallback, mDNS so the board answers as `trixium.local`, TCP for four
connections, and an HTTP server with the chat page and a streaming `POST /chat`. A request runs
from the main loop, so a reply streaming for a minute keeps the stack polled, and the console and
the page share one conversation.

{{< figure src="img/boot-chat.jpg" alt="The TriXium chat web page in a browser: the prompt Write a haiku about a tiny computer that thinks, and the reply Tiny chip hums, It reads the screen, It thinks and waits" caption="The page, the tokenizer and the model, all on the board. The first words of a reply arrive after about 1.5 s, then 12.3 tok/s in Chrome." >}}

We measured the power-up with a script timestamping the console, from the moment the kit's USB
bridge appears on the workstation:

| time | event |
| ---: | --- |
| 4.9 s | the FPGA has configured itself from the QSPI flash, the LiteX BIOS runs |
| 5.7 s | the firmware, copied from the flash, starts; the eMMC answers at 5.8 s |
| 6.9 s to 16.4 s | the 2B model from the eMMC: 436 MB in 9.6 s, every entry CRC-checked |
| 17.4 s | tokenizer loaded, model header parsed |
| 18.2 s | self-tests done, `TRIXIUM-READY`, the web server up, a DHCP lease 0.1 s later |
| 19.4 s | the first `GET /` answered |

About 20 seconds from the switch to the page, with about 2 seconds of slack still in it (a key
wait, a boot timeout, a memtest, the boot demo). It is honestly a lot of fun: you plug in power and
an Ethernet cable, wait the time it takes to sit down, open `http://trixium.local/` in a browser,
and a language model running entirely on the FPGA answers, on a board that draws a few watts. We
estimate 3 to 5 W for the kit; the dashboard takes the reading of a USB-C meter on the 12 V input,
which is the next measurement to publish.

{{< figure src="img/boot-emmc.jpg" alt="Frame from the boot video: the standalone block diagram with the eMMC, loader and LPDDR4x highlighted, next to a timer at T+ 11.3 s and the measured boot steps, with the caption The 2B model streams from the eMMC: 436 MB, 46 MB/s into the LPDDR4x, every entry CRC-checked" caption="Most of the 20 seconds is the model streaming from the eMMC into the LPDDR4x." >}}

Conversations also go past the 2,048-token attention window without a restart. The DeltaNet layers'
recurrent state carries the whole history, and the attention layers keep the conversation's first
turn plus a ring of the latest positions. In a five-turn story test of about 4,000 tokens, asking
at the end for the name of a character introduced in the first turn, the board answered correctly.

## int8 attention: a flat decode speed

One thing was still visible in the table above: the speed fell with the context, from 14.3 tok/s at
64 tokens to 10.1 at 2,040. The KV cache rows were FP32, and on the Ti375 long-context attention
turned out to be compute-bound rather than bandwidth-bound: 8 FP32 lanes, 512 cycles per position
and layer for the scores and the values. Storing int8 would only help with an integer datapath to
go with it.

So we checked the quality first, then built the unit. On the 2B with wikitext-2 (4 x 2,048 tokens),
int8 K/V rows with one power-of-two exponent per head row, using the unit's exact arithmetic, gave a
perplexity of 27.880 against 27.882 in FP32: no measurable loss. The new attention unit computes
eight exact int8 products per lane per clock on the DSPs, sums them exactly in integers, and rounds
once to FP32 per output: the scores and the values take 32 clocks per position instead of 256 on the
2B. A KV row shrinks from 64 to 17 memory beats, and the whole design is slightly smaller than the
FP32 one in LUTs, for 30 more DSPs.

{{< figure src="img/int8-attention.svg" alt="Grouped bar chart of the 2B decode speed on the Ti375 at 64, 256, 1,024 and 2,040 tokens of context: FP32 attention 14.32, 13.96, 11.98 and 10.05 tok/s; int8 attention 14.42, 14.59, 14.27 and 13.81 tok/s; gains of 0.7%, 4.5%, 19% and 37%" caption="int8 attention on the kit: the same speed at short context, and +37% at 2,040 tokens. The decode speed is now nearly flat with the context." >}}

On the kit it gives the mirror's int8 tokens exactly, and the decode speed is nearly flat with the
context: 7.24 M cycles per token over 2,040 tokens, against 9.95 M. It is in the kit's flash now.
That bring-up also had a good lesson: the first int8 bitstream ran coherent text but not the
reference tokens, and it bisected to a LiteX change of the day before. The LiteX change was right.
It made the Verilog follow Migen's width rules exactly, and one of our expressions in the softmax
unit's exp was 33 bits under those rules, so its sign bit fell off. The old Verilog had hidden our
bug; our unit tests now convert with LiteX's converter, as the SoC does.

## What went into LiteX for this board

Small boards are where LiteX gets exercised in new places. What came out of this one:

| PR | What |
| --- | --- |
| [LiteX #2684](https://github.com/enjoy-digital/litex/pull/2684) | Etherbone with the PHY's clock domains at every data width (the Titanium RGMII PHY) |
| [LiteX #2685](https://github.com/enjoy-digital/litex/pull/2685), [litex-boards #931](https://github.com/litex-hub/litex-boards/pull/931) | the Efinix hard DDR controller's 6-bit AXI IDs |
| [LiteX #2688](https://github.com/enjoy-digital/litex/pull/2688) | `EfinixDDR(targets=2)`: the controller's second AXI target, the full LPDDR4x rate |
| [LiteX #2689](https://github.com/enjoy-digital/litex/pull/2689) | `AXIArbiter(interleave=True)`: per-request reads, buffered write bursts |
| [LiteX #2690](https://github.com/enjoy-digital/litex/pull/2690) | crt0: the SMP lottery words aligned (the BIOS trapped before its banner from flash) |
| [LiteX #2694](https://github.com/enjoy-digital/litex/pull/2694), [#2695](https://github.com/enjoy-digital/litex/pull/2695), [litex-boards #933](https://github.com/litex-hub/litex-boards/pull/933) | SD card reads without DMA, eMMC support, the kit's eMMC in the BIOS and Linux |

## Try it

With the kit on the network and `[ti375]` in `~/.config/trixium/boards.ini`, from the
[repository](https://github.com/enjoy-digital/trixium):

```bash
trixium deploy --board ti375 --model 2b                 # bitstream, firmware, weights over UDP, tokens, bench
python3 -m software.host.ti375 model models/bons/bonsai2-2b-n4-ti375.img \
    --tokenizer models/2b/tokenizer.bin --persist       # store the model and tokenizer in the eMMC
python3 -m software.host.ti375 flash <build> <firmware> # bitstream and firmware in the QSPI flash
```

Then power-cycle it, and open `http://trixium.local/`, or `picocom -b 115200` on its console and
type `chat`.

What we find most interesting is where this points. The next step down is logic and memory in one
package: Titanium devices with HyperRAM inside, a much smaller model and its state, a few hundred
milliwatts. Not a chat assistant, but a small block of language-model intelligence next to a
sensor, a radio or a camera pipeline.

---

*TriXium runs [PrismML](https://prismml.com/news/ternary-bonsai)'s Ternary-Bonsai-2 models (on
Qwen's Qwen3.5 architecture). Built on [LiteX](https://github.com/enjoy-digital/litex),
[LiteEth](https://github.com/enjoy-digital/liteeth),
[LiteSDCard](https://github.com/enjoy-digital/litesdcard),
[LiteX-Boards](https://github.com/litex-hub/litex-boards) and
[VexiiRiscv](https://github.com/SpinalHDL/VexiiRiscv); the Ti375 bitstreams are built with Efinix's
Efinity. Previous part: [TriXium on the PCIe cards](/posts/trixium/). Next:
[from Bonsai to Bonsai 2](/posts/trixium-bonsai2/).*

*Work and ideas by Enjoy-Digital; written up with AI in the loop.*
