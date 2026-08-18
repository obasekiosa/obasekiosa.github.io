---
title: Matching the reference admin_peers response
date: 2024-11-29 12:30:00 +0100
categories: [Engineering, Ethereum]
tags: [ethereum, rpc, p2p, interoperability]
---

Every Ethereum client exposes `admin_peers`, and every client that wants to be a drop in replacement for the reference client has to answer it the way the reference client does. The gap was one boolean field: whether a peer connected to us or we connected to them. The field was small. The plumbing it required was not.

## Why peer direction matters

Operators debug their networks with `admin_peers`. If half the peers disappear, the question is always the same: who initiated those connections, and which side is failing? A node behind a restrictive NAT can only accept connections, it cannot dial out. A node that only accepts shows up as a dead weight in a network health check.

The reference client answers this with an `inbound` boolean on every peer entry. Ours did not. The response listed peers, their names, their enodes, their capabilities, but not the direction of each connection. Operators doing cross client comparisons got a schema mismatch, and any tooling written against the reference format could not parse our output.

## What a peer entry actually contains

The shape of the response, matching the reference:

```
{
  "enode": "enode://...",
  "id": "16Uiu2HA...",
  "name": "Geth/v1.13.0/linux-amd64",
  "caps": ["eth/66", "snap/1"],
  "network": {
    "localAddress": "192.168.1.10:30303",
    "remoteAddress": "81.2.69.142:46510"
  },
  "protocols": {
    "eth": { "version": 66, ... }
  },
  "inbound": true
}
```

Every field is already somewhere in the node. The peer manager knows the address of every connection, the identity, the protocol versions. What it does not expose, until you look, is the direction. The connection object has it, the dialer records it, but nothing ever asked the question, so it died at the layer boundary.

## The plumbing

The honest work was not adding the field. It was the questions that had to be answered along the way:

Where does the connection direction live? On the transport connection itself, in the dial code, recorded at the moment the connection is established. The RPC layer never touches transport objects, and it should not. So the direction had to be captured where the connection is created and carried up through the peer record into the RPC response model.

What does the response model need? A field, a type, and a wire contract. The field on the peer DTO, the type that serializes to a JSON boolean, and a contract test that pins the exact JSON shape so a future refactor cannot silently break the schema again.

Who tests it? The unit test for the response assembly, the integration test that starts two nodes, one dialing the other, and asserts the direction comes out right. The integration test is the one that catches real bugs, because a locally assembled response can always claim whatever it wants.

## What the docs needed

An RPC response is documentation too. The endpoint page listed every field with its type and meaning. It described the peer object and its network sub object. It did not mention direction, because the field did not exist. The docs and the schema were updated together, and that pairing is the part people skip. A field that ships without a docs update is a field nobody knows exists, which is the same as a field that does not exist for the people who read docs instead of code.

## The lessons

Schema parity is a feature. Cross client tooling assumes the reference format, and every deviation costs users a parser or a patch. The field was one boolean, and the effort was one boolean's worth of plumbing plus the discipline to pin it with tests and document it.

Small responses hide big questions. The interesting part of this task was never the JSON, it was deciding where the direction information lives and carrying it across three layers without leaking transport details into the RPC model. That is where the design is, in the boundary crossings.

The integration test is the truth. A unit test proves the response assembler can write the field. The two node test proves the node actually knows which way the connection went. Always test the boundary where the data is created, not just where it is serialized.
