---
title: "Design"
weight: 20
---

# Design {#design}

A summary of [`forge/docs/design.md`](https://github.com/go-vcs/forge/blob/main/docs/design.md),
which is the authoritative text (draft of 2026-11-25) and cites the source it
read for every claim. Where the two differ, the design document wins.

## Status {#status}

Milestone 2 (the admin gRPC API and its CLI; clone and fetch over SSH and smart HTTP) and milestone 3 (OIDC sign-in and the read-only web UI: tree, blob, raw, commits, commit diff, behind `--web-listen` and `--repos-root`) are implemented. The rest is not.

## Three repositories {#three-repositories}

| Repository | Holds |
| --- | --- |
| `gitproto` | Git wire protocol v2 (upload-pack, receive-pack), pkt-line, pack files and indexes. A library with no forge knowledge, fuzzed and benchmarked against `git`. |
| `actions` | Workflow parsing and planning (matrix, `needs`, expressions) and the server side of the gitea-runner protocol. Reusable without the forge. |
| `forge` | The server: storage, accounts, permissions, SSH and smart HTTP transports, web UI, admin gRPC API and its CLI, webhooks, the GARM-facing API. |

## Identity {#identity}

Web sign-in goes only through the go-authn bridge (OIDC), with a client
restricted to the bridge's local accounts. The forge implements no passwords. A
sign-in whose issuer and subject are not bound to an existing account is
refused: it never creates an account and never links to a signed-in session.
SSH push and pull use per-user SSH keys, preferably FIDO-backed `sk-` keys.

## Administration {#administration}

Everything administrative goes through one gRPC API, and the web UI has no admin
page the API lacks. Authentication is a FIDO2 credential on a hardware key with
user verification. Enrolment verifies the attestation against FIDO MDS3 and
accepts only allowlisted AAGUIDs, with no opt-out. Each mutating call carries a
fresh assertion over a challenge that combines a server nonce, the hash of the
exact request message and a TLS exporter value of the connection, so an
assertion is valid for one request on one connection.

## Git transport {#git-transport}

SSH and smart HTTP, protocol v2 only for fetch. Push policy is decided
in-process by parsing the receive-pack command list and pack with `gitproto`
before any ref moves: there are no server-side hooks and no internal callback
token.

## Actions {#actions}

The forge serves the gitea-runner protocol so that the existing runner runs our
jobs unchanged. Registration tokens are single-use, short-lived and bound to one
runner name; a job's token is scoped to its repository and job and revoked when
the job ends; secrets are never sent to runs triggered from forks. Runners are
created by GARM, first through its `gitea` forge type, later through a `govcs`
type to be contributed to GARM.

## Milestones {#milestones}

1. `gitproto`: pkt-line, protocol v2 capability advertisement, `ls-refs` and `fetch`, with fuzzers and a benchmark against `git upload-pack`.
2. `forge`: storage layout, accounts, admin gRPC API with FIDO2 assertions and its CLI; SSH and HTTP clone. Implemented: fetch is protocol v2 only and read only (`git-upload-pack`); push is refused with a message naming milestone 4. Repositories live at `<repos-root>/<owner>/<name>.git`; a reader must be the owner, or the repository must hold a `git-daemon-export-ok` file (anonymous HTTP read is public repositories only). A missing and a forbidden repository give the same answer. SSH uses the per-user keys of the admin API and runs `git-upload-pack` only (no shell, pty, forwarding or agent); HTTP is TLS 1.3, anonymous, and refuses protocol v0/v1, the dumb protocol and receive-pack with a message that says why. Each transport bounds request size, pkt-line count, time per request, and concurrent sessions per account (per address over HTTP) and in total.
3. `forge`: OIDC sign-in through the bridge, read-only web UI. Implemented, in three slices:
   - **OIDC client admin** (forge#19): `SetOIDCClient` (FIDO2 per-call assertion) and `GetOIDCClient`, `forgectl oidc show|set-client`. The client secret is sealed with AES-256-GCM (`internal/secretbox`) before it is stored; the key comes from `--secrets-key` or the systemd credential `secrets.key`, and without a key a secret is refused, never stored in clear. The secret is read from a no-echo prompt or `--secret-fd`, never from argv, the environment or a URL, and is never returned (`has_secret` only).
   - **Sign-in** (forge#20): `--web-listen` serves over TLS 1.3 only; authorization-code flow with PKCE S256, state, nonce and the RFC 9207 issuer check, done by `go-authn/rp`. The only key is `(issuer, subject)`: a sign-in never creates an account, never links an identity to a live session, and never reads an e-mail address or a username. Every refusal is the same 403 page. Sessions are server side, with a `__Host-` cookie (Secure, HttpOnly, SameSite=Lax) whose SHA-256 alone is stored; they end by idle time (30 min), by age (12 h), and with the account or the binding.
   - **Read-only web UI** (forge#21): home, tree, blob, raw, commit log and commit diff, with a strict CSP (no inline script or style). A repository is visible to its owner, or to anyone when it holds `git-daemon-export-ok`; anything else is a 404. Every git command runs with a sanitized environment, a deadline, a bounded output and bounded concurrency. The path, ref and escaping parsers are fuzzed.
   - There is no performance table: it was skipped because the machine load was above the limit of 8. Each page runs one to four git processes, as the reference commands would.
4. `gitproto` and `forge`: receive-pack with in-process push policy.
   - **A1** (agent-oriented layer, see [Agent VCS]({{< relref "agent-vcs" >}})): read and write without a clone, stable change ids and immutable revision refs, server-signed commits, compare-and-swap ref updates, idempotent writes, agent-session identities.
5. `actions`: workflow planning, runner protocol server, GARM `gitea` type.
   - **A2**: CI verdicts bound to the exact commit, server-side merge gated on them and on a FIDO2-signed human approval for protected refs, event stream. **A3**: path-scoped leases, path permissions. **A4**: the forge backend of `forge-agentd`. **A5**: atomic multi-repository change sets (at most 64 repositories).
6. Pull requests and reviews in the web UI.
7. GARM `govcs` forge type; artifacts.

**A0**, the local daemon `forge-agentd` with a GitHub backend, does not depend on
the milestones above and comes first (see [Agent VCS]({{< relref "agent-vcs" >}})).
