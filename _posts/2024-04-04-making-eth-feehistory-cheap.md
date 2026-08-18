---
title: Making eth_feeHistory cheap with a history cache
date: 2024-04-04 10:30:00 +0100
categories: [Engineering, Ethereum]
tags: [ethereum, rpc, caching, performance]
---

A wallet is nothing but a fee oracle with opinions. Every wallet UI in the ecosystem polls `eth_feeHistory` to draw its fee charts, and every poll was triggering a fresh scan of hundreds of blocks. The fix was a cache that only works because of one lucky property of the data: history is immutable.

## What the endpoint does

`eth_feeHistory` returns two things for a range of blocks:

- `baseFeePerGas` for each block, which is cheap to produce because it sits in the block header.
- `rewardPercentiles`, the gas price at the given percentile of each block's transactions, which is expensive because you have to walk every transaction in the block, read its effective gas price, and compute the distribution.

The client computes the rewards on demand, every time, across the whole requested range. A wallet asking for 200 blocks of history with five percentiles triggers five sorts per block, tens of thousands of transaction reads, per request, per wallet, all the time.

The endpoint showed up in profiling as a hot path, and it is a very user visible one. Fee UIs poll it on a timer. Slow calls mean broken charts and angry users.

## The lucky property

A block's fee data is fixed the moment the block is mined and confirmed. No one can change the transactions inside it, the gas prices they paid, or the base fee that was charged. History never changes.

That makes `eth_feeHistory` the ideal cache target. The past cannot go stale. There is no time to live, no invalidation clock, no drift problem. The only thing that can invalidate a cached block is the chain itself replacing it with a different block at the same height, a reorg, and those are rare and detectable.

## The design

Two layers, both keyed by content, not height.

**The block layer.** For each block, cache the reward distribution per percentile set. The key is the block hash plus the requested percentiles, never the block number. Heights are ambiguous, hashes are not. If the chain reorgs and a different block appears at the same height, the new block has a different hash, so it gets a different cache entry and the old one ages out. No invalidation logic needed, the key does the work.

**The range layer.** Cache the assembled response for a full range request, keyed by the range edges and the percentiles. This turns the wallet's identical repeated poll into a single lookup after the first call.

The cache needs two policies: a size bound and an eviction order. Percentile lists can be large, so I bounded the cache by bytes rather than entries, and evicted least recently used. A fee history cache is a good fit for LRU because wallets tend to query the same recent window over and over.

## The reorg edge case

Heights are safe because of the hash key, but there is one more subtlety. A reorg swaps blocks at the tip, so the newest cached range may now be wrong even though every block in it is still canonical. The fix is to revalidate the range edge when the chain head advances: if the newest block hash in the cache no longer matches the canonical block at that height, drop the cached response. One cheap hash comparison per head update keeps the cache honest without any per block bookkeeping.

## Concurrency

RPC servers are multi threaded and wallets are impatient, so two identical requests can arrive in the same millisecond. I used double checked insertion per block: compute under a per block lock, publish the entry, and let concurrent callers wait on the first computation instead of duplicating it. The contention is negligible because the expensive part is CPU bound work, not the lock.

## What changed

The endpoint stopped appearing in profiling. The first wallet request pays the full scan, and every identical poll after that is a cache hit. The percentile computations that used to run thousands of times a minute now run once per block per percentile set.

## The lessons

Cache history, not futures. The trick that makes this cache trivial is the immutability of the underlying data. If the data could change, the design would be full of timeouts and invalidation races. Because it cannot, a hash keyed LRU is the whole answer.

Key by content, not by position. Block numbers look like natural cache keys and they are the wrong thing. Position is a lie that reorgs can correct. Content is the truth.

Profile before caching. The fix was only obviously right because profiling showed the endpoint as a hot path with a specific shape of repeated calls. The same cache on a cold endpoint would have been dead weight. The data said cache, the profile said where.
