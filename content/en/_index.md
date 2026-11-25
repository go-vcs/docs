---
title: "Overview"
weight: 10
---

# go-vcs {#go-vcs}

A light, secure Git forge for people and agents, written in Go: repository
hosting, a web UI, an administration API over gRPC, and Actions runners
scheduled through GARM.

**It is early.** The admin gRPC API and its CLI are implemented (milestone 2a).
Hosting, sign-in and the web UI are still design only, and the `actions`
repository holds a licence and a README. This site says which is which; it does
not describe the design as if it were running.

## Why it exists {#why-it-exists}

go-vcs is written to replace a Forgejo install whose security design conflicts
with ours: an OAuth2 client secret that can only be given on a command line or a
web form, server secrets held inline in one configuration file, and OAuth2
auto-registration that ignores the switch meant to disable registration. Each
conflict is cited to the source line it was read in, in the
[design document](https://github.com/go-vcs/forge/blob/main/docs/design.md).

It is designed from its own stated semantics, not for compatibility with
Gitea or Forgejo. The one deliberate compatibility is with existing runners and
runner managers, because that saves building them.

## Invariants {#invariants}

These are meant to hold for every component and to be tested, not only
documented.

- **No secret in argv, environment or URL.** Secrets come from systemd
  credentials, a file descriptor or a no-echo prompt.
- **Nobody gets an account by signing in.** Accounts exist only through the
  admin API, against an allowlist.
- **Identities are never linked by e-mail.** An external identity is bound to an
  account by an explicit admin act.
- **Every parser of untrusted input is fuzzed**, with its corpus committed, and
  every input has a size, depth and time bound that a test proves is refused.
- **No component trusts another through a shared static token**, and there are
  no git hooks calling back into the server.
- **Performance is measured against git itself**, single core.

## What is here {#what-is-here}

- **[Design]({{< relref "design.md" >}})** — a summary of the design, with the milestones and what is built.
- **[Modules]({{< relref "modules.md" >}})** — every repository in go-vcs, with its source and its reference.
