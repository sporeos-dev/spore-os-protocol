<!--
Copyright 2026 Matt Harrison
SPDX-License-Identifier: Apache-2.0
-->

# Spore OS Hub — Normative Contract
**Version:** v0d2  
**Protocol spec:** [SPEC.md](SPEC.md)  
**Date:** 2026-06-30  
**Status:** Development

## Overview

The normative `SPORE.*` subjects and topics a conforming hub must expose. The hub (`dev.sporeos.SPORE`, `system` trust) is itself a node — its full manifest is in [spore-os/spored/spored.manifest.spore.yaml](https://github.com/sporeos-dev/spore-os). Implementations may expose additional subjects beyond those listed here.

> **Known gaps:** The permission enforcement model and security semantics are implemented but not fully specified. See [OPEN.md](OPEN.md).

---

## Introspection

Five commands form the introspection API. Together they let clients, shells, and tools discover, inspect, complete, and guide Spore expressions without weakening the rule that an executable subject must exactly match a manifest API entry.

Each takes `body=` — the target or partial expression being inspected. `SPORE.complete` and `SPORE.hint` also accept `cursor=`, the caller's cursor offset into `body=`. A target need not be an executable subject; see [SPEC.md §5.2](SPEC.md).

| Command | Answers | Returns |
|---|---|---|
| `SPORE.help` | What is this, and how is it used? | `help=` |
| `SPORE.state` | What is true about it right now? | `state=` |
| `SPORE.list` | What are its immediate children? | `children=` |
| `SPORE.complete` | What could this partial text validly become? | `completions=` |
| `SPORE.hint` | What is relevant to what I'm typing? | any of the above |

**`SPORE.help`** returns static information about a target — what it is, what it means, how it is used, and what inputs and outputs are expected. Help is manifest-derived and does not depend on current runtime state.

```
SPORE.help body=clock.get_time ~h1
← ~h1:SPORE.help help={...} ok capture=SPORE.hub

SPORE.help body=com.example.clock ~h2
← ~h2:SPORE.help help={...} ok capture=SPORE.hub

SPORE.help body=RouteNotFound ~h3
← ~h3:SPORE.help help={...} ok capture=SPORE.hub
```

Called with no `body=`, `SPORE.help` describes the hub itself.

**`SPORE.state`** returns runtime information about a target — whether a node is installed or connected, whether a topic has subscribers, whether a permission is granted, what hub-level condition currently applies. State describes what is true now, not how to use the thing.

```
SPORE.state body=com.example.clock ~h1
← ~h1:SPORE.state state={...} ok capture=SPORE.hub

SPORE.state body=SPORE.node.spawned ~h2
← ~h2:SPORE.state state={...} ok capture=SPORE.hub
```

**`SPORE.list`** returns the immediate children of a target in the introspection hierarchy. With no `body=`, the target is the introspection root.

```
SPORE.list ~h1
← ~h1:SPORE.list children=["SPORE", "com.example.clock", "com.example.filesystem"] ok capture=SPORE.hub

SPORE.list body=SPORE.node ~h2
← ~h2:SPORE.list children=["SPORE.node.install", "SPORE.node.spawn", "SPORE.node.kill"] ok capture=SPORE.hub

SPORE.list body=com.example.clock ~h3
← ~h3:SPORE.list children=["clock.get_time", "clock.set_timer"] ok capture=SPORE.hub
```

Listing is hierarchical: the root lists the hub and every installed node, a node lists its commands, a prefix lists the names below it, and a topic area lists its topics. `SPORE.list` subsumes the per-category list commands of earlier drafts — any result those produced is reachable by walking the hierarchy.

**`SPORE.complete`** returns possible textual completions for an incomplete Spore expression. It is intended for shells, command runners, and editors. Completion is about syntax and continuation — what this text could validly become — which makes it distinct from `SPORE.list`, which is about what belongs to a target.

```
SPORE.complete body="clock.get_ti" cursor=12 ~h1
← ~h1:SPORE.complete completions=["clock.get_time"] ok capture=SPORE.hub

SPORE.complete body="clock.get_time timez" cursor=20 ~h2
← ~h2:SPORE.complete completions=["timezone="] ok capture=SPORE.hub
```

**`SPORE.hint`** returns contextual guidance for an in-progress or complete expression. A hint response may carry any combination of `help=`, `state=`, `children=`, and `completions=`, along with examples, warnings, validation information, or a suggested default action.

```
SPORE.hint body="clock.set_timer " cursor=16 ~h1
← ~h1:SPORE.hint help={...} completions=["duration=", "label="] ok capture=SPORE.hub
```

`SPORE.hint` is the broadest interactive affordance, for workflows where the user is typing, exploring, or unsure what comes next. It does not replace the precise commands — a client that knows exactly what it needs should call `SPORE.help`, `SPORE.state`, `SPORE.list`, or `SPORE.complete` directly. Hint responses are contextual, so a client must not assume any particular section is present.

### Interactive Sugar

Shells and command runners may layer sugar over these commands: a bare non-executable prefix such as `SPORE.node` treated as `SPORE.list body=SPORE.node`, a tab keypress treated as `SPORE.complete` or `SPORE.hint` with the current input and cursor, a trailing `help`, `state`, or `list` treated as the corresponding command against the preceding target.

This is client behaviour. It does not change the wire grammar and does not make prefixes executable. See [SPEC.md §5.2](SPEC.md).

---

## Node Management

This group covers the full node lifecycle: registering manifests, starting and stopping processes, and inspecting what the hub knows.

**Installing and uninstalling** registers or removes a node's manifest from the hub's registry. A node must be installed before it can be called or spawned — the hub can route to an installed node's subjects even when it isn't running.

```
SPORE.node.install path=/nodes/clock/clock.manifest.spore.yaml ~h1
← ~h1:SPORE.node.install ok capture=SPORE.hub

# Pipeline — open a file picker, then install the selected manifest:
dialog.file_picker ~h2 | SPORE.node.install
← ~h2:SPORE.node.install ok capture=SPORE.hub

SPORE.node.uninstall node=com.example.clock ~h3
← ~h3:SPORE.node.uninstall ok capture=SPORE.hub
```

`SPORE.node.uninstall` accepts `node=` (reverse-domain ID).

**Spawning and killing** starts or stops a node process. The hub uses the `app` field from the installed manifest to launch the node. Spawned nodes are always headless.

```
SPORE.node.spawn node=com.example.clock ~h1
← ~h1:SPORE.node.spawn ok capture=SPORE.hub

SPORE.node.kill node=com.example.clock ~h2
← ~h2:SPORE.node.kill ok capture=SPORE.hub
```

**Inspecting nodes** — node management exposes no list or describe subjects of its own. Use `SPORE.list` to enumerate installed nodes and their commands, `SPORE.help` for a node's manifest details, and `SPORE.state` for whether it is installed or connected.

---

## Topics

Topics are the broadcast channel. This group handles subscribing to topics and inspecting what's available.

**Subscribing** — a node subscribes to a topic on connect and will receive published messages while connected. Subscriptions do not persist across disconnect; nodes must re-subscribe each time they connect.

```
SPORE.topic.subscribe topic=SPORE.node.spawned ~h1
← ~h1:SPORE.topic.subscribe ok capture=SPORE.hub

SPORE.topic.unsubscribe topic=SPORE.node.spawned ~h2
← ~h2:SPORE.topic.unsubscribe ok capture=SPORE.hub
```

**Discovery** — topics have no list or describe subjects of their own. Use `SPORE.list` to enumerate them, `SPORE.help` to describe one, and `SPORE.state` to see whether it currently has subscribers.

**Hub lifecycle events** — the hub publishes no lifecycle topics of its own. Inspect node state directly via `SPORE.state`.

---

## Permissions

> **Open issue:** The enforcement model — the policy matrix, how requests are surfaced to users, and how grants are stored — is not yet fully specified. See [OPEN.md](OPEN.md).

Permissions govern which nodes may call which capabilities (subjects or topics). The core flow is: a node requests access, the hub presents it to the user, and the result is granted or denied.

```
# Node requests access at runtime:
SPORE.permission.request node=com.example.agent capability=filesystem.read ~h1
← ~h1:SPORE.permission.request granted ok capture=SPORE.hub

# Granting and revoking can also be done directly:
SPORE.permission.grant node=com.example.agent capability=filesystem.read ~h2
← ~h2:SPORE.permission.grant ok capture=SPORE.hub

SPORE.permission.revoke node=com.example.agent capability=filesystem.read ~h3
← ~h3:SPORE.permission.revoke ok capture=SPORE.hub
```

Both API subjects and topics are considered capabilities. Current grants are inspected with `SPORE.list body=SPORE.permission` and `SPORE.state`.

---

## Security

The security group manages the hub's keyring of trusted public keys and handles signing and verifying files (manifests, binaries). All subjects in this group are `protected` — accessible only to hub-whitelisted nodes.

**Keyring** — the keyring is the hub's set of trusted public keys used to verify node manifests and binaries at install time. `SPORE.security.keyring.grant` and `SPORE.security.keyring.revoke` add and remove keys; the keyring is inspected through `SPORE.list` and `SPORE.state`.

```
SPORE.security.keyring.grant path=/keys/trusted.gpg ~h1
← ~h1:SPORE.security.keyring.grant ok capture=SPORE.hub

SPORE.security.keyring.revoke key=<public-key> ~h2
← ~h2:SPORE.security.keyring.revoke ok capture=SPORE.hub
```

**Signing and verification** — a file is signed with a private key passphrase, producing a `.sig` companion file. Verification checks the file against the signature and a trusted key.

```
SPORE.security.signature.sign path=/nodes/clock/clock passphrase=... ~h1
← ~h1:SPORE.security.signature.sign signature=/nodes/clock/clock.sig ok capture=SPORE.hub

SPORE.security.signature.verify path=/nodes/clock/clock signature=/nodes/clock/clock.sig key=<public-key> ~h2
← ~h2:SPORE.security.signature.verify verified ok capture=SPORE.hub
```

---

*Spore OS Hub Contract v0d2 — June 2026*

