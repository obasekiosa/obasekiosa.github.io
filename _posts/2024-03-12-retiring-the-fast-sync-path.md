---
title: Retiring the fast sync path in an Ethereum client
date: 2024-03-12 10:00:00 +0100
categories: [Engineering, Ethereum]
tags: [ethereum, sync, deprecation, configuration]
---

Every long lived client accumulates flags that describe how the world used to work. Fast sync was one of them. Removing it took a plan, not a delete, because in a config driven client, the config file is a public API.

## What fast sync was

When an Ethereum node joins the network it has to agree with everyone else about the state of the chain. That agreement happens during synchronization, and clients have offered several strategies for getting there:

- Full sync. Verify every block and reexecute every transaction from genesis, building the full state tree yourself. Bulletproof, brutally slow for new nodes.
- Fast sync. Download block headers and bodies, skip reexecuting history, then pull a recent state snapshot from peers. Orders of magnitude faster to get to the tip.
- Snap sync. The successor to fast sync, which fetches recent state in parallel with proofs, so you still verify, just not since genesis.

Fast sync was the answer for years. But it had a wart: skipping execution meant trusting peers for the state you bootstrapped with. Snap sync closed that hole, and the ecosystem moved. Clients that kept the old path were maintaining two code paths with one actually being used.

## Why the flag was a problem

The client I worked on exposed this choice as a config option: a boolean that turned the fast block download path on. The flag surface was load bearing. Installations in the field had it written into config files. Automation scripts set it. The docs explained it. Tests covered it.

You cannot just delete a flag like that. A hard removal breaks upgrades for everyone who has the key in a config file, and strict parsers refuse unknown keys. An ignored flag is worse: it parses, it logs nothing, and the user believes they are getting fast sync behavior when the node has quietly routed into a different mode. Silent behavior drift is how support tickets are born.

## The deprecation ladder

I retired the flag in four steps, each one a separate release, so at every point in time the client behaved sensibly for the configurations that existed in the wild.

**Step one, announce.** Mark the option as obsolete in the config schema. Keep reading it, keep honoring it, but emit a warning on first load. One log line, once per process, telling the operator what will change and when. Log warnings work because operators who read logs start migrating before the breakage lands.

**Step two, change behavior.** Fast mode stops being a separate code path and routes into the default sync path. The flag still parses, so no config breaks, but the behavior is the maintained one. This is the sneaky part of the ladder: behavior changes first, parsing changes later. Users who set the flag get the same outcome as users who never knew it existed, which is exactly the convergence you want before a removal.

**Step three, drop the schema entry.** Remove the key from the schema, remove it from the docs, but keep a compatibility shim in the parser that recognizes the old key and logs a migration hint instead of failing. By now the field population has moved, and the shim catches the stragglers.

**Step four, delete.** Remove the shim, the constant, the parser branch, and the tests in one commit. There is no good reason to keep dead parsing code around, and a shim left in place becomes its own documentation problem two years later.

## What the code looked like

The shape of step one, in a C# style config schema:

```csharp
[Obsolete("Fast block sync is no longer supported. The option will be removed in a future release.")]
public bool FastBlocks { get; set; }
```

And the load time warning, emitted once:

```csharp
if (config.FastBlocks)
{
    Logger.Warn("FastBlocks is deprecated and has no effect. " +
                "The node will use the default sync mode. " +
                "Remove the option from your config file.");
}
```

Step four was the satisfying one. The diff removed the flag from the schema, the migration shim, the docs page, and the sync mode branch. What was left was a smaller parser and one less mode to test.

## The lessons

Config is a public API. It outlives the feature it gates, it ships in files you never see, and it is read by scripts you cannot update. Treat every flag like a commitment and every removal like a migration.

Deprecation is a process, not a delete. Warn, change behavior, remove the parser entry, then delete. The ladder exists because at each rung, the client does the right thing for the configurations that actually exist out there.

And the cheapest time to delete a flag is the release after you stop testing the feature it controls. If the mode has no test coverage, no docs traffic, and no config files in the wild, it is already retired. The flag is just the paperwork.
