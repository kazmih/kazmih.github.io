---
layout: default
title: "Five AXI Handshake Mistakes That Pass Simulation and Hang Silicon"
description: "A review checklist for AXI interfaces, and the assertions that make the review unnecessary."
---

# Five AXI Handshake Mistakes That Pass Simulation and Hang Silicon

AXI's handshake looks trivial. Two wires per channel: `VALID` from the source, `READY` from the destination, transfer happens on the cycle both are high. Four rules in the specification, about a page of text.

It is trivial — right up until the moment a block is integrated behind an interconnect it has never seen, talking to a slave that applies backpressure in a pattern its testbench never generated. Then it is a long debug with a logic analyzer and a lot of people in the room.

This is a review checklist. Five failure modes that recur across designs, why each survives verification, and the assertion that would have caught it.

---

## 1. The master waits for `READY` before asserting `VALID`

The classic, and an outright specification violation rather than a matter of taste.

The AXI protocol is explicit: **a source must not wait for the destination to assert `READY` before it asserts `VALID`.** The destination *is* permitted to wait for `VALID` before asserting `READY`. That asymmetry is deliberate and it is the only thing preventing deadlock.

The broken code usually looks reasonable:

```systemverilog
// WRONG - awvalid depends on awready
assign awvalid = have_request && awready;
```

The author's intent was "only issue when the slave can take it." The effect is that if the slave also waits for `VALID` before asserting `READY` — which it is fully entitled to do — both sides wait for each other forever.

**Why simulation misses it:** most testbench slave models assert `READY` unconditionally. `awready` is a constant 1, the combinational dependency resolves immediately, and everything works for as long as that model is the only slave in the picture. Then the design meets a real interconnect with a registered slice that asserts `READY` only in response to `VALID`, and the bus stops on the first transaction.

**Catch it:** an assertion is the weaker tool here. This is a *structural* bug, and the reliable catch is a lint rule or a five-minute review that greps every `*valid` assignment for a dependency on the matching `*ready`. Do the grep. It is the highest-value five minutes in an AXI review, and unlike an assertion it cannot be defeated by a stimulus set that never exercises the case.

---

## 2. `VALID` is deasserted before the handshake completes

Once a source asserts `VALID`, it **must keep it asserted until the cycle in which `READY` is also high.** No timeouts, no retry because the slave is taking too long, no deassertion because an upstream FIFO went empty.

```systemverilog
// WRONG - valid drops when the source FIFO empties
assign wvalid = !fifo_empty;
```

If the slave is applying backpressure while the FIFO drains from another port, `wvalid` glitches low mid-transaction. Some slaves tolerate this. Interconnects generally do not, and the ones that don't tend to lose a beat silently rather than raising an error — so the symptom is corrupted data three modules downstream, not a bus error anywhere near the cause.

**Catch it:**

```systemverilog
property valid_stable_until_ready;
  @(posedge aclk) disable iff (!aresetn)
    (wvalid && !wready) |=> wvalid;
endproperty
assert property (valid_stable_until_ready);
```

Write it for all five channels. Ten lines total, and it is the highest-yield assertion set in AXI verification.

---

## 3. Payload changes while `VALID` is held

The companion to mistake 2, and subtler. `VALID` stays high correctly, but `AWADDR`, `WDATA`, `WSTRB`, `AWLEN` or `AWSIZE` change underneath it while the destination is applying backpressure.

The specification requires payload to remain stable from the assertion of `VALID` until the handshake completes. A destination may sample the payload on *any* cycle where `VALID` is high — not only the cycle where the transfer completes. A pipelined slave that captures the address one cycle early and the data on the handshake cycle will pair address A with data B, and you get a write to the wrong location with no protocol error raised anywhere.

**Catch it:**

```systemverilog
property payload_stable;
  @(posedge aclk) disable iff (!aresetn)
    (awvalid && !awready) |=> $stable({awaddr, awlen, awsize, awburst, awprot});
endproperty
assert property (payload_stable);
```

**Why simulation misses it:** an always-ready testbench slave never holds `VALID` for more than one cycle, so the payload never gets an opportunity to change during a stall. Which brings us to the root cause of three of these five.

---

## 4. The testbench never applies backpressure

Not a design bug — a verification bug, and the one that lets the others escape.

If the slave model drives `awready`, `wready` and `arready` to constant 1, and the master model drives `bready` and `rready` to constant 1, then **no stall has ever been tested**. Every stall-related bug in the design is invisible, and stall-related bugs are the majority of AXI bugs.

The fix is a few lines:

```systemverilog
// Randomized backpressure in the slave BFM
always @(posedge aclk) begin
  if (!aresetn) awready <= 1'b0;
  else          awready <= ($urandom_range(0, 99) < ready_pct);
end
```

Sweep `ready_pct` across the regression: 100% as a baseline, 50% as typical, 10% for congestion, and — critically — a directed mode that holds `READY` low for 20+ consecutive cycles. Long stalls find the counters that were sized for the happy path.

Add a second dimension: independent backpressure per channel. A slave that stalls `W` but not `AW` exposes a different class of bug than one stalling both together.

---

## 5. Assuming `AW` arrives before `W`

The write address and write data channels are **independent**. Write data may arrive before, after, or simultaneously with its address. There is no ordering guarantee, and real masters — particularly behind a register slice or a width converter — routinely deliver data first.

A slave written as `IDLE → wait for AW → wait for W → respond` will deadlock against such a master: the master waits for `wready`, the slave waits for `awvalid`, neither is violating the specification, and the bus is dead.

A correct slave buffers whichever arrives first and pairs them when both are present:

```systemverilog
// Independent channel acceptance, paired downstream
always_ff @(posedge aclk) begin
  if (awvalid && awready) begin aw_pending <= 1'b1; addr_q <= awaddr; end
  if (wvalid  && wready ) begin w_pending  <= 1'b1; data_q <= wdata;  end
  if (aw_pending && w_pending && !b_busy) begin
    aw_pending <= 1'b0; w_pending <= 1'b0;
  end
end
```

The related rule: **`BVALID` must not be asserted until both the address and the final data beat have been accepted.** An early response tells the master a write completed that hasn't, and the master will read back stale data and believe it.

---

## The pattern underneath

Four of these five share one root cause: **the verification environment was more polite than the real system.**

Constant-`READY` models, ordered channel stimulus, short bursts, no concurrent traffic — that combination produces an environment in which a substantial class of protocol bug is structurally unobservable. The design passes because the test cannot fail.

So, in order of return on effort:

1. Grep every `*valid` assignment for a dependency on `*ready`. Five minutes.
2. Bind the stability assertion set — `VALID` held, payload stable — to every AXI interface. Ten lines per interface, bindable to the interface rather than the DUT, so it costs nothing per instance.
3. Randomize `READY` on every channel independently, and add a directed long-stall test.
4. Test `W` before `AW` explicitly. One directed sequence.
5. Use a protocol checker. The checkers bundled with most simulators encode the full rule set, and none of the above is a reason to skip them.

None of this is difficult. It is only ever skipped because the handshake looks too simple to be worth the ceremony.

Two wires. Four rules.

---

*Written by Syed Hassan Raza Kazmi, a working RTL and physical-design engineer. If your team needs documentation, application notes, or developer content that survives contact with an actual engineering audience, [get in touch](mailto:kzm5286@gmail.com).*
