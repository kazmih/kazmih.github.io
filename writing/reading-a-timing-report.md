---
layout: default
title: "Reading Your First Timing Report Without Panicking"
description: "A field guide to the numbers that decide whether your design ships."
---

# Reading Your First Timing Report Without Panicking

The first static timing analysis report most engineers see is about four hundred lines long, contains roughly eleven numbers that matter, and arrives attached to a message that says "we have negative slack, can you look."

This guide is about those eleven numbers, and about the order to read them in — which matters more than most people expect, because the goal is not to understand the report. The goal is to stop reading it as early as possible, having learned what to fix.

The examples use PrimeTime-style output, but the structure is close enough to Tempus, or to an FPGA tool's timing analyzer, that the reading strategy transfers.

## Start at the bottom: slack is the only verdict

Every timing path report ends with two lines that are the entire summary:

```
  data required time                                      1.184
  data arrival time                                      -1.271
  --------------------------------------------------------------
  slack (VIOLATED)                                       -0.087
```

**Slack = required time − arrival time.** Positive means the signal got there before the deadline. Negative means it didn't, by that many nanoseconds.

Everything above those two lines exists to explain why the arrival time is what it is, and every fix you will ever make is an attempt to move one of those two numbers.

Two aggregate figures usually accompany a run, and they answer different questions:

- **WNS** (worst negative slack) — the single worst path. *How bad is the worst problem?*
- **TNS** (total negative slack) — the sum of all negative slacks. *How many problems do I have?*

A WNS of −0.087 with a TNS of −0.09 is one path, probably one cell, likely a lunchtime fix. A WNS of −0.087 with a TNS of −340 is not a path problem at all; it is structural — a wrongly constrained clock, a missing exception, a floorplan that put two communicating blocks in opposite corners.

Comparing those two numbers is the cheapest diagnostic in the flow and the most commonly skipped. Do it before you open a single path report.

## Then the header: is this even a real violation?

```
  Startpoint: u_core/u_alu/result_reg[7]
              (rising edge-triggered flip-flop clocked by clk_core)
  Endpoint:   u_dma/u_fifo/wr_ptr_reg[3]
              (rising edge-triggered flip-flop clocked by clk_bus)
  Path Group: clk_bus
  Path Type:  max
```

Four checks, in this order, before you touch a cell.

**1. Do the startpoint and endpoint clocks match?** Above, they don't — `clk_core` to `clk_bus`. Either these clocks are genuinely synchronous and related, or this is an asynchronous crossing that should have been declared as such and excluded. A surprising share of "urgent" violations turn out to be paths between clocks that never had a defined phase relationship in the first place. The tool cannot know that. It will faithfully analyze a path with no physical meaning and report a terrifying number for it.

**2. Path type `max` or `min`?** `max` is a **setup** check — data arrived too late. `min` is a **hold** check — data arrived too *early* and overwrote the capture flop before it sampled the previous value.

These are different problems with different fixes and conflating them wastes days. Setup is fixed by making logic faster or the clock slower. Hold is fixed by making data *slower*, and is largely insensitive to clock period — which is why hold violations stubbornly fail to disappear when you relax the frequency, and why a team that has just "fixed timing" by dropping 100 MHz can be baffled to find half their violations untouched.

**3. Which corner and mode?** A setup violation in the slow corner (low voltage, high temperature, slow process) is expected and real. A setup violation in the *fast* corner usually means something is miscorrelated in your setup rather than wrong in your design. Hold behaves in reverse.

**4. Which path group?** Register-to-register, input-to-register, register-to-output and clock-gating paths are different populations with different owners. An input-to-register violation is frequently an I/O budgeting question for the SoC integrator, not a synthesis problem inside your block — and arguing about it inside the block is a way to spend a week solving someone else's problem.

## The path body: where did the time go?

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

Three things, and nothing else on a first pass.

**Logic depth.** Count the cells. A path through 30 cells at 1 GHz will not close by resizing gates; it needs pipelining or restructuring, and that is an RTL conversation, not a place-and-route one. A path through 6 cells that misses by 80 ps is a buffer-and-resize problem the tool can usually solve itself if you let it.

Knowing which of those two you're looking at, within ten seconds, is most of the skill.

**The one fat increment.** Scan the `Incr` column for a single cell costing 5–10× its neighbours. A 0.4 ns increment in a chain of 0.06 ns increments is almost always one of: a minimum-drive cell driving a large fanout, a long unbuffered net that placement stretched across the block, or a high-Vt cell swapped in during leakage recovery and never swapped back. All three are cheap fixes with large returns.

This is the highest-yield twenty seconds in the entire report, and it is the thing experienced engineers do reflexively while newer ones read the path from the top.

**Transition times.** If your report includes the `Trans` column, look for slow edges. Slow input transitions inflate cell delay, which slows the next edge, which inflates the next delay. One badly-driven net poisons the four cells downstream of it — and fixing the driver fixes all five delays at once, which is why chasing transitions often beats chasing delays.

## The required-time side: the half everyone ignores

```
  clock clk_bus (rise edge)                1.500      1.500
  clock network delay (propagated)         0.389      1.889
  clock reconvergence pessimism            0.021      1.910
  clock uncertainty                       -0.150      1.760
  library setup time                      -0.576      1.184
  data required time                                  1.184
```

Engineers spend hours optimizing the data path and never read this block. It frequently contains the actual problem.

**Clock network delay** — 0.389 ns to the capture flop versus 0.412 ns to the launch flop. That 23 ps difference is **skew**, and here it works against you: the capture clock arrives *earlier*, stealing 23 ps from your setup budget. On a poorly balanced tree, skew is routinely 100–200 ps, which at 1 GHz is 10–20% of the entire cycle.

**Clock uncertainty** — 150 ps subtracted to cover jitter and, pre-CTS, estimated skew.

This number deserves suspicion out of all proportion to its size. It is set early, as a deliberately conservative placeholder, by someone who intended to revisit it after clock tree synthesis. Frequently nobody does. Worse, on a project with several active branches it is entirely normal to find two different values in two places with no record of which is authoritative — and a 100 ps disagreement is 100 ps of logic delay that somebody is being asked to find and remove for no reason at all.

Check what the value is, check that it is the value the project agreed on, and check that it was updated post-CTS. Before you optimize anything.

**Library setup time** — 0.576 ns is large for a setup requirement. It usually means the capture flop is seeing a slow input transition, or that it is a scan flop with a worse setup characteristic than its non-scan sibling.

**CRPR** (clock reconvergence pessimism removal) — credit the tool gives back for shared clock path segments it pessimistically derated twice. You want this enabled. With it off, every path with a common clock ancestor reports worse than it is, and you can spend real effort closing pessimism that does not exist.

## The failure mode that produces the most wasted effort

Before the mechanics, one structural point, because it accounts for more lost time than anything above.

**Most timing violations are not timing problems.** They are constraint problems wearing a timing costume.

The specific case worth internalizing is test and configuration pins. A DFT scan-enable, a test-mode select, a memory's built-in-self-test control — these are tied to a known value in functional operation, but unless you say so with `set_case_analysis`, the tool treats them as freely switching. It then analyzes clock and data paths through a test multiplexer that will never be selected, and reports violations on structures that cannot exist in functional mode.

The signature is distinctive: violations of *implausible* magnitude, often on clock-related paths, often ten nanoseconds or more on a design with a sub-nanosecond period. When the number is absurd, suspect the constraints before you suspect the design. A real logic path does not miss by fifteen cycles; a mis-analyzed test path does it easily.

The same logic covers missing false paths, undeclared generated clocks, and multicycle paths that exist in the designer's head and nowhere in the SDC.

So read your constraints with the same suspicion you would apply to the design. The tool is never wrong about the arithmetic. It is only ever as right as what you told it.

## One technique that saves an afternoon

When you have many violating paths and are deciding where to spend effort, check each path group's ceiling before analyzing anything inside it:

```tcl
get_attribute [get_timing_paths -groups {my_group} -max_paths 1] slack
```

If the worst path in a group is better than your threshold, nothing inside that group can violate it, and you can skip the group entirely. It is an obvious move once seen and very few people do it — the instinct is to open the reports and start reading. Checking the ceiling first routinely eliminates half the candidate groups in under a minute.

## The reading order, compressed

1. **WNS vs TNS** — one problem or a systemic one?
2. **Clock pair** — is this path even real?
3. **`max` or `min`** — setup or hold? Different universe.
4. **Magnitude sanity** — is the number physically plausible, or is a constraint missing?
5. **Cell count** — architectural fix or optimization fix?
6. **The fat increment** — the one bad cell; there is usually one.
7. **Uncertainty and skew** — is the deadline itself wrong?

Seven questions, about ninety seconds, and you will know whether to open the RTL, call the floorplanner, or go and read the constraint file.

---

*Written by Syed Hassan Raza Kazmi, a working RTL and physical-design engineer. If your team needs documentation, application notes, or developer content that survives contact with an actual engineering audience, [get in touch](mailto:kzm5286@gmail.com).*
