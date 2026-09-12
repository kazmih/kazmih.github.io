---
layout: default
title: "Reading Your First Timing Report Without Panicking"
description: "A field guide to the numbers in a static timing analysis report."
---

# Reading Your First Timing Report Without Panicking

The first static timing analysis report most engineers see is about four hundred lines long, contains roughly eleven numbers that matter, and arrives attached to a message that says "we have negative slack, can you look." This guide is about those eleven numbers.

I'm going to walk through a report the way you actually read one: top down, asking a specific question at each stage, and stopping as soon as you know what to fix. The examples use PrimeTime-style output, but the structure is close enough to Tempus, or to an FPGA tool's timing analyzer, that the reading strategy transfers.

## Start at the bottom: slack is the only verdict

Every timing path report ends with two lines that are the entire summary:

```
  data required time                                      1.184
  data arrival time                                      -1.271
  --------------------------------------------------------------
  slack (VIOLATED)                                       -0.087
```

**Slack = required time − arrival time.** Positive means the signal got there before the deadline. Negative means it didn't, by that many nanoseconds.

That's it. Everything above those lines exists to explain *why* the arrival time is what it is, and every fix you will ever make is an attempt to move one of those two numbers.

Two aggregate figures usually accompany a run, and they answer different questions:

- **WNS** (worst negative slack) — the single worst path. Answers *"how bad is the worst problem?"*
- **TNS** (total negative slack) — the sum of all negative slacks. Answers *"how many problems do I have?"*

A WNS of −0.087 with a TNS of −0.09 is one path, probably one cell, likely a lunchtime fix. A WNS of −0.087 with a TNS of −340 is not a path problem; it is a structural problem — a wrongly constrained clock, a missing multicycle path, a floorplan that put two communicating blocks in opposite corners. **Never start fixing paths before you have compared WNS to TNS.** It is the cheapest diagnostic in the flow and the most commonly skipped.

## Then look at the header: is this even a real violation?

```
  Startpoint: u_core/u_alu/result_reg[7]
              (rising edge-triggered flip-flop clocked by clk_core)
  Endpoint:   u_dma/u_fifo/wr_ptr_reg[3]
              (rising edge-triggered flip-flop clocked by clk_bus)
  Path Group: clk_bus
  Path Type:  max
```

Four things to check, in this order, before you touch a single cell:

**1. Do the startpoint and endpoint clocks match?** Above, they don't — `clk_core` to `clk_bus`. Either these clocks are genuinely synchronous and related, or this is an asynchronous crossing that should have been declared as such and excluded. Roughly a third of the "urgent" violations I've been handed were paths between clocks that never had a defined phase relationship in the first place. The tool cannot know that; it will faithfully analyze a path that has no physical meaning and report a terrifying number.

**2. Path type: `max` or `min`?** `max` is a **setup** check — data arrived too late. `min` is a **hold** check — data arrived too *early*, and overwrote the capture flop before it had sampled the previous value. These are completely different problems with completely different fixes, and conflating them wastes days. Setup is fixed by making logic faster or the clock slower. Hold is fixed by making data slower — inserting delay buffers — and is largely insensitive to clock period, which is why hold violations do not disappear when you relax the frequency.

**3. Which corner and mode?** The report header names the scenario. A setup violation in the slow corner (low voltage, high temperature, slow process) is expected and real. A setup violation in the *fast* corner usually means something is miscorrelated in your setup, not in your design. Hold violations behave in the reverse.

**4. What path group?** Register-to-register, input-to-register, register-to-output, and clock-gating paths are different populations with different owners. An input-to-register violation is frequently an I/O budgeting question for the SoC integrator, not a synthesis problem inside your block.

## Now the path itself: where did the time go?

The body of the report is a table of arrival times accumulating down the path:

```
  Point                                    Incr       Path
  --------------------------------------------------------
  clock clk_core (rise edge)               0.000      0.000
  clock network delay (propagated)         0.412      0.412
  u_alu/result_reg[7]/CK (DFFQ_X2)         0.000      0.412 r
  u_alu/result_reg[7]/Q (DFFQ_X2)          0.093      0.505 f
  u_glue/i_23/Z (BUFF_X1)                  0.061      0.566 f
  u_glue/i_24/Z (AND2_X1)                  0.148      0.714 f
  ... 27 more cells ...
  u_fifo/wr_ptr_reg[3]/D (DFFQ_X1)         0.000      1.271 f
  data arrival time                                   1.271
```

You are looking for exactly three things.

**Logic depth.** Count the cells. A path through 30 cells at 1 GHz is not going to close by resizing gates; it needs pipelining or restructuring, and that is an RTL conversation, not a place-and-route one. A path through 6 cells that misses by 80 ps is a buffer-and-resize problem the tool can usually solve itself.

**The one fat increment.** Scan the `Incr` column for a single cell that costs 5–10× its neighbours. A 0.4 ns increment in a chain of 0.06 ns increments is almost always one of: a minimum-drive cell driving a huge fanout, a long unbuffered net that placement stretched across the block, or a high-Vt cell that got swapped in during leakage recovery and never swapped back. All three are cheap fixes with large returns. This is the highest-yield 20 seconds in the entire report.

**Transition times.** If your report includes the `Trans` column, look for slow edges. Slow input transitions inflate cell delay, which slows the next edge, which inflates the next delay. A single badly-driven net poisons the four cells downstream of it, and fixing the driver fixes all five delays at once.

## The required-time side: the half everyone ignores

```
  clock clk_bus (rise edge)                1.500      1.500
  clock network delay (propagated)         0.389      1.889
  clock reconvergence pessimism            0.021      1.910
  clock uncertainty                       -0.150      1.760
  library setup time                      -0.576      1.184
  data required time                                  1.184
```

Engineers spend hours optimizing the data path and never read this block, which is a mistake, because it frequently contains the actual problem.

**Clock network delay** — 0.389 ns to the capture flop versus 0.412 ns to the launch flop. That 23 ps difference is **skew**, and here it works against you: the capture clock arrives *earlier* than the launch clock, stealing 23 ps from your setup budget. Skew is not noise; on a poorly balanced tree it is routinely 100–200 ps, which at 1 GHz is 10–20% of your entire cycle.

**Clock uncertainty** — 150 ps subtracted to cover jitter and, pre-CTS, estimated skew. If someone set this to a conservative placeholder early in the flow and never updated it after clock tree synthesis, you are hunting for 150 ps of logic delay that doesn't need to exist. Check this number before you optimize anything. I have watched a team spend a week closing timing that a stale uncertainty value had invented.

**Library setup time** — 0.576 ns is large for a setup requirement and usually means the capture flop is seeing a slow input transition, or it is a scan flop with a worse setup characteristic than its non-scan sibling.

**CRPR** (clock reconvergence pessimism removal) — credit the tool gives back for shared clock path segments it had pessimistically derated twice. You want this enabled; if it is off, every path with a common clock ancestor is reported worse than it is.

## The reading order, compressed

1. **WNS vs TNS** — one problem or a systemic one?
2. **Clock pair** — is this path even real?
3. **`max` or `min`** — setup or hold? Different universe.
4. **Cell count** — architectural fix or optimization fix?
5. **The fat increment** — the one bad cell, usually there is one.
6. **Uncertainty and skew** — is the deadline itself wrong?

Six questions, about ninety seconds, and you will know whether to open the RTL, call the floorplanner, or fix a constraint file typo.

## The thing nobody tells you

Most timing violations are not timing problems. They are constraint problems wearing a timing costume — a missing `set_false_path`, a clock defined without its generated children, a multicycle path that exists in the designer's head and nowhere in the SDC, an I/O budget nobody agreed on.

So before you optimize a single gate, read your constraints with the same suspicion you'd apply to the design. The tool is never wrong about the arithmetic. It is only ever as right as what you told it.

---

*Written by Syed Hassan Raza Kazmi, a working RTL and physical-design engineer. If your team needs documentation, application notes, or developer content that survives contact with an actual engineering audience, [get in touch](mailto:kzm5286@gmail.com).*
