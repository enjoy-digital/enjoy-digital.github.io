---
title: "TriXium, part 3: from Bonsai to Bonsai 2, porting an FPGA LLM design to a hybrid architecture"
date: 2026-10-30T09:30:00+02:00
draft: false
description: "TriXium was built around PrismML's first ternary Bonsai models (Qwen3). The Bonsai 2 models are a different architecture: a Qwen3.5 hybrid where three layers out of four are Gated DeltaNet recurrences. This is how the design followed with little effort: what carried over unchanged, what had to be added (a DeltaNet stream, an attention stream, a Hadamard unit and others), in which order, and the results from the 27B on an XCU1525 to the 2B on a $100 Kintex-7 card and an Efinix Ti375."
summary: "Part 3 of TriXium: porting a design built for the Qwen3-based Bonsai models to the Qwen3.5 hybrid Bonsai 2 family. The CPU path first, token-exact in two days, then one fixed-function unit at a time, and the 2B and 4B on the small boards two days after their release."
tags: ["litex", "fpga", "llm", "ternary", "deltanet", "qwen3.5", "bonsai", "trixium"]
categories: ["hardware"]
showHero: false
---

*TL;DR: [Part 1](/posts/trixium/) and [part 2](/posts/trixium-ti375/) showed TriXium on PCIe
cards and on a small Efinix board. This one is about models. TriXium was designed around
PrismML's first ternary Bonsai models, 1.7B and 8B on the Qwen3 architecture: a classic
transformer. The Bonsai 2 family is built on Qwen3.5, a hybrid where three layers out of four are
not attention at all but a Gated DeltaNet recurrence, with a state per head updated at every token.
Because the control was in firmware and the engine was a generic weight streamer, the 27B ran
token-exact on an XCU1525 two days after its golden model, with no new gateware, at 0.04 tok/s. Eight
days of moving the hot paths into units took it to **3.58 tok/s**, and **4.03** with the DDR4
overclock. When PrismML released the 2B and 4B, they ran with the reference tokens on the
YPCB-00338 two days later, and reached **20.3** and **9.2 tok/s** on that ~$100 Kintex-7 card,
against 12.1 and 5.9 for openTPU running the original Qwen3.5 models at 4 bits on the same card.*

{{< figure src="img/models.jpg" alt="Slide titled Then 8B, then 27B: Ternary-Bonsai 8B at 30.3 tok/s on the FK33's HBM2 and Ternary-Bonsai-2 27B at 4.03 tok/s on the XCU1525's four DDR4 controllers" caption="Then 8B, then 27B: the same core, a new model architecture." >}}

## Two model generations

The first Bonsai models (1.7B, 8B) are Qwen3 with ternary weights: every layer is the familiar
pair of grouped-query attention and a SiLU-gated feed-forward network, with RMSNorm and RoPE. The
design of part 1 was built for that: an engine that streams ternary matrices from every memory
channel, a VexiiRiscv running the model loop in C, a small microcoded vector unit for the
elementwise work, and a few fixed-function units.

The Bonsai 2 models are Qwen3.5, and Qwen3.5 is a hybrid:

| | Bonsai 2 2B | Bonsai 2 4B | Bonsai 2 27B |
| --- | ---: | ---: | ---: |
| layers (DeltaNet + attention) | 24 (18 + 6) | 32 (24 + 8) | 64 (48 + 16) |
| hidden / FFN | 2,048 / 6,144 | 2,560 / 9,216 | 5,120 / 17,408 |
| attention heads / KV heads / head dim | 8 / 2 / 256 | 16 / 4 / 256 | 24 / 4 / 256 |
| DeltaNet value / key heads | 16 / 16 | 32 / 16 | 48 / 16 |
| Hadamard block | 512 | 512 | 1,024 |
| vocabulary | 248,320 | 248,320 | 248,320 |
| weights streamed per token (5-trit codes) | 0.41 GB | 0.91 GB | 5.5 GB |

Three things are new for the hardware:

- **Gated DeltaNet layers.** Three layers out of four replace attention with a linear-attention
  recurrence: each head keeps a 128x128 FP32 state, and every token updates it with the delta rule
  (decay the state, correct it towards the new key/value pair, read it with the query). Around it:
  a short causal convolution, L2 norms on q and k, small int8 projections for the gates, and a
  gated RMSNorm on the output. Unlike a KV cache, the state does not grow with the context, but it
  is read and written in full at every token: 18.9 MB per token on the 2B.
- **Gated attention.** The remaining attention layers have an output gate, per-head norms and RoPE
  on part of the head dimension only.
- **Hadamard rotations.** PrismML stores the weights of the Bonsai 2 models rotated: before a
  projection, the activation vector gets a sign flip and a block Hadamard transform, so the
  ternary weights see a better-conditioned input. Hadamard is additions and subtractions only, but
  it is one more pass over every activation, at every projection.

## What carried over unchanged

This is the part we are happiest with, because it was not luck. Most of the design did not need to
know the model had changed:

- **The engine and the memory system.** A projection is a ternary matrix-vector product whatever
  the layer type. The channels, the 5-trit codes, the readers, LiteDRAM, the HBM2 and DDR4 targets,
  all unchanged. The DeltaNet's small gate projections are int8, and the engine already had an int8
  (Q8) path.
- **The converter.** It reads the GGUF and copies the ternary codes and the scales bit for bit, so
  new tensor names and a new format variant (`PTQ1_0`) were converter work, not hardware work.
- **The control in firmware.** The model loop is C on the RISC-V. A new topology is a new loop,
  `firmware/hybrid.c`, written against the same primitives (post a descriptor list, run a
  projection, read a result).
- **The verification chain.** The golden model and the bit-exact mirror got the new layers, and the
  same simulation gate and board checks applied. The 27B's golden model reproduced PrismML's
  llama.cpp fork exactly before any hardware ran it.

So the first 27B on hardware used none of the new units, because there were none yet: the engine
streamed the projections, and the CPU did everything else in FP32. On 20 September, two days after
the golden model, the 27B produced the reference tokens on the XCU1525, at 0.036 tok/s. Slow, and
exactly right, which is the order we want.

{{< figure src="img/exact.jpg" alt="Slide titled Verified at every level: a software model with a golden numpy model and bit-exact Python mirror, unit tests with Migen and Verilator, a SoC simulation with 87 FP32 checkpoints bit-exact against the mirror, and the card, with tokens compared against the software model" caption="The verification chain carried over unchanged: every new unit had to reproduce the CPU path bit for bit before it landed." >}}

## Then one unit at a time

From there the method was always the same: profile a token on the card, take the largest piece the
CPU is still doing, move it into a unit that reproduces the CPU path bit for bit, measure, keep or
revert. On the 27B:

| bitstream | what moved into hardware | cycles | 27B tok/s |
| --- | --- | ---: | ---: |
| CPU path | baseline: the CPU does all the float work | 702 M per forward | 0.21 |
| v12 | the resident projection chain: quantize, integer Hadamard (FWHT), requantize, restore | 412 M | 0.36 |
| v14 | the SiLU-multiply unit | 324 M | 0.46 |
| v17 | the DeltaNet stream (engine op 5) | 81 M | 1.84 |
| v18 | the attention stream (engine op 6) | 49 M | 3.04 |
| v19 | unit speed-ups, overlap of units under the streams | 44 M | 3.42 |
| v22 | RoPE on the norm unit | 42 M | 3.58 |
| v31b | DDR4-1500, sys at 137.5 MHz | 34 M | **4.03** |

{{< figure src="img/27b-builds.jpg" alt="Bar chart of seconds per 27B token by bitstream: v8/v12 4.69 s (0.213 tok/s, the CPU does the float work), v12 resident 2.75 s, v14 2.17 s with the SiLU unit, v17 0.54 s with the DeltaNet stream, v18 0.33 s with attention streamed, v19 0.29 s, v22 0.28 s with RoPE in hardware" caption="Seconds per 27B token, bitstream by bitstream, on the XCU1525." >}}

Two of these steps are worth a closer look, because they are where the design's structure paid
off.

**The Hadamard unit.** The rotations first ran as dense 1,024x1,024 matrices on the engine, which
is correct but expensive. The resident chain keeps a projection's input in the scratchpad: the
activation unit quantizes it to int8 blocks, an integer fast Walsh-Hadamard transform rotates it
(only additions and subtractions, in a butterfly), it is requantized, the engine runs the ternary
projection, and the restore unit turns the int32 results back into FP32. The FFN down projection
went from 180 M to 39 M cycles. Because the transform is integer, it is exact, and the mirror can
reproduce it bit for bit.

**The DeltaNet stream.** The recurrent state is 128x128 FP32 per head, 48 heads per layer on the
27B: far too large for on-chip memory, and it must be read and written at every token. Rather than
a new memory path, it became another stream on the existing one. The state lives in DRAM, striped
across the engine channels: lane l owns rows l, l+16, and so on of every head, reads them through
its channel's reader, updates them in exact FP32 (decay, delta correction with the key, output with
the query), and writes them back through its channel's writer, in place. The engine already knew
how to stream from every channel at full rate; the DeltaNet heads went from 21.6 M to 3.5 M cycles
per forward. The attention stream (op 6) follows the same idea for the K/V rows.

Not everything we tried stayed. A multi-token-prediction head grafted on the 27B for speculative
decoding was exact and had 60 to 64% of its drafts accepted on the board, but each verified token
still paid about 10.5 M cycles of serial work: 0.72x the plain decode speed, so it was removed.
Speculation needs a cheap verification pass, and on this engine it is not cheap yet.

## Bonsai 2 2B and 4B: two days

On 15 October, PrismML released the Ternary-Bonsai-2 2B, 4B and 9B. Reading their headers, the
first finding was that they are the same architecture as the 27B we already ran. The differences
were small:

- **Hadamard blocks of 512** (4,096 for the 9B) instead of 1,024. The firmware's resident path
  refused other sizes and fell back, correctly, to dense rotations on the engine, at the cost of
  about 10% more streamed bytes. The 1,024-point Hadamard unit and the requantize unit then learned
  half blocks, so the 2B and 4B run the integer rotation in hardware too.
- **Hadamard version 2, a tied LM head.** The 2B and 4B have no output matrix: the LM head reuses
  the token embedding table as a folded weight, with the input rotated and the lookup inverted.
  The LM head is about a quarter of the 2B's streamed bytes, so it streams like any projection.
- **Two converter bugs**: a check that assumed layer 0 has attention (it is a DeltaNet layer), and
  an image name.

The golden model ran the 2B unmodified and answered "The capital of France is Paris." Two days
after the release, on 17 October, the 2B and 4B gave the reference tokens on the YPCB-00338, at
7.4 and 3.2 tok/s on the first hybrid bitstream.

The real work was fitting the hybrid units into a Kintex-7 480T next to eight channels and two
DDR3 controllers, and it changed the design in useful ways:

- **No vector unit.** Once every operation had its fixed-function unit, the microcoded vector unit
  was no longer needed on the hybrid builds. Removing it gave the first hybrid bitstream that
  closed timing outright.
- **DeltaNet sub-lanes.** The DeltaNet stream computes as lanes x sub-lanes in lockstep, so a board
  with four channels can still run it at width 8 without changing the DRAM layout, the engine or
  the firmware.
- **Clock domains where the work is.** The channel unpackers and pipes in their own 143 MHz clock,
  each channel's reader in the DDR controller's clock, the rest at 100 MHz.
- **A softmax unit**, which cut the attention at 1,024 tokens of context from 6.5 M to 1.9 M
  cycles per forward.
- **Keeping the streams away from the DDR I/O.** The engine's switching next to the DDR3
  controllers' I/O narrowed their read eye. Keeping the streaming datapath out of those clock
  regions restored it, and the hybrid builds use ECC with a forward retry for the rare
  uncorrectable read.

| build (YPCB-00338, DDR3) | change | 2B tok/s | 4B tok/s |
| --- | --- | ---: | ---: |
| first hybrid (17 Oct) | golden tokens, DDR3-933 | 7.41 | 3.16 |
| `ypcb_hyb7` | RoPE on the norm unit, list image cache, no vector unit | 13.20 | 5.98 |
| `ypcb_hyb13` | channel pipes in their own clock | 16.78 | |
| `ypcb_hyb15` | softmax unit, fused firmware lists | 18.45 | 8.15 |
| `ypcb_hyb21` (20 Oct) | DDR3-1066, pipes at 143 MHz | **20.33** | **9.23** |

The next day, the same core and the 2B ran on the Efinix Ti375 of [part 2](/posts/trixium-ti375/),
which reached 14.3 tok/s and, with int8 attention, 13.8 tok/s at 2,040 tokens of context.

## The like-for-like comparison with openTPU

[Part 1](/posts/trixium/) compared TriXium and [openTPU](https://github.com/FeSens/openTPU) on the
same YPCB-00338 card with different models. Bonsai 2 makes the comparison like for like: the 2B and
4B are ternary versions of the Qwen3.5-2B and 4B that openTPU runs at 4 bits.

| Qwen3.5 model, YPCB-00338, DDR3-1066 | openTPU (4-bit weights) | TriXium (ternary) |
| --- | ---: | ---: |
| 2B | 12.1 tok/s | **20.33 tok/s** |
| 4B | 5.9 tok/s | **9.23 tok/s** |
| bytes per token | ~2.5x more | |

TriXium reads about 2.5x fewer bytes per token and decodes 1.6 to 1.7x faster. The gap is smaller
than the bytes ratio because the hybrid build uses only about half of the DRAM bandwidth: the 2B
streams 0.41 GB per token, so 20.3 tok/s is about 8.3 GB/s of the card's 17.1 GB/s. The rest of a
token is serial work on each layer's chain (the scratchpad units, the DeltaNet recurrence, the
CPU's list round trips) that does not yet overlap with the weight streams. That is where the ternary
format's advantage is still to be collected. It is also a fair warning about the comparison:
openTPU runs the original models at 4 bits, PrismML's ternary models are different models, and this
compares speed, not answer quality.

## Results, all models

Batch-1 decode on the card, first tokens checked against the reference:

| board | memory | model | tok/s |
| --- | --- | --- | ---: |
| FK33 (VU33P) | HBM2, 16 channels | Bonsai 1.7B / 8B | 106.3 / 30.3 |
| XCU1525 (VU9P) | 4x DDR4-1500 | Bonsai 1.7B / 8B | 53.8 / 14.9 |
| | | Bonsai 2 27B | 4.03 |
| YPCB-00338 (Kintex-7 480T) | 2x DDR3-1066 | Bonsai 1.7B / 8B | 35.2 / 8.11 |
| | | Bonsai 2 2B / 4B | 20.33 / 9.23 |
| Efinix Ti375 C529 | LPDDR4x x32 | Bonsai 2 2B | 14.32 (13.81 at 2,040 tokens, int8 attention) |

## What made the port easy

Looking back, three decisions made Bonsai 2 a matter of days rather than a redesign:

- **The control plane in software.** A RISC-V running the model loop in C meant the new
  architecture ran, slowly but exactly, before any new gateware existed. The hardware then only had
  to take over the parts that measured slow.
- **A generic streamer instead of a model-shaped datapath.** The engine streams from every memory
  channel; the DeltaNet state and the KV rows became more streams on the same channels, not new
  memory paths.
- **Bit-exactness as the contract.** Every unit reproduced the CPU path bit for bit, so each step of
  the port could be checked in simulation in minutes, and the boards only had to confirm.

The cost is honest too: each bitstream is specialised for a model family, and a new family means
firmware, sometimes gateware, then a few hours of build. For models that change architecture every
few months, we think that is the right trade on an FPGA, and the 9B and the next Bonsai generation
will tell.

---

*TriXium runs [PrismML](https://prismml.com/news/ternary-bonsai)'s Ternary-Bonsai and
Ternary-Bonsai-2 models (on Qwen's Qwen3 and Qwen3.5 architectures, with Gated DeltaNet) from
their GGUF files, with PrismML's llama.cpp fork as the reference. openTPU's numbers are from its
README. The code is on [GitHub](https://github.com/enjoy-digital/trixium), BSD-2-Clause. Previous
parts: [the PCIe cards](/posts/trixium/) and [the Efinix Ti375](/posts/trixium-ti375/). Thumbnail:
PrismML's [Bonsai 2 announcement](https://prismml.com/news/bonsai-2-family) image.*

*Work and ideas by Enjoy-Digital; written up with AI in the loop.*
