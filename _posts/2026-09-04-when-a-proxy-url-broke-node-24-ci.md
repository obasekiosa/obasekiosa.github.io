---
title: "Fixing a GitHub Actions Proxy Error"
date: 2026-09-04 12:00:00 +0100
categories: [Engineering, DevOps]
tags: [github-actions, ci, node, proxy, ssh]
description: "How a GitHub Actions runner upgrade exposed a SOCKS proxy mismatch, and how an HTTP bridge kept PR comments working."
---

A CI failure arrived with a warning that Node 20 was being deprecated. The obvious reading was that the workflow needed to be moved to Node 24. That was true, but it was not the whole failure.

The real error came from `actions/checkout` during post-job cleanup:

```text
InvalidArgumentError: Invalid URL protocol: the URL must start with http: or https:
```

The workflow used an SSH dynamic tunnel to avoid a GitHub API IP ban when Buf posted its pull request comment. The tunnel was a SOCKS5 proxy on `127.0.0.1:1080`, and the workflow exported it directly as `HTTP_PROXY` and `HTTPS_PROXY`:

```text
HTTPS_PROXY=socks5://127.0.0.1:1080
```

That had worked until the runner began executing JavaScript actions with Node 24. The newer proxy client tried to parse the SOCKS URL as an HTTP proxy URL and rejected it. The proxy was still necessary. Its shape was the problem.

## The bridge

The fix keeps both requirements:

1. The SSH tunnel still provides a different public IP through SOCKS5.
2. A small local HTTP CONNECT bridge listens on `127.0.0.1:3128` and forwards each connection through the SOCKS tunnel.

The workflow now exposes this valid URL to JavaScript actions:

```text
HTTPS_PROXY=http://127.0.0.1:3128
```

Git also uses the bridge. Checkout can clean up normally, Buf can reach the GitHub API through the alternate IP, and the PR comment can still be created or updated.

## The lesson

The Node 20 message was a migration warning, not the root cause. Node 24 changed which proxy implementation handled the request, and that exposed an assumption hidden in the workflow: a SOCKS proxy URL is not automatically an HTTP proxy URL.

The final change upgraded `actions/checkout` to v5 and `actions/setup-go` to v6, then added the bridge before checkout starts. The result is a Node 24 workflow that keeps the proxy path the project actually needs.

The fix is in [pull request #11](https://github.com/uLesson-Education/lms-grpc-contracts/pull/11). The useful debugging habit is simple: when a platform warning and a stack trace arrive together, treat them as separate clues until the failing network path is clear.
