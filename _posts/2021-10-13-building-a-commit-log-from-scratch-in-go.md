---
title: Building a commit log from scratch in Go
date: 2021-10-13 11:23:44 +0100
categories: [Engineering, Distributed Systems]
tags: [go, distributed-systems, storage]
---

Every serious distributed system I've touched has a log hiding somewhere inside it. Kafka is a log. Ethereum is, if you squint, a very expensive log with strong opinions about who gets to append. Databases replicate by shipping their logs around. So at some point I decided to stop squinting and build one myself: [slog](https://github.com/obasekiosa/slog), a commit log service in Go.

The rule I set for myself was simple. No skipping layers. Start at the bytes on disk and work up until there's a network API. This post walks through the storage engine, bottom to top.

## The store: just bytes, appended

The lowest layer is the store, a wrapper around a plain file. Every record goes in the same way: write the length of the payload as an 8 byte big endian integer, then write the payload itself.

```go
pos = s.size
binary.Write(s.buf, enc, uint64(len(p))) // 8 byte length prefix
w, err := s.buf.Write(p)                 // then the payload
```

That length prefix is doing a lot of quiet work. Reads don't need delimiters or escaping, they just read 8 bytes, learn the size, and read exactly that much. Writes go through a buffered writer so lots of small appends don't each pay for a syscall. The store hands back the position where the record landed, and that position is all you ever need to find it again.

Appends only. Nothing is ever updated in place. This one constraint is what makes everything above this layer simple.

## The index: mmap and 12 byte entries

Reading a record by position is cheap, but users don't think in byte positions, they think in offsets. Record 0, record 1, record 42. So the next layer is an index that maps offsets to positions.

Each index entry is 12 bytes: a 4 byte relative offset and an 8 byte position in the store. The index file is memory mapped with mmap, so a lookup is just arithmetic on what looks like a byte slice. No read syscalls, no deserialization step, the OS page cache does the heavy lifting.

There's one funny consequence of the mmap approach. You can't grow a memory mapped file easily, so the file gets truncated up to its maximum size when the index opens, and truncated back down to its real size when it closes. Which means a crash leaves garbage at the end of the index, and recovery has to deal with that. Append only designs don't remove failure handling, they just move it somewhere you can reason about it.

## Segments: because one file can't grow forever

A single store file that grows without bound is a problem. You can't delete old data, and recovery time scales with the whole history. So the log gets chopped into segments, each one a store file plus an index file, named by the base offset of the first record it holds.

The segment ties the two layers together. On append it writes the record to the store, then writes the offset to position mapping into the index. When either file hits its configured size limit, the segment is full and the log rolls over to a fresh one.

## The log: the part users actually see

The top layer holds the list of segments and knows which one is active. On startup it scans the directory, parses the base offsets out of the file names, sorts them, and rebuilds the segment list from what's on disk. That's the whole recovery story at this layer, and it falls out of the naming scheme almost for free.

## Serving it: gRPC with mutual TLS

On top of the storage engine sits a gRPC service with Produce and Consume RPCs, plus streaming variants of both. ConsumeStream is where the log model shines: a client can sit on the stream and receive records as they arrive, which is the exact shape you want for building anything reactive on top.

Access is locked down with mutual TLS, so the server and client both prove who they are, and an ACL based authorizer decides who can produce and who can consume.

## What I actually learned

The book Distributed Services with Go was my map for this build, and the thing it taught me is that none of these layers is clever on its own. A length prefix, an mmap, a file naming scheme. The design is in how the constraints stack: append only makes indexing trivial, indexing makes segments cheap, segments make recovery boring. Boring recovery is the whole prize.

Next up for slog is the actual distributed part: replication and consensus so multiple nodes can agree on the log. Given how much of my time at Nethermind went into consensus, I have opinions. That's the next post.
