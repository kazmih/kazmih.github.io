---
layout: default
title: "Five AXI Handshake Mistakes That Pass Simulation and Hang Silicon"
description: "Protocol violations that survive regression, and the assertions that catch them."
---

# Five AXI Handshake Mistakes That Pass Simulation and Hang Silicon

AXI's handshake looks trivial. Two wires per channel: `VALID` from the source, `READY` from the destination, transfer happens on the cycle both are high. Four rules in the specification, about a page of text.

It is trivial — right up until the moment your block is integrated behind an interconnect it has never seen, talking to a slave that applies backpressure in a pattern your testbench never generated, and the system stops. Then it is a three-week debug with a logic analyzer and a lot of people in the room.

Here are the five failures I keep finding, why each one survives verification, and the assertion that catches it.

---

## 1. The master waits for `READY` before asserting `VALID`

This is the classic, and it is an outright specification violation rather than a matter of taste.

The AXI protocol is explicit: **a source must not wait for the destination to assert `READY` before it asserts `VALID`.** The destination *is* permitted to wait for `VALID` before asserting `READY`. The asymmetry is deliberate and it is the only thing preventing deadlock.

The broken code usually looks reasonable:

```systemverilog
// WRONG — awvalid depends on awready
assign awvalid = have_request && awready;
```

The author's intent was "only issue when the slave can take it." The effect is that if the slave also waits for `VALID` before asserting `READY` — which it is fully entitled to do — both sides wait for each other forever.

**Why simulation misses it:** because most testbench slave models assert `READY` unconditionally. `awready` is a constant 1, the combinational dependency resolves immediately, and everything works beautifully for eighteen months of regression. Then the design meets a real interconnect with a registered slice that asserts `READY` only in response to `VALID`, and the bus stops on the first transaction.

**Catch it:**

```systemverilog
property no_valid_dependency;
  @(posedge aclk) disable iff (!aresetn)
    $rose(awvalid) |-> $past(awready) || 1'b1; // structural check is better
endproperty
```

Honestly, an assertion is the weaker tool here — this is a *structural* bug, and the reliable catch is a lint/CDC-class rule or a five-minute code review that greps every `*valid` assignment for a dependency on the matching `*ready`. Do the grep. It takes five minutes and it is the highest-value five minutes in an AXI review.

---

## 2. `VALID` is deasserted before the handshake completes

Once a source asserts `VALID`, it **must keep it asserted until the cycle in which `READY` is also high.** No timeouts, no "the slave is taking too long, let me retry," no deassertion because an upstream FIFO went empty.

```systemverilog
// WRONG — valid drops when the source FIFO empties
assign wvalid = !fifo_empty;
```

If the slave is applying backpressure while your FIFO drains from another port, `wvalid` glitches low mid-transaction. Some slaves tolerate this. Interconnects generally do not, and the ones that don't tend to lose a beat silently rather than erroring — which means you find out via corrupted data three modules downstream, not via a bus error.

**Catch it:**

```systemverilog
property valid_stable_until_ready;
  @(posedge aclk) disable iff (!aresetn)
    (wvalid && !wready) |=> wvalid;
endproperty
assert property (valid_stable_until_ready);
```

Write this for all five channels. It is ten lines total and it is the single highest-yield assertion set in AXI verification.

---

## 3. Payload changes while `VALID` is held

The companion to mistake 2, and subtler. `VALID` stays high correctly, but `AWADDR`, `WDATA`, `WSTRB`, `AWLEN` or `AWSIZE` change underneath it while the destination is applying backpressure.

The specification requires payload to remain stable from the assertion of `VALID` until the handshake completes. A destination is allowed to sample the payload on *any* cycle where `VALID` is high — not only the cycle where the transfer completes. A pipelined slave that captures the address one cycle early and the data on the handshake cycle will happily pair address A with data B, and you will get a write to the wrong location with no protocol error anywhere.

**Catch it:**

```systemverilog
property payload_stable;
  @(posedge aclk) disable iff (!aresetn)
    (awvalid && !awready) |=> $stable({awaddr, awlen, awsize, awburst, awprot});
endproperty
assert property (payload_stable);
```

**Why simulation misses it:** an always-ready testbench slave never holds `VALID` for more than one cycle, so the payload never has an opportunity to change during a stall. Which brings us to the root cause of three of these five bugs.

---

## 4. The testbench never applies backpressure

Not a design bug — a verification bug, and the one that lets the other four escape.

If your slave BFM drives `awready`, `wready`, and `arready` to constant 1, and your master BFM drives `bready` and `rready` to constant 1, then **you have never tested a stall**. Every stall-related bug in your design is invisible, and stall-related bugs are the majority of AXI bugs.

The fix is a few lines:

```systemverilog
// Randomized backpressure in the slave BFM
always @(posedge aclk) begin
  if (!aresetn) awready <= 1'b0;
  else          awready <= ($urandom_range(0, 99) < ready_pct);
end
```

Sweep `ready_pct` across your regression: 100% (baseline), 50% (typical), 10% (heavy congestion), and — critically — a directed mode that holds `READY` low for 20+ consecutive cycles. Long stalls find the counters that were sized for the happy path.

Add a second dimension: independent backpressure per channel. A slave that stalls `W` but not `AW` exposes a different class of bug than one that stalls both together.

---

## 5. Assuming `AW` arrives before `W`

The write address and write data channels are **independent**. Write data may arrive before, after, or simultaneously with its address. There is no ordering guarantee between them, and real masters — especially those behind a register slice or a width converter — routinely deliver data first.

A slave written as a state machine that goes `IDLE → wait for AW → wait for W → respond` will deadlock against such a master: the master is waiting for `wready`, the slave is waiting for `awvalid`, neither side is wrong per the spec, and the bus is dead.

A correct slave buffers whichever arrives first and pairs them when both are present. Roughly:

```systemverilog
// Independent channel acceptance, paired downstream
always_ff @(posedge aclk) begin
  if (awvalid && awready) begin aw_pending <= 1'b1; addr_q <= awaddr; end
  if (wvalid  && wready ) begin w_pending  <= 1'b1; data_q <= wdata;  end
  if (aw_pending && w_pending && !b_busy) begin
    // issue the write, then clear both
    aw_pending <= 1'b0; w_pending <= 1'b0;
  end
end
```

The related rule: **the write response `BVALID` must not be asserted until both the address and the final data beat have been accepted.** An early response tells the master a write completed that hasn't, and the master will happily read back stale data and believe it.

---

## The pattern underneath all five

Four of these five bugs share one root cause: **the verification environment was more polite than the real system.**

Constant-`READY` BFMs, ordered channel stimulus, short bursts, and no concurrent traffic produce an environment in which a substantial class of protocol bugs is structurally unobservable. The design passes because the test cannot fail.

So the practical checklist, in order of return on effort:

1. Grep every `*valid` assignment for a dependency on `*ready`. Five minutes.
2. Bind the stability assertion set — `VALID` held, payload stable — to every AXI interface in the design. Ten lines per interface, and bindable to the interface rather than the DUT, so it costs nothing per instance.
3. Randomize `READY` on every channel independently, and add a directed long-stall test.
4. Test `W` before `AW` explicitly. It is one directed sequence and it finds a real bug surprisingly often.
5. Use a protocol checker — the free VIPs and the checkers bundled with most simulators encode the full rule set, and none of the above is a reason to skip them.

None of this is difficult. It is only ever skipped because the handshake looks too simple to be worth the ceremony. Two wires. Four rules. Three weeks of debug.

---

*Written by Syed Hassan Raza Kazmi, a working RTL and physical-design engineer. If your team needs documentation, application notes, or developer content that survives contact with an actual engineering audience, [get in touch](mailto:kzm5286@gmail.com).*
