---
title: "LiteSATA on ECP5: SATA for the fully open FPGA flow"
date: 2026-07-31T08:30:00+02:00
draft: false
description: "SATA on the Lattice ECP5 had been parked since 2023: the DCU's slow electrical idle stopped two attempts, and a small company cannot gamble weeks of R&D on a nice-to-have. One weekend of AI-assisted bring-up got OOB working with gaps made of data, a week of follow-up finished the link and command layers, and the resulting PHY is now merged in LiteSATA: 3.0 Gb/s with yosys/nextpnr on an ECPIX-5, IDENTIFY, BIST and a BIOS boot path."
summary: "The full story of LiteSATA's ECP5 PHY: four years parked on issue #27, one weekend of AI bring-up to first link-up, one very silly bench bug, and three merged PRs. What AI-assisted FPGA engineering looks like when it works, effort included."
tags: ["litesata", "litex", "sata", "ecp5", "lattice", "serdes", "litescope", "ai"]
categories: ["hardware"]
series: ["AI-era FPGA"]
showHero: false
---

*TL;DR: SATA on the Lattice ECP5 had been on the LiteSATA todo list since
[issue #27](https://github.com/enjoy-digital/litesata/issues/27) in 2023: two earlier attempts
hit the same silicon limitation, and as a small company we could never justify the weeks of R&D
a real attack would cost, so it sat abandoned. One long weekend of AI-assisted bring-up, with
real hands (CSRs over UARTBone) and real eyes (LiteScope, plus an oscilloscope on SCPI), found
a way around the limitation that no datasheet suggests: make the OOB gaps out of data. A week of
follow-up campaigns then finished the link and the command layer, and the cleaned-up PHY is
[now merged in LiteSATA](https://github.com/enjoy-digital/litesata/pull/38): link-up at 3.0 Gb/s
with yosys/nextpnr, full IDENTIFY, BIST, and a BIOS boot path, with broader drive coverage as
the main remaining work. This is the whole story, dead ends and power wires included.*

{{< figure src="img/oob-challenge.svg" alt="Three-part diagram: the SATA OOB requirement of 106.7 ns bursts of a 375 MHz square separated by precisely timed gaps; the wall, with measured ECP5 electrical-idle floors of 220 ns (2022) and 650 ns (2026) bars overshooting the COMWAKE and COMRESET shall-detect windows; and the way out, a waveform where gaps are a constant zero-transition data pattern instead of electrical idle" caption="The whole problem on one page: what the spec demands, what the ECP5's electrical idle can actually do, and the trick that finally unlocked it." >}}

## Four years on the todo list

LiteSATA has supported Xilinx transceivers for a decade, but a fully open flow (core plus
toolchain) needs a Lattice part, and the ECP5 is the obvious one: cheap, well supported by
yosys/nextpnr, and its SerDes reaches the 3.0 Gb/s SATA needs. The catch has always been **OOB
signaling**, the wake-up handshake that precedes any data on a SATA link (explained in a minute,
if transceivers are not your daily bread). Xilinx transceivers implement it in hardware. The
ECP5 implements none of it.

I first tried in December 2020: a COMGenerator in fabric, driving the DCU's electrical idle. The
drive answered with COMINIT, and then nothing; it never accepted our COMWAKE. In March 2022 I
came back with a Versa board and measured why: the shortest electrical-idle pulse the DCU would
actually produce was about **220 ns**, and a COMWAKE gap must be 106.7 ns. In 2023 I wrote it all
up as issue #27, with a roadmap and a pointer to
[ktemkin's LFPS work](https://ktemk.in/post/serdes-lfps/) as the likely direction:

> Some initial tests have been done to generate/receive COMRESET/COMINIT/COMWAKE in the ecp5
> branch, but the integrated electrical idle detection feature of the ECP5 was apparently not
> fast enough to detect/generate the OOB sequencing correctly.

And there it sat, mostly abandoned. Not because it stopped being interesting, and not because
the limitation looked unbeatable, but because Enjoy-Digital is a small company. A real attack
meant betting a few weeks of engineering time on a nice-to-have with no guaranteed outcome,
against client work that pays for everything else, and we cannot afford that kind of R&D gamble.
That is the honest reason most open issues like this one stay open. It is also exactly the
situation where an agent earns its keep: let it hunt for a first proof of concept at a cost we
can absorb, and put ourselves back on it once it is unlocked. The rest of this post is what that
looked like in practice.

## OOB and the DCU, in two minutes

Two pieces of context first, for FPGA developers who have so far stayed away from transceivers.

A transceiver (on the ECP5 the hard block is called a **DCU**, Dual Channel Unit: one macro, two
channels, up to 5 Gb/s on the ECP5-5G) turns parallel words in your fabric into a multi-gigabit
serial stream on a differential pair, and back. At SATA's 3.0 Gb/s the fabric side is a 20-bit
word every 6.7 ns. The rest of the block exists to make that stream usable: the receiver
recovers a clock from the data's own transitions (the CDR), 8b10b coding guarantees those
transitions exist and marks word boundaries with comma characters, and a word aligner finds
them. The DCU also exposes a few raw controls that matter for this story: an electrical-idle
switch (stop driving the pair), a loss-of-signal detector (RLOS), and a low-data-rate bypass
(LDR) that lets fabric logic drive the pads directly.

The bootstrap problem is that none of this machinery works when a SATA link starts. The line is
quiet, the CDR has nothing to lock to, word boundaries do not exist yet. So SATA begins below
all of it, with **OOB (out-of-band) signaling**: the two ends wake each other by alternating
bursts of activity with silences, and the information is carried by the timing of the silences.
A receiver needs nothing more than a squelch detector (is there signal?) and a timer.

{{< figure src="img/oob-sequence.svg" alt="Sequence diagram between a host ECP5 and a SATA drive: the host sends COMRESET (six bursts with 320 ns gaps), the device answers COMINIT (same shape), host and device exchange COMWAKE (six bursts with 106.7 ns gaps), then ALIGN primitives flow at line rate, CDR locks, speed is negotiated and the link is up; a note explains that the receiver only needs a squelch and a timer, and that gap duration is the message" caption="The OOB handshake. Each COM signal is six bursts; the burst content barely matters, the gap length is what the receiver times to tell them apart." >}}

The exchange is short: the host sends COMRESET, the device answers COMINIT ("I am here"), they
trade COMWAKE, and only then does real data appear: the device streams ALIGN primitives at its
preferred rate, the host echoes them, the CDR locks, 8b10b takes over, speeds are agreed, link
up. On a Xilinx transceiver the whole dance is a hard state machine: assert a request pin, wait
for a detect flag. The ECP5 DCU has no such engine, so all of it has to be rebuilt in fabric
from the raw controls above. How hard can six bursts and some silence be? That question is the
rest of this post.

## Hands and eyes

What changed is not the problem; it is the cost of iterating on it. The
[AI-era post](/posts/ai-era-fpga/) argued that agents change FPGA work when you give them a fast
feedback loop. Bring-up on real hardware pushes that idea to its limit, because the loop runs
through a physical board, a physical drive and a physical instrument. So the first hours of the
weekend went into building the bench, not fighting SATA:

{{< figure src="img/test-setup.svg" alt="Bench diagram: a host PC running Claude (driving the bench) and Codex (independent review) connected by UARTBone to an ECPIX-5 board carrying the DCU SerDes PHY, about twenty runtime CSR knobs and a LiteScope analyzer, connected by SATA cable to a drive; a Siglent SDS1104X-E probes the TX coupling cap and reports back over Ethernet SCPI" caption="The bench. Hands: every experiment is a CSR write over UARTBone. Eyes: LiteScope inside the FPGA, a Siglent on the TX pair outside it, both readable by the agent without anyone touching the desk." >}}

**Hands.** The bench design (`bench/ecpix5.py`, a `SoCMini` on a LambdaConcept ECPIX-5) exposes
UARTBone at 1 Mbaud, so `litex_server` plus `RemoteClient` gives the agent full CSR access from
Python. Crucially, the PHY grew about twenty runtime knobs during the weekend (burst lengths, gap
modes, electrical-idle shaping, alignment behavior, pattern selection), all mapped as CSRs. Most
experiments were a register write and a re-run, not a rebuild. When a rebuild was needed,
yosys/nextpnr/trellis turned one around in 20 seconds to 2 minutes, and `openFPGALoader` put it
on the board. The agent could try things at the rate it could think of them.

**Eyes.** Two of them. Inside the FPGA, a LiteScope analyzer (115 bits at 100 MS/s, probe groups
for the OOB controller, the datapath and the link layer, in the sys, tx or rx clock domain) that
the agent drove through `litescope_cli`, value triggers included. Outside, a Siglent SDS1104X-E
on the TX coupling cap, automated over Ethernet: a small helper wraps SCPI capture and turns
waveforms into burst/gap run lengths in nanoseconds, an automated ruler for OOB timing. (Bench
reality: the Siglent's raw SCPI socket crashes under heavy use; its VXI-11 server survives.
The helper knows both.)

Without either half, none of what follows happens: an agent that cannot act has nothing to
test, and one that cannot observe has no way to tell whether a change helped. With both, it runs
the experiment, looks at the result, and updates. Every step went into a lab journal in the repo
(`bench/BRINGUP.md`), which is where most of the quotes below come from.

## The campaign

What follows is compressed from three days and 47 commits, kept in order, because the
repetition is the point: hypothesis, measurement, refutation, next.

### A dead clock and a 2019 bug

Night one: the TX word clock measured 0 MHz in every configuration, while RX ran fine. The agent
diffed our DCU parameters against a Diamond-generated reference netlist, parameter by parameter,
and found the one Diamond always sets and LiteICLink never had: `D_SYNC_LOCAL_EN`. One bit,
silently wrong since 2019, and TX came alive. The fix went
[upstream to LiteICLink](https://github.com/enjoy-digital/liteiclink) the same day, together
with a LiteScope CLI fix found along the way. Not a bad haul for a first night.

### The same wall, four years later

With the clock alive, OOB generation through electrical idle hit a sharp cliff: gaps of 226 ns
and above provoked the drive massively, gaps of 200 ns and below did nothing at all. That is the
2022 measurement reproduced on a different board, four years apart. The journal's later
literature dig added context that made the wall make sense: **no Lattice family ever had a
hardware SATA OOB engine**, the ECP2M and ECP3 tool support was soft logic around the same
electrical-idle pin, and the DCU feature literally named "OOB" in the ECP5 sheet is a low-speed
video bypass path, a naming collision that cost the campaign real time.

### The drive that never heard us

This is the one that changed how the rest of the weekend was run. Early results looked
encouraging: the drive seemed to answer our COMRESET storms. Then the agent measured instead of
believing, timing hundreds of drive responses against our transmissions:

> The drive free-runs an autonomous COMINIT beacon: period 10.003 ms, rock-stable. Phase test:
> COMRESET storm at P~441 us, latency distribution over [0,P): UNIFORM. The drive's beacon phase
> is completely uncorrelated with our COMRESET sequences: the drive has NEVER decoded a single
> sequence we sent. All prior "COMRESET answered" results were this beacon plus attribution
> error.

Every "success" so far had been us hearing the drive talk to itself. The journal then catalogs
the false-positive classes explicitly "so nobody re-chases them", and switches all further phase
analysis to hardware-paced tests after catching its own host-paced measurements aliasing against
the 10.003 ms beacon. That catalog sounds bureaucratic; it caught two more artifacts before
Sunday.

### Proving everything else

To separate "our OOB is wrong" from "everything is wrong", the agent looped TX back to RX
through the physical connector path and brought the link up against itself: full OOB handshake,
8b10b ALIGN/SYNC at 3.0 Gb/s, ten out of ten enable cycles. That pinned the fault surface to
exactly one thing, the shape of our transmitted gaps as seen by a real drive's squelch, and
proved every other block (DCU configuration, CDR, comma alignment, gearboxes, clock-domain
crossings, the control FSM) worked.

## Gaps made of data

Two course corrections arrived mid-campaign from Codex, the second agent reviewing the journal
cold: the gaps had been tuned against the spec's loose *may-detect* windows instead of the tight
*shall-detect* ones (COMWAKE: 101.3 to 112 ns), and the burst content rule got pinned down
(D24.3 timed at the gen1 rate, a 375 MHz square at any line rate). With the target finally
exact, the breakthrough came out of a failed experiment: a de-emphasis idea was implemented,
measured and cleanly disproven, but its implementation carried a mechanism nobody had pointed at
the drive yet: switching the serializer between a toggling pattern and a **constant** pattern.
From the journal:

> The trick: make the gaps out of DATA, not electrical idle. The serializer switches between a
> toggling pattern (burst) and a CONSTANT pattern (gap) in ONE WORD, 6.7 ns, no EI pipeline, no
> dependence on the coupling caps. The gap still carries differential voltage, but it has ZERO
> transitions, and a real SATA squelch is transition/energy sensitive rather than a DC amplitude
> detector, so it reads as idle.

The proof is the drive's own beacon rate as a function of our gap length, three independent
passes:

```text
gap(ns)  pass1 pass2 pass3
    53    600   600   600   (below the COMWAKE window)
   107     78    26     6   SUPPRESSED       <- COMWAKE nominal 106.7 ns
   160   1280  1280  1280   rate doubles
   213    600   600   600
   320      0     0     0   FULLY SUPPRESSED <- COMRESET nominal 320 ns
   427      0     0     0   FULLY SUPPRESSED
   533    600   600   600   (above the window)
```

The drive stops beaconing exactly when our gaps land in the windows it must recognize: it hears
us. The planned hardware modifications (cap swap, RF switch, fabric-IO rework) all became
unnecessary, and the ~650 ns electrical-idle floor stopped mattering because electrical idle is
no longer how gaps are made. The 2023 issue guessed the right family of technique (drive the
line from user logic, like ktemkin's LFPS trick); the missing insight was that for SATA's much
shorter gaps, the data path itself is the fast switch.

## The last mile of the weekend

With the drive responding, the remaining bugs fell in a line. The RX word aligner sat on the
wrong 10-bit boundary, decoding SYNC primitives into garbage; letting the DCU's link state
machine own the alignment fixed it, and clean ALIGN primitives appeared, 1488 of them in one
LiteScope capture. Then the link would come up and immediately collapse, and the reason is a
lesson about specs: our ALIGN timer only reset on ALIGN primitives, but an established SATA link
switches to SYNC, so the timer expired 41 µs after success and tore the link down. Holding it on
any K-prefixed primitive produced this commit, verbatim from the log:

```text
b69e358  phy: FIRST SATA LINK-UP ON ECP5 - fix ALIGN timer to hold on any primitive.
```

`sata_phy_status = 0xf`: PHY ready, TX ready, RX ready, control ready, at 3.0 Gb/s, through a
real cable into a real drive. As far as the weekend's fairly obsessive literature search could
tell, no public end-to-end SATA link on an ECP5 existed before that Saturday evening. One more
classic bug after that (the bench only instantiated the SATA core when BIST was enabled, so the
PHY transmitted D0.0 forever the moment the datapath mux switched, and the drive hung up), plus
parameterized link tolerances, and the link now comes ready in under 0.1 s and holds for
multi-second stretches, 89% of samples over a 20 s window at last measurement.

## The week after: a drive that would not speak

The weekend closed with a link that held and an IDENTIFY that did not. On Monday I rotated the
crew: the Claude session had run its course, and Codex picked up the same bench, the same hands
and eyes, and the same journal. The journal did the actual handoff: a few thousand lines of what
we know and how we know it, which moves between agents far better than any chat history.

The first days brought real gateware kills. The bypass PCS path had never had word alignment at
all, so a fabric word aligner was written and proven in simulation at all 20 possible bit
offsets. Then a bad one to have to write down, verbatim from the log: "THE FABRIC DECODERS WERE
NEVER IN THE NETLIST", the 8b10b decoders had been silently optimized away and had to be
registered explicitly. With the aligner re-arm policy fixed on top, the link against the real
drive reached 100% uptime, and the link layer accepted our IDENTIFY frame end to end: `X_RDY`
through `WTRM`, the drive answering `R_RDY`, `R_IP`, `R_OK`, zero transmission errors.

And then, for a while, nothing. Not one FIS back, not even the power-up signature every SATA
drive must send unprompted. The journal's verdict hardened into "evidence converges on an
ATA-dead drive", which turned out to be true, for the most mundane reason possible: the bench
power had been set up for the SSDs of the early campaigns, which run on 5 V alone, and the
known-good test drive was a 3.5-inch HDD whose spindle wants 12 V. The controller booted, did
textbook OOB, politely acknowledged our frames, and had no spinning disk behind any of it. A few
hours of very sophisticated digital forensics went into an assumption made for SSDs and never
rechecked for an HDD; sometimes the bug is a wire. 😄 With the power fixed, the drive returned
all 256 IDENTIFY words:

```text
model       TOSHIBA DT01ACA100
firmware    MS2OA8U0
sectors     1,953,525,168
capacity    1,000,204,886,016 bytes
features    Gen1, Gen2, Gen3, LBA48
```

Read BIST followed, then bounded write BIST. For the record, the gateware work of that week was
all real and all needed: the aligner, the decoders and the timers had to be right regardless.
The wire just decided when we would find out.

## Three PRs and a merge

The last job for Codex was the unglamorous, important one: take an exploration branch of 120
commits, twenty debug knobs and a 3900-line journal, and distill just the useful work into
something a maintainer would want to read. The result landed as three PRs, all merged on
30 July 2026:

- **[liteiclink#33](https://github.com/enjoy-digital/liteiclink/pull/33)**: opt-in ECP5 OOB
  support in the SerDes (`with_oob=True`), backward compatible, with independent TX/RX
  initialization and direct electrical-idle control.
- **[litesata#38](https://github.com/enjoy-digital/litesata/pull/38)**: the ECP5 Gen2 SATA PHY
  as one clean commit on the raw 10BSER datapath, ECP5-specific behavior contained, generic
  controller untouched. Validation in the PR: the full test suite passing, timing met with
  margin on the ECPIX-5, a complete 256-word IDENTIFY, a destructive 16 MiB pseudorandom
  write/verify with zero errors, and the Xilinx PHYs regression-checked.
- **[litex-boards#773](https://github.com/litex-hub/litex-boards/pull/773)**: `--with-sata` on
  the ECPIX-5 target, with a dedicated PLL for the fixed 150 MHz reference clock and the
  LiteSATA core exposed to the LiteX BIOS, which also gives Linux-on-LiteX-VexRiscv a boot path
  from the drive.

{{< alert "circle-info" >}}
Merged means a first working implementation, not a finished one. In particular, the data-driven
OOB trick is accepted by some drives but not yet by all of them: every squelch circuit is built
a little differently, and a drive that rejects our gaps never gets past the handshake. The
encouraging part is that it is all-or-nothing: a drive that accepts the OOB sequence has then
worked fine (link, IDENTIFY, reads and writes). Making the OOB pass on every drive is the main
piece of future work, along with gen1 rate support. Refinements welcome; but three years after
it was filed, issue #27 finally has its answer.
{{< /alert >}}

## What this says about the method

The honest accounting: one long weekend of Claude driving the bench mostly unattended, then a
week of Codex sessions on evenings and overnights, with me directing the way you would a
colleague. Reading the journal, saying "this smells like an attribution problem, go measure
it", choosing between proposed directions, vetoing hardware mods, and eventually walking to the
bench to fix a power wire. The agents did the rest: RTL, bench, builds, CSR pokes, LiteScope
and Siglent captures, the journal, the retractions, about 120 commits.

So no, this did not make SATA on ECP5 easy. It took 70 numbered campaigns and a real week of
guidance, and the silliest bug of the whole story was a wire on my side of the bench. What
changed is the cost of trying, and for a small company that is the whole point. We cannot bet
weeks of engineering time on speculative R&D, which is why this sat abandoned rather than
half-done. An agent changes that math: the wrong theories (shaped EI, PCIe EI mode, SCI gating,
de-emphasis cancellation, five PCS modes) were each implemented, measured and buried in hours,
on nights and weekends, at a cost we could actually absorb. Let the agent chase the first proof
of concept; once it is unlocked, the engineers take over for the part that has to be right.
That split is pragmatic R&D for a small structure, and it is the
[AI-era post](/posts/ai-era-fpga/) thesis running on real hardware.

Two practical things I would repeat. Use more than one agent: Claude and Codex reviewed each
other during the weekend and relayed cleanly afterwards, and it is the lab journal, not the
conversation, that makes the relay work. And put the effort into the bench before the prompts:
the hands and eyes are what turned the agents into engineers here, and the one rail we gave
them no eyes on is where that silly wire bug got to sit undetected. The next bench gets a
multimeter the agent can read. 🙂

One last thing the merge taught me: the distillation step matters as much as the exploration.
What landed upstream contains none of the twenty debug knobs and all of the lessons, because a
maintainer merges a clean diff, not a heroic journal.

LiteX's pitch has always been "build your hardware easily". This month is what the continuation
feels like: and bring it up efficiently. 🙂

## Try it

The support is on master now. On an ECPIX-5 with a drive (and both supply rails 🙂):

```sh
python3 -m litex_boards.targets.lambdaconcept_ecpix5 --with-sata --build --load
```

then `sata_init` / `sata_read` from the LiteX BIOS, or LiteSATA's `bench/` utilities for the
full init/IDENTIFY/BIST flow. The `ecp5-v2` branch keeps the complete bring-up history and the
journal for the curious.

---

*Built on [LiteSATA](https://github.com/enjoy-digital/litesata),
[LiteX](https://github.com/enjoy-digital/litex),
[LiteICLink](https://github.com/enjoy-digital/liteiclink) and
[LiteScope](https://github.com/enjoy-digital/litescope), on a
[LambdaConcept ECPIX-5](https://shop.lambdaconcept.com/home/46-2-ecpix-5.html).
[ktemkin's LFPS write-up](https://ktemk.in/post/serdes-lfps/) is the acknowledged prior
art for driving OOB-class signaling from ECP5 user logic.*

*Work and ideas by Enjoy-Digital; engineered and written up with AI in the loop: Claude on the
weekend exploration, Codex on the finish and the PRs.*
