---
layout: default
title: "The Constraint That Matched Nothing"
description: "Why a correct SDC constraint can silently stop applying, and how to catch it before it costs you a week."
---

# The Constraint That Matched Nothing

Here is a failure mode that costs teams days at a time, produces no error message, and is almost absent from the documentation.

The setup: a block that closed timing last month now reports a worst negative slack of −10 ns. Nobody touched the RTL. The constraint file is identical — you can diff it and confirm that. The constraints in it are correct; you can read them and verify each one. The design is broken anyway.

The cause is that one of those constraints stopped matching anything, and not a single tool in the flow is obliged to tell you.

## How a constraint stops applying without changing

Most SDC exceptions reference objects by hierarchical path:

```tcl
set_case_analysis 0 [get_pins u_mem_array/*/u_tmem_macro/FCA*]
set_false_path -from [get_pins u_ctrl/u_cfg_regs/*/CK]
set_multicycle_path 2 -to [get_cells u_dsp/u_mac/acc_reg*]
```

Those paths describe the hierarchy as it exists *in the RTL*. Synthesis is under no obligation to preserve it. Boundary optimization, ungrouping, and flattening all exist precisely to dissolve module boundaries in pursuit of timing and area, and when a tool ungroups `u_mem_wrapper/u_tmem_macro`, the resulting instance is frequently named by joining the levels with an underscore — `u_mem_wrapper_u_tmem_macro` — rather than preserving the slash.

Your constraint still says `u_mem_wrapper/u_tmem_macro/FCA`. There is no longer any object by that name. `get_pins` returns an empty collection. `set_case_analysis` is applied to nothing at all.

And here is the part that makes it expensive: **applying a constraint to an empty collection is not an error.** In most flows it is a warning at most, buried among the thousands of warnings every real run produces. The constraint file is still correct. The constraint is still there. It simply has no effect, and the timing report gives you no hint that this is the reason it looks insane.

## Why it tends to strike right after you change something unrelated

The cruel part of this failure mode is its timing. It usually appears immediately after a change to the constraints — a new I/O budget, a different set of timing exceptions, a revised clock definition.

That feels like cause and effect, and it is, but not in the way most people first assume. The chain is indirect: constraint content changes what synthesis optimizes for, which changes its structural decisions, which changes whether a given module boundary survives. So the file you edited breaks a *different* constraint that you did not edit, several thousand lines away, in a section nobody has looked at in six months.

Which means the natural debugging instinct — "I changed the I/O budget, so the problem is in the I/O budget" — sends you in precisely the wrong direction. You will spend two days auditing delay values that are fine.

## The constraints most exposed to this

Anything that names a hierarchical instance or pin path:

- `set_case_analysis` — the worst one, because its absence doesn't cause a violation, it causes *phantom* violations. Test and configuration pins that should be tied off are instead treated as freely switching, and the tool dutifully analyzes clock paths through a test multiplexer that will never be selected in functional mode. The result is tens of nanoseconds of violation on paths that do not physically exist.
- `set_false_path` and `set_multicycle_path` — silently lost, so real exceptions come back as real-looking violations.
- `set_dont_touch` — silently lost, so the cell you were protecting gets optimized away.
- `group_path` — the group exists but is empty, so a whole category of paths quietly reverts to the default group and disappears from the reporting you set up to watch it.
- `set_max_delay` / `set_min_delay` on internal points.

Constraints anchored on ports are largely safe; ports survive synthesis. It is everything *inside* the design that is at risk.

## The second version of this bug: generate blocks

There is a variant worth knowing separately, because it bites even when nothing was flattened.

A generated instance sits inside an extra level of hierarchy that does not appear in the RTL you read. A `for` loop inside a `generate` block introduces a level named something like `genblk1` or `gen_pipe[0]`, depending on whether the block was labelled. So:

```tcl
# Matches nothing
get_cells u_stripe_ctrl/*/data_q_reg*

# Matches, because it spans the generate level
get_cells u_stripe_ctrl/*/*/data_q_reg*
```

Same silent failure, same absence of any error, and this one is present from the very first run rather than appearing later. Any wildcard you wrote by reading the RTL source rather than by querying the netlist is a candidate.

## Catching it: verify the match, not the syntax

The fix is not cleverer wildcards. It is a verification step that most constraint flows simply do not have.

**After every synthesis run, confirm that each hierarchical pattern in your constraints actually matches something in the netlist that run produced.** Not the previous netlist. Not the RTL. The one in front of you.

```tcl
proc check_match {desc collection} {
  set n [sizeof_collection $collection]
  if {$n == 0} {
    puts "CONSTRAINT ERROR: '$desc' matched 0 objects"
  } else {
    puts "ok: '$desc' matched $n objects"
  }
}

check_match "tmem test pins" [get_pins -quiet u_mem_array/*/u_tmem_macro/FCA*]
```

Run it as part of constraint sourcing and make an empty match loud. It takes an afternoon to write for an existing design and it converts a category of silent, multi-day failures into a line of console output.

Three habits that make the check less necessary in the first place:

**Write both hierarchy shapes.** If you know a boundary is liable to be dissolved, constrain both spellings and let one of them match:

```tcl
set_case_analysis 0 [get_pins -quiet u_mem_array/*/u_tmem_macro/FCA*]
set_case_analysis 0 [get_pins -quiet u_mem_array/*_u_tmem_macro/FCA*]
```

This looks redundant and offends the tidy-minded. Leave it in. The cost of the extra line is nothing; the cost of the missing match is a week. When someone asks in review why both are there, the answer is that the netlist decides which one is correct, and it decides late.

**Preserve what you constrain.** If a module boundary carries constraints that depend on it, tell synthesis not to dissolve it — `set_dont_touch` on the instance, or the flow's equivalent ungrouping control. The clean version of this problem is not to be robust against flattening, but to prevent flattening where it matters.

**Query the netlist, not the source.** Build wildcards by running `get_cells` and `get_pins` interactively against the actual synthesized design and confirming the collection size, rather than by reading the RTL and inferring what the path must be. The RTL tells you what you wrote. Only the netlist tells you what exists.

## Why this is worth an explicit process step

Most timing debug methodology assumes the constraints are a fixed input and the design is the variable. That assumption is wrong in a specific and dangerous way: the constraints are text, but *the set of objects they apply to* is an output of synthesis, and it changes every run.

So a constraint file that was correct in March can be inert in September with no edit in between, and the flow will report this to you as a timing problem, because a timing report is the only language it has.

Which reduces to a rule worth putting on a checklist:

> A constraint that matches nothing is indistinguishable, in every report you will look at, from a constraint you never wrote.

The only difference is that you believe it is there.

---

*Written by Syed Hassan Raza Kazmi, a working RTL and physical-design engineer. If your team needs documentation, application notes, or developer content that survives contact with an actual engineering audience, [get in touch](mailto:kzm5286@gmail.com).*
