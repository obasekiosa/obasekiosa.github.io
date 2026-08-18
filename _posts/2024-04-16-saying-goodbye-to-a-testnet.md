---
title: Saying goodbye to a testnet
date: 2024-04-16 11:00:00 +0100
categories: [Engineering, Ethereum]
tags: [ethereum, testnet, deprecation]
---

Goerli was the last public proof of work testnet on Ethereum. It survived the merge, carried on as proof of stake, and then the community agreed on its retirement: Sepolia for application developers, Holesky for staking infrastructure. The interesting work was not the announcement, it was removing a network from a client without breaking the people still running it.

## The trap of a retired network

A testnet that no longer reaches consensus is not like a feature you delete. It is a destination. Nodes across the internet have chain specs pointing at it, bootnodes advertising it, and config files naming it. When the network stops, those nodes keep trying. They attempt sync, fail, retry, and fill logs with connection errors that look like networking problems when the real problem is that the destination no longer exists.

The worst outcome is a node silently trying to sync a ghost forever. The second worst is a hard error with no hint. Users who were mid test get no signal about why, what changed, or where to go.

## What phasing out actually means

Removing Goerli support was a checklist, not a single diff:

- **The chain spec.** Genesis configuration, bootnodes, checkpoint sync endpoints, all of it goes. This is the code that knows how to join the network.
- **Chain ID detection.** A node pointed at a Goerli genesis must fail loudly, and the failure must say what happened. Chain ID 5 is a name the ecosystem recognizes.
- **Config migration.** The network selection flag that offered Goerli as a choice drops out of the schema. Anyone still selecting it gets a clear rejection, not a silent fallback.
- **Docs and guides.** Every page that teaches a Goerli command becomes a trap for beginners. They all get rewritten to point at the right successor.
- **Tests and integration.** Any test harness bootstrapped on Goerli fixtures needs a new home, because the fixture data stops being meaningful.

## The failure mode that matters

The design constraint I kept coming back to: retired is a state that has to be reported, not a state that can be silently occupied.

The client keeps a small registry of known networks. The removal turned the Goerli entry into a sentinel: the chain ID is recognized, the name is printed, and the node exits with a message explaining the retirement and naming the replacement testnets. That costs a few lines of code and saves an enormous amount of user confusion, because the alternative is weeks of support threads that start with "my node will not sync."

A retired network deserves the same courtesy as a dead endpoint: a clear, actionable error with a pointer to the alternative.

## The lessons

Deprecation has a geography. A feature lives in code, but a network lives in other people's configs, docs, scripts, and CI pipelines. Removing the code is the smallest part of the job. The map of where the network is referenced decides how long the transition takes.

Loud failures beat graceful ones when the state is terminal. Most engineering instincts push toward graceful degradation, but a network that will never come back is not a degradation case. It is a hard stop, and the only graceful thing is the quality of the error message.

Pick the successor in the announcement, not in the FAQ. The reason Goerli's retirement was clean was that the ecosystem had already agreed on the destination. A testnet shutdown without a successor is just a hole.
