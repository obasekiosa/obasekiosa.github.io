---
# the default layout is 'page'
icon: fas fa-code
order: 4
---

Stuff I've built or contributed to. Some of it is open source, some of it is private product work I'm still cooking. Ask me about those ones.

## Featured

### slog, a distributed log system in Go

A commit log service built from first principles. An append only store with length prefixed binary records, a memory mapped index for fast offset lookups, size bounded segments, and a gRPC API (produce, consume, plus bidirectional streaming) secured with mutual TLS and ACL based authorization.

[github.com/obasekiosa/slog](https://github.com/obasekiosa/slog)

### Nethermind and Juno, production blockchain clients

Protocol level contributions to two production clients at [Nethermind](https://nethermind.io):

- **[Juno](https://github.com/NethermindEth/juno)** (StarkNet client, Go and Rust): led the sequencing subsystem redesign, implementing a Tendermint style consensus algorithm to decentralize StarkNet's sequencer. Also fixed VM execution bugs in Rust.
- **[Nethermind](https://github.com/NethermindEth/nethermind)** (Ethereum client, C#): root caused a concurrency bug in fee history retrieval and designed a custom caching algorithm for a 10x latency win. Optimized an archival log index with Elias-Fano compression for a 40% gain. Contributed to [dotnet-libp2p](https://github.com/NethermindEth/dotnet-libp2p) with QUIC benchmarking and Gossipsub 1.1/1.2 work.

### Handcraft, an AI markdown editor *(private, in progress)*

A document editor where AI helps you write and edit incrementally. Every paragraph is a node in a graph, so edits branch rather than overwrite and no version is ever lost. You control exactly what context the AI sees for each generation. Built on Next.js 16.

### Sauce, non linear LLM chat *(private, in progress)*

A chat tool where conversations are nodes in a graph instead of a single thread. Branch from any point to explore a new direction, and each branch inherits only its ancestor chain as context. Streaming responses, persistent graph state, React Flow canvas. Built on Next.js 16.

### sendme, a P2P logistics engine *(private, in progress)*

The engine for peer to peer logistics. Send and track anything, anywhere, in real time. Elixir/OTP.

## More

- **whot**: the classic Nigerian card game, implemented in Elixir ([repo](https://github.com/obasekiosa/whot))
- **chain-lens**: a Bitcoin transaction parser (CLI) plus web visualizer in Go, from the Summer of Bitcoin 2026 developer challenge
- **p2p-arbitrage-data-aggregator**: a data pipeline aggregating P2P market data for arbitrage analysis, in Python ([repo](https://github.com/obasekiosa/p2p-arbitrage-data-aggregator))
- **cwlviewer**: maintained the Common Workflow Language's workflow visualization web service, raised test coverage from 60% to 94% and led a MongoDB to PostgreSQL migration ([repo](https://github.com/common-workflow-language/cwlviewer))
- **Raytracers**: the same raytracer three times, in [Java](https://github.com/obasekiosa/raytracer), [Rust](https://github.com/obasekiosa/simple-ray-tracer), and [TypeScript for the browser](https://github.com/obasekiosa/raytracer-web)
- **nand2tetris**: a complete computer simulated from basic logic gates all the way up to an operating system ([repo](https://github.com/obasekiosa/nand2tetris))
- **vlsi-cad**: a logic circuit synthesizer in Python ([repo](https://github.com/obasekiosa/vlsi-cad))
- **Robotics**: differential drive and robot arm simulations in ROS/Gazebo, plus bipedal robot controllers as hybrid automata in MATLAB
