---
title: "forge-agentd"
weight: 26
---

# forge-agentd {#forge-agentd}

`forge-agentd` is the first implemented slice of [Agent VCS]({{< relref "agent-vcs" >}})
(milestone A0): a local daemon with a GitHub backend, and `forge-agent`, the
small client agents call. Source and releases:
[go-vcs/agentd](https://github.com/go-vcs/agentd).

## What it gives an agent {#guarantees}

- **No token in the agent.** The daemon reads one token from a file (mode 0600,
  never from argv, environment or URL); agents talk to it over a unix socket
  (directory 0700, socket 0600, peers of another uid dropped).
- **A verdict bound to one exact commit.** Object ids are full (40 or 64
  lowercase hex); an abbreviation is refused, never expanded. Only the newest
  run per (workflow, event) counts, a run with zero jobs is never green, a null
  conclusion or an unknown job count is pending, and an observation about
  another commit is refused.
- **One poller per commit.** Any number of waiters on the same commit cost one
  upstream poll loop, so they share one rate-limit budget.
- **A gated merge.** `merge` succeeds only if the commit named is still the pull
  request's head and its verdict is green; otherwise `REF_MOVED` (naming the
  current commit), `VERDICT_RED` or `VERDICT_PENDING`. GitHub's own `sha`
  precondition closes the remaining race.
- **Idempotent writes.** A 32-hex request id replays the first answer; the same
  id with another body is `REQUEST_ID_REUSED`.
- **Structured errors.** `{"error":{"reason":"...","retryable":...}}`, and a
  per-session identity from `hello`.

## Use {#use}

```
forge-agentd --token-file ~/.github-token
forge-agent --label build-7 verdict go-vcs/forge <40-hex-commit>
forge-agent --label build-7 wait    go-vcs/forge <40-hex-commit> --timeout 20m
forge-agent --label build-7 events  go-vcs/forge <40-hex-commit> --timeout 30m
forge-agent --label build-7 merge   go-vcs/forge 6 <40-hex-commit>
forge-agent --label build-7 materialize go-vcs/forge <40-hex-commit> work/1 --path cmd
```

Exit codes: 0 green or done, 1 error, 3 red or refused by the gate, 4 pending
or timed out. `merge` prints its request id on stderr so a retry can reuse it
with `--request-id`.

## Status {#status}

Done: verdicts, waits, gated merge, idempotency, sessions, the socket,
structured errors, fuzzed parsers, the verified object cache (`blob`, `tree`)
in memory and on disk (`--cache-dir`), `Materialize` with a bulk fetch path, the
keychain credential option (`--token-keychain`) and the event stream. Not done:
a push feed from GitHub (GitHub has none, so the hub polls), the timings of
`Materialize` against `git clone` (deferred: the machine load was above 8).
Leases, changes and change sets answer `UNSUPPORTED_BY_BACKEND` until the forge
backend exists.

Windows gap of the disk cache: on Unix the cache directory is refused unless it
is mode 0700. On Windows Go reports every directory's mode bits as 0777, so that
check is skipped there and the directory's access control list is not read: the
privacy of the cache directory is not verified on Windows.

## Objects and checkouts {#objects}

Every object the daemon returns is verified: its id is recomputed from the bytes
GitHub sent, and a mismatch is `OBJECT_MISMATCH`, nothing kept. The cache is
bounded in entries and bytes.

`forge-agent materialize <repo> <commit> <dir> [--path p]...` writes a commit's
tree, or only the paths given, into a new directory under the daemon's
`--materialize-root` (0700, never under the temporary directory). Writes go
through a root scoped to that directory; `..`, absolute and empty elements,
`.git` in any case, duplicate names and symlinks that escape are refused; files,
bytes, depth and concurrency are bounded; a failure removes the output.

## Events {#events}

`forge-agent events <repo> <commit>` prints one JSON line per change of a
commit's verdict and ends on the final one. Streams and waiters share the one
poll loop of their commit. The semantics are level-triggered: observations
between two polls coalesce, and the final verdict is never skipped. Counted, not
timed: 13 waiters and 13 streams on one commit made 21 upstream reads where 26
independent pollers made 520 over the same window.
