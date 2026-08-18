---
title: A double dispose that was eating sockets
date: 2024-04-08 12:00:00 +0100
categories: [Engineering, .NET]
tags: [dotnet, rpc, memory, debugging]
---

The bug report was vague in the way memory bugs always are: a JSON RPC server threw exceptions, intermittently, only under load, and only on some requests. The root cause was a resource being disposed twice, and the fix was one line that changed how we thought about ownership.

## The symptom

Each request wrapped the underlying connection in a request scope. That scope owned the response stream, the request context, and a handful of pooled buffers. The code looked sensible: create the scope, do the work, release the scope when done.

Under load the server started throwing exceptions mid request. The errors pointed at disposed objects: streams being written after close, contexts being read after release. The failures were nondeterministic. Load tests passed for minutes and then collapsed in a burst.

## The hunt

The first suspect was the obvious one, race conditions on shared state. That was wrong. The scope was request local, no request shared it, and the traces never showed two threads fighting over one object. They showed the same thread, two code paths, each convinced it owned the cleanup.

The second suspect was a leak. Also wrong. Memory was flat, because the scope was being released, in fact it was being released more than once. That is the opposite of a leak and its own kind of bug.

The trace that cracked it: one request path hit an error, an error handler failed fast and released the scope so it could write an error response, and the outer request handler released it again on the way out. The second release returned buffers that were already back in the pool, and the pool handed out corrupted buffers to unrelated requests. The failure landed far from the cause, which is why it was so hard to find.

## Why double dispose is evil

A single dispose is a contract. A double dispose is a contract violation, and pooled resources make it catastrophic. Many types tolerate double disposal because their dispose is a no op after the first call. Pooled buffers are not that type. The first dispose returns the buffer to the pool. The second dispose corrupts the pool's bookkeeping. Some other request allocates the buffer, reads garbage, and the crash points at innocent code.

## The fix

Two parts. The discipline part: only the outer handler owned the scope, and the error path built its own short lived response. Ownership moved to exactly one place.

The defensive part, the line that matters:

```csharp
private void ReleaseScope()
{
    if (_scope != null)
    {
        _scope.Dispose();
        _scope = null;
    }
}
```

Null out the field on release. A second call becomes a no op by construction. The guard converts a whole class of future ownership mistakes into silent no ops instead of corruption, and it costs one line.

## The lessons

Dispose is a lifecycle contract, not a cleanup suggestion. In a language with manual resource management, who owns a resource is a design decision that must be made once and written down, because the runtime will not help you. This bug was a design bug wearing a memory bug costume.

Idempotency is the cheapest insurance. The null check converts the worst class of lifecycle bugs from corruption into a no op. It is not a substitute for getting ownership right, but it is what keeps a future mistake from taking the server down.

Bugs that only appear under load are usually lifecycle bugs. Load changes the timing, and timing changes which interleaving you hit. When a failure is intermittent and load sensitive, stop profiling the hot path and start auditing ownership.
