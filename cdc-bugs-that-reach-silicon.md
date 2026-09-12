---
layout: default
title: "The Clock-Domain-Crossing Bugs That Reach Silicon"
description: "Why a correct two-flop synchroniser does not mean a correct design."
---

# The Clock-Domain-Crossing Bugs That Reach Silicon

Every hardware engineer learns the two-flop synchronizer early, and it works. Metastability is a probabilistic phenomenon, the second flop gives the first one a full clock period to resolve, and the resulting mean time between failures is measured in centuries. That part of the problem is solved.

Which is precisely why the CDC bugs that reach silicon are never metastability bugs. They are *correlation* bugs, *convergence* bugs, and *protocol* bugs — failures that occur even when every single crossing in the design is properly synchronized. They pass RTL simulation because RTL simulation has no notion of a setup violation. They pass gate-level simulation because the specific delay combination that triggers them didn't occur in that run. They appear on silicon, intermittently, at one temperature, on nine parts out of a thousand.

Here are the four that matter.

---

## 1. Multi-bit crossings synchronized bit by bit

A 4-bit counter crosses from `clk_a` to `clk_b`, and someone has correctly instantiated a two-flop synchronizer on each of the four bits.

Each bit is now individually safe. The bus is not.

When the counter transitions from `0111` to `1000`, all four bits change simultaneously in the source domain. In the destination domain, each bit's synchronizer independently resolves its metastable state, and there is no mechanism forcing them to resolve in the same cycle. Bit 3 may settle one cycle before bits 2, 1 and 0. For exactly one destination clock cycle, the receiver sees `1111` — a value the counter never held.

If that value is a FIFO pointer, the receiver concludes the FIFO is full when it is nearly empty, or empty when it is full. If it is a state encoding, the receiver enters a state that the transmitter never entered.

**Fixes, in order of preference:**

- **Gray coding.** Only one bit changes per increment, so the worst case is that the receiver sees the previous value for one extra cycle — always a legal value. This is why every async FIFO uses Gray-coded pointers, and it is the only reason.
- **MCP / handshake.** Hold the data stable in the source domain, cross a single-bit request, wait for a synchronized acknowledge. The data bus itself is never synchronized — it is simply guaranteed stable while the receiver samples it.
- **Async FIFO.** The general solution; use it when the crossing is a data stream rather than a status value.

**The rule:** if more than one bit can change in the same source clock cycle, per-bit synchronizers are wrong. Not risky — wrong.

---

## 2. Reconvergence: one signal, two synchronizers

A single source signal is synchronized into the destination domain twice — perhaps by two different sub-blocks that each wanted their own copy, perhaps because a synthesis tool duplicated the synchronizer for fanout or timing reasons.

The two synchronizer chains resolve independently. For one cycle, the destination domain contains two copies of the "same" signal holding different values. Any logic that combines them — an `AND`, a comparison, a state machine reading both — sees a transient state that is logically impossible.

This one is especially unpleasant because it can be *introduced by the tools* after the RTL has passed CDC review. A synthesis tool that duplicates a synchronizer flop to meet fanout constraints has created a reconvergence bug in a design that was clean at RTL.

**Fixes:**

- Synchronize once, in one place, and fan the synchronized signal out within the destination domain.
- Mark synchronizer flops with the appropriate `dont_touch` / `async_reg` style attribute so the tools stop optimizing them.
- Run CDC analysis on the *netlist*, not only the RTL. This is the step most teams skip, and it is where tool-introduced reconvergence is caught.

---

## 3. The signal isn't stable long enough to be sampled

A pulse generated in a fast domain crosses into a slow domain. The two-flop synchronizer is present and correct. The pulse is one `clk_fast` cycle wide. `clk_fast` is 400 MHz; `clk_dest` is 50 MHz.

The pulse is 2.5 ns wide. The destination samples every 20 ns. Most of the time, the pulse simply isn't there when the destination looks. The synchronizer does nothing wrong; it faithfully synchronizes whatever was on its input at the sampling edge, and the answer was zero.

This one is particularly nasty because it works *intermittently* — pulses that happen to align with a destination edge get through, and the ones that don't vanish. In simulation with a clean 8:1 clock ratio and aligned edges, it may work every time. On silicon with real jitter and an unrelated PLL, it works about as often as you'd expect from the duty cycle.

**The requirement:** a signal crossing into a slower domain must be held stable for **at least one and a half destination clock periods** — in practice, two, because you cannot assume edge alignment.

**Fixes:**

- **Toggle synchronizer**: toggle a level in the source domain instead of pulsing, and edge-detect it in the destination. The level stays until the next event, so it cannot be missed.
- **Open-loop pulse stretcher**: widen the pulse in the source domain to span the destination period. Works only if you know the frequency ratio and it cannot change — which, with DVFS in the picture, is a dangerous assumption.
- **Closed-loop handshake**: source holds the request until it sees a synchronized acknowledge. Slower, immune to frequency ratio entirely, and the right default when the clock relationship is programmable.

---

## 4. Asynchronous reset removal is itself a crossing

Everyone remembers that asynchronous reset *assertion* is fine — that's the point of an async reset. Almost everyone forgets that **deassertion** is a timing-critical event.

When an asynchronous reset releases, every flop in the domain leaves reset. If the release edge lands too close to an active clock edge, a flop may violate its *recovery* time and go metastable — or, worse, flops in different parts of the domain may exit reset on *different clock cycles*. A state machine whose bits leave reset one cycle apart does not start in its reset state. It starts somewhere else, and for a one-hot encoding that can mean no state at all.

**The fix is the reset synchronizer**, and it is about eight lines:

```systemverilog
// Async assert, sync deassert
logic rst_meta, rst_sync_n;
always_ff @(posedge clk or negedge async_rst_n) begin
  if (!async_rst_n) begin
    rst_meta   <= 1'b0;
    rst_sync_n <= 1'b0;
  end else begin
    rst_meta   <= 1'b1;
    rst_sync_n <= rst_meta;
  end
end
```

Reset asserts immediately and asynchronously, exactly as intended. Reset *releases* synchronously to `clk`, so every flop in the domain leaves reset on the same edge. One of these per clock domain, and reset ordering across domains becomes a design decision you make deliberately rather than one the tools make for you.

---

## Why simulation will not save you

It is worth being precise about this, because "we ran a lot of simulation" is the most common reason these bugs survive.

**RTL simulation** uses a zero-delay or unit-delay model. There is no setup window, so there is no metastability, so there is no CDC failure — RTL simulation is *structurally incapable* of showing you these bugs. It will show you correlation bugs only if the specific clock phase relationship in that run happens to expose one, which is a lottery.

**Gate-level simulation with SDF** can show timing violations, but it explores exactly one clock phase relationship per run, and real CDC failures depend on a phase relationship that drifts continuously with temperature, voltage and PLL jitter. You would need an implausible number of runs to sample the failure.

**Formal CDC tools** are the actual answer. They do not simulate; they structurally identify every crossing, classify the synchronization scheme, and prove or disprove the stability requirements. A CDC run is a static check like lint, not a dynamic one, and it finds all four bugs above by construction.

The practical sign-off flow:

1. Structural CDC on RTL — every crossing found and classified.
2. Waive nothing without a written justification that names the protocol guaranteeing stability. "Reviewed, looks fine" is not a justification; it is how waivers become bugs.
3. Re-run CDC on the synthesized netlist to catch tool-introduced reconvergence.
4. Metastability injection in simulation — randomly delay synchronizer outputs by a cycle — to prove the design tolerates what will actually happen.

Step 4 is the one that gets skipped, and it is the one that finds the correlation bugs, because it is the only technique in the list that makes RTL simulation capable of modelling the failure at all.

---

## The summary you can hand to a reviewer

| Crossing | Correct mechanism |
|---|---|
| Single control bit, slow → fast | Two-flop synchronizer |
| Single pulse, fast → slow | Toggle synchronizer or handshake |
| Multi-bit counter/pointer | Gray code |
| Multi-bit arbitrary data | MCP handshake or async FIFO |
| Data stream | Async FIFO |
| Async reset | Async assert, sync deassert, per domain |

Six rows. Every CDC bug I have personally debugged was a violation of one of them, and in every case the design contained correct two-flop synchronizers throughout.

The synchronizer was never the hard part.

---

*Written by Syed Hassan Raza Kazmi, a working RTL and physical-design engineer. If your team needs documentation, application notes, or developer content that survives contact with an actual engineering audience, [get in touch](mailto:kzm5286@gmail.com).*
