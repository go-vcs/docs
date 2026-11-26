---
title: "Agent VCS"
weight: 25
---

# Agent VCS {#agent-vcs}

A summary of [`forge/docs/agent-vcs.md`](https://github.com/go-vcs/forge/blob/main/docs/agent-vcs.md),
the authoritative text (design approved 2026-11-25). Where the two differ, the
source document wins.

**Design stage, not implemented.** Nothing on this page runs yet. It extends the
[design]({{< relref "design" >}}) and inherits all of its invariants.

## Why {#why}

The forge's first users are not only people: on one build machine about 25
coding-agent sessions run at once against many repositories, through plain `git`
and the GitHub API. The design answers frictions measured there: worktrees and
clones lost or duplicated, a full commit id rebuilt from a 7-character prefix, a
CI verdict left on the commit before an amend, concurrent merges into one
branch, whole-repository leases that lapse silently, many CI pollers sharing one
rate-limit budget, and dependent changes spread over many repositories.

## Constraints {#constraints}

1. git stays the storage and interchange format: `git clone`, `git fetch` and
   mirroring to GitHub keep working unchanged.
2. The design's invariants hold: no secret in argv, environment or URL; every
   untrusted parser fuzzed; every input bounded.
3. A branch moves only to the exact commit CI judged green. No "merge, then let
   an agent fix it".
4. Every write is idempotent under retry with the same request id.

## The layer {#the-layer}

- **Change ids.** A stable id for a change across amends and rebases, with
  immutable revision refs.
- **CI verdicts bound to the exact commit.** A verdict names the full object id
  it judged; a merge names the revision it intends to merge.
- **Server-side merge.** `merge-tree` computes the result without a checkout,
  and the ref moves by compare-and-swap, so concurrent merges need no manual
  "update branch".
- **Path leases.** Leases on paths with fencing epochs, replacing
  whole-repository, silently lapsing leases.
- **Reads and writes without a clone.** Trees, blobs, diffs and
  `CreateChange` at any revision, over gRPC.
- **Atomic multi-repository change sets.** Joint CI and a two-phase merge, at
  most 64 repositories.
- **Human approval.** Merges to protected refs need a FIDO2-signed approval.
- **Server-signed commits.**

## Local daemon: forge-agentd {#forge-agentd}

One per user per machine, and the only process that talks to the server. It
serves local sessions over a Unix socket (peer uid checked), keeps a
content-addressed object cache (an object is admitted only after its id is
recomputed from its content), can `Materialize` a sparse checkout into a
directory the caller owns, and fans one upstream event stream out to local
subscribers, so N waiters cost one stream, not N pollers. Its first backend is
GitHub (milestone A0), which does not wait for the forge.

## Milestones {#milestones}

- **A0**: `forge-agentd` with a GitHub backend: local protocol, verified object
  cache, `Materialize`, one poller per machine, `WaitVerdict` answered from the
  stream. Operations that need the forge answer `UNSUPPORTED_BY_BACKEND`, never
  a weaker imitation. Comes first.
- **A1** (after forge milestone 4): reads without a clone, `CreateChange`,
  revision refs, server-signed commits, compare-and-swap ref updates, idempotent
  writes, agent-session identities.
- **A2** (after milestone 5): verdicts, merge gated on them and on signed human
  approval, event stream.
- **A3**: leases with fencing epochs, path permissions.
- **A4**: the forge backend of `forge-agentd`.
- **A5**: atomic multi-repository change sets.

## Inspiration {#inspiration}

This design is inspired by Anthropic's account of its internal agent-oriented
source control. Scion is Anthropic's internal system and has not been released;
go-vcs has no affiliation with Anthropic and is not endorsed by it. The agent
layer is an independent design on top of git.

- "Redesigning source control at Anthropic", Anthropic Engineering, 2026-10-13
  (updated 2026-10-15): <https://www.anthropic.com/engineering/redesigning-source-control>.
  It also mentions Antique, Anthropic's code review tool, which is not described
  in a post of its own.

Taken from it: server-side reads and writes without a clone, stable change
identity, server-side merge gated on CI for the exact commit, path leases, one
local daemon, atomic multi-repository changes. Not taken: merge-then-fix,
Spanner, FUSE checkouts.
