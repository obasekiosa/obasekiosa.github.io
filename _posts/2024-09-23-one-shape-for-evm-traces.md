---
title: One shape for EVM traces
date: 2024-09-23 13:00:00 +0100
categories: [Engineering, Ethereum]
tags: [ethereum, tracing, rpc]
---

Ethereum has two trace families. The debug family, `debug_traceTransaction` and `debug_traceBlock`, comes from the reference client. The trace family, `trace_transaction` and `trace_block`, comes from a different lineage. They describe the same execution, and for years they disagreed about the one case every tracing user eventually hits: when a transaction reverts.

## The divergence

On a successful transaction both families agree. The structure differs in naming and nesting, but the data is there: steps, gas, opcodes, memory, stack, storage changes.

On a revert they disagree. One family returns a trace with the error embedded inside the result structure, the error dump attached where the step data would go. The other returns a trace where the failure is expressed in the block level response, with the individual transaction trace left looking broken or empty. Tooling that handles one shape misparses the other. Debuggers built for one client behave erratically on the other. Support threads fill with "my trace has no error field" and "my trace has an error field where the data should be".

The deeper problem: the two endpoints in the same client did not even agree with each other. The error dump path in one endpoint emitted a different structure than the error path in the sibling endpoint. A developer debugging with one tool, then switching tools, saw two different shapes for the same reverted transaction.

## Why uniformity is a contract

Tracing is a debugging interface. Its consumers are not production automation that tolerates version churn, they are humans staring at execution, and tool builders who have to harden every shape they parse. Every shape difference is a tax on both.

The decision was to make the error output uniform: one structure for a failed execution across both families and both endpoints. A reverted trace carries its error in the same place, with the same fields, whether it came from `debug_traceTransaction`, `debug_traceBlock`, or `trace_block`. The success shape already differed by family and that difference is historical, but the error shape is where the confusion lives, because every project that touches tracing eventually touches a revert.

## The design

The uniform error shape needed three decisions:

**Where the error lives.** At the top of the individual trace result, alongside the data. Not nested inside the steps, not in the block level envelope. The trace result is the unit that describes one transaction, so the error belongs to that unit.

**What it contains.** The error message string, the reason the execution failed, and enough context to locate the failure. For a revert with data, the decode of that data, because a raw revert hex without a signature match is useless to the human who asked.

**What happens to the rest.** The partial trace data is still emitted. A reverted execution is often exactly what the user wanted to inspect, and wiping the steps because the transaction failed destroys the value of the call.

The work was a mapping layer. Each endpoint produced its own internal result type. The error paths were the wild divergence, and the mapping layer normalized them into the single shape before serialization. One function, one schema, one set of tests, four endpoints all passing through it.

## The testing that mattered

The fixture set was built from real reverted transactions: arithmetic overflows, failed transfers, reverted calls with data, reverted calls without data, deep revert chains where an inner call fails and the outer call passes the failure up. Each fixture ran through every endpoint, and the assertion was structural: same shape, same fields, same placement, only the values differing.

The deep revert chain is the case that catches copy paste implementations. The outer trace contains a nested failure, and the error must be reported at the level where the failure actually happened, not just at the top. Getting that right is the difference between a shape normalization and a real trace.

## The lessons

Error output is part of the interface. The happy path gets all the design attention, and the error path, where the users actually are when they call a tracer, is left to drift. Uniformity work should start at the failure cases, because that is where the users live.

Two families of the same protocol will diverge, and divergence costs real money in tooling. The fix is not to rewrite history, it is to normalize at the boundary and pin the result with fixtures from the real world.

A mapping layer is the right amount of architecture. Not a new tracing engine, not a rewrite of either family. One choke point where every endpoint's error output converges, plus fixtures that prove the choke point holds.
