---
title: A baseline Tendermint state machine in Go
date: 2024-05-15 14:30:00 +0100
categories: [Engineering, Distributed Systems]
tags: [go, consensus, byzantine, p2p]
---

The Tendermint paper is a choreography. Height by height, round by round, validators move through propose, prevote, and precommit, carrying locked values forward, respecting valid rounds, and only ever committing a value that enough of the network agreed on. Before wiring that into a network, into a database, into a real node, I built the state machine alone, as a baseline: the transitions, the state, and the boundaries where the rest of the system plugs in.

## The design split

The core insight from the start was that the consensus logic and the environment must not know each other. The state machine needs three things from the outside world, no more:

**A proposer.** Tells the machine who proposes at a given height and round, and what value to propose when it is our turn.

**A gossiper.** Carries messages in and out. The machine never opens a socket, it submits messages for broadcast and receives messages from the network.

**A decider.** Stores decided values. When the machine commits, the decider keeps the result, and the chain above it does whatever a chain does with a committed block.

Everything else, the transport, the validator set bookkeeping, the persistence, lives behind those boundaries. That is the whole architecture, and it is the part that survives contact with production, because it makes the consensus core testable with fake adapters and replaceable in a real node.

## The state

The state is the paper's state, verbatim:

```go
type State struct {
    step             Step
    currentHeight    HeightType
    round            RoundType
    lockedValue      Proposable
    lockedRound      RoundType
    validValue       Proposable
    validRound       RoundType
    isFirstPreVote   bool
    isFirstPreCommit bool
}
```

Height, round, and step are the position in the dance. Locked value and locked round are the safety memory: once we prevote a value and see the network agree on it, we do not switch lightly. Valid value and valid round are the liveness memory: a value that got a prevote majority in an earlier round remains proposable so progress is not lost.

The two flag fields are a derivation not handed over by the paper. The paper guards its actions with implicit conditions like "while step is propose". The flags make those guards explicit, which matters for a baseline, because it turns the paper's prose conditions into testable state.

## The transitions

Every transition is a pure function of the current state and the incoming message. The step discipline is strict:

- A proposal from the designated proposer at our height and round, while we are in propose, earns a prevote if the value is valid and consistent with our lock, and a nil prevote otherwise.
- A prevote with a quorum level moves us to precommit, carrying the agreed value as the new locked value.
- A precommit with a quorum level triggers the decision, resets the state, and starts the next height from round zero.
- A proposal that skips forward in round moves the machine along, because the machine that falls behind must be able to catch up to where the network already is.

The transition functions are small and the state copy on every transition is deliberate. A transition that mutates state in place makes every test a time bomb. A transition that produces a new state makes the test a comparison: build the expected state, run the transition, diff.

## The message discipline

Messages carry a type, a height, a round, a value, and a sender. The machine checks senders against the proposer for the given height and round before trusting proposals. It checks heights so stale messages from an old height cannot corrupt a new one. It checks rounds so a message from a future round either advances the machine or is ignored.

The baseline made an explicit scope decision at the aggregation boundary. Message levels carry the quorum information into the machine, while the counting of votes across the validator set, the derivation of 2f + 1 from actual voting power, lives at the adapter boundary. The state machine reads the level and acts on it; the boundary owns the math. That split keeps the transition logic pure and gives the counting a single home.

## The tests

The test suite is a transition table: construct a state, feed it a message, assert the resulting state and the broadcast side effects. Start transitions, first vote handling, locked value consistency, round advancement, decision and reset. The tests that earn their keep are the ones that pin the order of operations, because consensus bugs live in ordering: the message that arrives before the step is ready, the vote that lands after the state moved on.

## The lessons

A baseline is a boundary map. The value of the work is not the happy path working, it is knowing exactly where the consensus logic ends and the rest of the world begins. Every production system built on this will thank the boundary, because the transport and the persistence will change and the core will not.

Model the paper, then add what it omits. The flags for first vote semantics are the best piece of the design and they are not in the paper, they are what the paper means when it says "while step is propose". The paper describes the dance; the baseline's job is to make every condition explicit enough to test.

Test transitions as values. Copying state on every transition looks wasteful and is the cheapest insurance in the system. A transition is a pure function, and pure functions are testable by comparison, which is the only kind of testing that survives consensus complexity.
