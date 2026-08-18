---
title: Exposing Cairo step counts as a response header
date: 2024-08-23 13:30:00 +0100
categories: [Engineering, Starknet]
tags: [starknet, cairo, rpc, metering]
---

On Starknet, execution cost is counted in Cairo steps. Every program running on the network consumes steps, and every tool that wants to reason about cost, profiling, fee estimation, or block scheduling, needs to know how many. The step count was computed by the VM, used internally, and then thrown away. The work was exposing it.

## What a step is

Cairo programs are compiled to a bytecode where each instruction costs a metered unit of execution. The number of steps a transaction burns is the fundamental measure of how expensive it was to run. It is the closest thing the network has to a gas measurement that everyone agrees on.

The VM produces the step count as a natural byproduct of execution. It must, because it tracks its own progress through the program. The question was never how to compute steps. It was how to get them out of the execution layer and into a place consumers could read them.

## The header decision

The first question was transport: body or header? The body was attractive because response bodies are structured and easy to extend. The problem was that bodies are contractual. Every field in a response body is part of the schema, and clients version their parsers against it. Adding a body field is a schema change with versioning consequences.

A header is metadata. It does not participate in the response schema, it does not break parsers that were written against the body, and it is trivially accessible in every HTTP client and in middleware. For a measurement that describes the work done to produce the response, a header is exactly the right home. The header carries the step count as a string to sidestep any JSON number precision concerns, a decision that matters on chains where counters grow beyond what floating point can represent exactly.

## The plumbing

The step count is born deep in the VM invocation. The path to the header crosses several layers, and each crossing had to be made explicit:

**The execution result.** The VM returns its result plus the resources consumed. The step count was already in that structure, already computed, just not propagated. The work was promoting it from a side detail to a first class field on the result type.

**The RPC handler.** The handler calls the executor and builds the response. It was the first place that had both the step count and the HTTP response object, and the natural choke point for the header. The handler reads the count off the execution result and sets the header before writing the body.

**The client contract.** Clients that wrap the RPC needed to know the header exists and what it means. The docs describe the header, its format, and when it is present. Absence semantics matter too: the header is only meaningful when execution actually ran, so its presence signals that the count comes from a real execution rather than a guess.

## The decision that mattered

There was an earlier temptation to fold the step count into a response body field of a different endpoint, the one that reports execution resources. It was rejected for the same reason the header was chosen: the body field would have forced every consumer of that endpoint to either ignore the new field or version against it, and the endpoint's schema is shared across network versions. The header sidesteps the entire versioning problem. Consumers that want the count read the header. Consumers that do not know it exists are unaffected. That is the property that makes a measurement safely additive.

## The lessons

A measurement is worthless until it crosses a boundary. The step count was computed on every single execution and discarded on every single execution. The entire task was a data flow problem, and the interesting engineering was deciding which boundary to cross at, not the code.

Headers are the right home for response metadata. If a value describes the work that produced a response rather than the response itself, it does not belong in the body schema. The header keeps additive metadata invisible to existing consumers, which is the definition of a non breaking change.

Promote numbers with care. A value that will be consumed by external clients should not risk precision loss in transport. The string representation for the header looks like overengineering until someone's fee calculation comes back wrong by one unit.
