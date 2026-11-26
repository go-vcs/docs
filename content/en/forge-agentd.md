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
forge-agent --label build-7 merge   go-vcs/forge 6 <40-hex-commit>
```

Exit codes: 0 green or done, 1 error, 3 red or refused by the gate, 4 pending
or timed out. `merge` prints its request id on stderr so a retry can reuse it
with `--request-id`.

## Status {#status}

Done: verdicts, waits, gated merge, idempotency, sessions, the socket,
structured errors, fuzzed parsers. Not done: the verified object cache,
`Materialize`, a push event stream (the hub polls), a keychain credential
option. Leases, changes and change sets answer `UNSUPPORTED_BY_BACKEND` until the
forge backend exists.
