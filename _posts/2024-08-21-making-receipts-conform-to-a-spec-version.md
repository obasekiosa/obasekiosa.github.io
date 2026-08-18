---
title: Making receipts conform to a spec version
date: 2024-08-21 14:00:00 +0100
categories: [Engineering, Starknet]
tags: [starknet, rpc, json-rpc, spec]
---

A node on a blockchain network is nothing but a promise about shapes. The spec defines the response format of every endpoint, tools are written against the spec, and the moment a node returns a field with the wrong name, the wrong type, or the wrong presence rule, the tool breaks and the node gets the blame. My work was making a receipt endpoint return exactly what the spec version demanded, no more, no less.

## The divergence

The endpoint returns a transaction receipt. The internal model had grown organically: fields accumulated over time, some were named loosely, some were present conditionally, and none of it had been checked against the published spec for the network version in question.

The diff was a list. A field renamed to match the spec. A field that should be an object with amount and unit, flattened into a plain integer. A field that must be present on success and absent on revert, present in both. A nested structure for emitted events that disagreed with the spec's field names. The receipt looked right to someone reading it in isolation and wrong to any tool written against the contract.

## What conformity actually means

Conformity is not correctness in the loose sense. The receipt contained correct data. It contained the right values, computed honestly, from the right source. What it did not do is place those values where the spec places them, under the names the spec names them, with the presence rules the spec assigns.

That distinction matters because it changes what the fix looks like. There was no bug in the data path, there was a bug in the contract path. The fix was a mapping layer between the internal model and the wire model, one that knows the spec version and translates: internal field to spec field, internal type to spec type, internal presence rule to spec presence rule.

## The mapping layer

The design was a dedicated wire model per spec version. Not a patch on the internal model, a separate type that exists only at the boundary:

```rust
pub struct TransactionReceipt {
    pub transaction_hash: Felt,
    pub actual_fee: FeePayment,
    pub messages_sent: Vec<MessageToL1>,
    pub events: Vec<Event>,
    pub execution_status: ExecutionStatus,
    pub finality_status: FinalityStatus,
}
```

The mapping function is the whole contract. It takes the internal receipt, produces the wire receipt, and the tests for that function are the real spec conformance suite. Every field, every type conversion, every conditional presence rule is tested there, because that function is the only place the node and the spec meet.

The details that mattered:

**Enums as strings.** The execution and finality statuses are enumerations with exact spellings. A status spelled differently from the spec is a parse failure for every client. The wire model used the spec's exact string values and the mapper translated internal variants to them.

**Units.** Fee amounts carry a unit field in the spec. The internal model carried a raw integer. The mapper attached the unit explicitly rather than leaving consumers to guess.

**Conditional presence.** The spec defines which fields appear on which execution outcomes. The mapper applied the rules rather than always emitting everything, because a tool that expects a field absent on revert will misparse a receipt that includes it.

## The testing that mattered

The conformance tests used real receipts from real blocks: successful transactions, reverted transactions, transactions with L1 messages, transactions emitting many events. Each was mapped and the output was compared structurally against the spec's field table. Field by field, name, type, presence. The reverted cases are the ones that catch lazy implementations, because the success path is where everyone looks and the revert path is where the presence rules actually differ.

## The lessons

Specs are the API. The node's internal model is a private implementation detail, and it is tempting to think the internal model is the truth. It is not. The wire format is the truth, and the mapping layer is where the truth is enforced.

Wire models belong at the boundary. The temptation is to reuse the internal model at the edge, and the consequence is that every internal drift becomes a breaking change. A separate wire type per spec version is the only thing that keeps a node honest when the spec moves.

Conformance is a test problem. The mapping function is small. The tests that pin every field, every type, and every presence rule are the actual deliverable, and they are also the documentation of what the spec demands.
