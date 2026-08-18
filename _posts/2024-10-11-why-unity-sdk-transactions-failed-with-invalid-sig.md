---
title: Why Unity SDK transactions failed with INVALID_SIG
date: 2024-10-11 11:00:00 +0100
categories: [Engineering, Starknet]
tags: [starknet, unity, csharp, signatures]
---

A Unity SDK for Starknet builds and signs transactions on device, in C#, under IL2CPP. When wallets started reporting `INVALID_SIG` on the network and failures to pay fees in STRK, the signature path became the prime suspect. The work was tracing exactly where a correct signature turns into an invalid one.

## What makes a Starknet signature valid

A Starknet transaction is signed with ECDSA over the Stark curve, using a message hash built from the transaction's fields. The verifier, the sequencer, recomputes the message hash from the transaction it received and checks the signature against it. There is no ambiguity tolerated anywhere in that chain. If the signer and the sequencer hash even one field differently, the signature is mathematically fine and categorically invalid.

The hash is a Pedersen based construction over the serialized transaction: the type, the sender, the nonce, the fee, the calls, the chain id, and the domain separator for the network. The chain id matters because a signature over the wrong chain id is a signature over a different message entirely.

## The failure modes

**The missing field.** The SDK computed the message hash over a subset of the transaction fields. Any field the sequencer includes and the SDK omits shifts the hash, and the shift is silent. The transaction serializes, the signature computes, the submission succeeds, and the network answers `INVALID_SIG`. The omission that bit hardest was the fee related fields, because those are exactly the fields that change with the fee path.

**The STRK path.** Paying fees in STRK instead of ETH changes the transaction shape. The fee token participates in the hash, and the v3 transaction format replaced the flat fee with resource bounds, a different structure entirely. A wallet that signed a v1 style hash and submitted a v3 shaped transaction was guaranteed to fail, and the failure mode for STRK payments specifically, "unable to pay", is the same signature mismatch wearing a fee costume, because the signed hash and the submitted shape disagree.

**The serialization layer.** C# big integer handling is the quiet assassin. Field elements are 252 bit numbers, and converting between the felt and the byte representations has two classic failure points: endianness, where the same number serializes to different bytes in big endian and little endian worlds, and zero handling, where a leading zero byte in a field element changes the hash computation on some implementations. Under IL2CPP the runtime changes nothing about the math, but the same C# code path that works in the editor can behave differently under the AOT compiler, which makes editor tested code fail on device.

**The chain id.** The domain separator must match the network the transaction is submitted to. Testnet hash signed and sent to mainnet, or a default value used instead of the actual network id, produces the classic "signature verified locally, rejected remotely" behavior.

## The method

The fix process was a comparison against ground truth, not a code reading:

**Known vectors first.** Every network has published signed transaction examples with their hashes. The SDK's message hash function was tested against those vectors before any wallet was involved. A single vector mismatch locates the bug in minutes, because it isolates the hashing layer from the signature layer.

**Field element round trips.** Every felt in the transaction was exercised through a serialize, parse, reserialize cycle and compared byte for byte. The tests that matter are the ones with edge values: zero, one, and the maximum 252 bit value, because those are the values that expose endianness and zero handling bugs.

**Signature round trip.** A signed message was verified with the SDK's own public key code. A signature that fails its own verifier is a serialization bug in the SDK. A signature that passes its own verifier and fails the network is a message hash mismatch. The two outcomes point at different layers, and the test separates them.

**Cross implementation signing.** The same transaction was signed with a reference implementation and the result compared. When the SDK's signature matches the reference for the same message, the curve math is exonerated and the remaining suspects are the hash and the field encoding.

## The findings pattern

The failures that surfaced traced back to three concrete defects: the message hash omitting the fee related fields of the transaction, the v1 hash structure being used for transactions that carry the v3 shape, and field elements being serialized with an endianness mismatch in one conversion path. Each one is invisible in happy path testing and immediate in vector testing, which is the lesson the whole exercise was built to teach.

## The lessons

Signatures are contracts with the network, not with your own code. The SDK can verify its own signatures forever and learn nothing, because the contract is with the sequencer's hash. Ground truth lives in published vectors and reference implementations, and testing against them is not optional.

The fee path and the signature path are the same path. The STRK payment failures looked like a fee bug and were a signing bug, because the fee fields are part of the signed message. When payments break, recheck the hash before the wallet.

Serialization is cryptography. In an SDK, the felt conversion code is security code. Endianness, zero handling, and exact field inclusion are where every signature bug lives, and they only show up at the edges: the maximum value, the zero value, the network boundary. Test the edges, because the network will.
