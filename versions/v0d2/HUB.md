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

## Meta

```
SPORE.help ~h1
← ~h1:SPORE.help nodeinfo={...} ok capture=SPORE.hub
```

`SPORE.help` returns the hub's own node info — equivalent to calling `SPORE.node.help` on itself.

---

## Node Management

This group covers the full node lifecycle: registering manifests, starting and stopping processes, and inspecting what the hub knows.

**Installing and uninstalling** registers or removes a node's manifest from the hub's registry. A node must be installed before it can be called or spawned — the hub can route to an installed node's subjects even when it isn't running.

```
SPORE.node.install path=/nodes/clock/clock.manifest.spore.yaml ~h1
← ~h1:SPORE.node.install ok capture=SPORE.hub

# Inline substitution — open a file picker to choose the manifest:
SPORE.node.install path=(dialog.file_picker) ~h2
← ~h2:SPORE.node.install ok capture=SPORE.hub

SPORE.node.uninstall node=com.example.clock ~h3
← ~h3:SPORE.node.uninstall ok capture=SPORE.hub
```

`SPORE.node.uninstall` accepts either `node=` (reverse-domain ID) or `path=` (manifest path).

**Spawning and killing** starts or stops a node process. The hub uses the `app` field from the installed manifest to launch the node. Spawned nodes are always headless.

```
SPORE.node.spawn node=com.example.clock ~h1
← ~h1:SPORE.node.spawn ok capture=SPORE.hub

SPORE.node.kill node=com.example.clock ~h2
← ~h2:SPORE.node.kill ok capture=SPORE.hub
```

**Listing and describing** — `SPORE.node.list` returns all installed node IDs; pass the `connected` flag to filter to currently running nodes only. `SPORE.node.help` returns the manifest details for a specific node.

```
SPORE.node.list ~h1
← ~h1:SPORE.node.list nodes=["com.example.clock", "com.example.filesystem"] ok capture=SPORE.hub

SPORE.node.list connected ~h2
← ~h2:SPORE.node.list nodes=["com.example.clock"] ok capture=SPORE.hub

SPORE.node.help node=com.example.clock ~h3
← ~h3:SPORE.node.help nodeinfo={...} ok capture=SPORE.hub
```

---

## Introspection

These subjects expose the hub's knowledge of what subjects and error codes exist across all installed nodes. Useful for discovery, tooling, and building dynamic interfaces.

**Subjects** — `SPORE.command.list` lists all known API subjects across installed nodes; scope to one node with `node=`. `SPORE.command.help` describes a specific subject.

```
SPORE.command.list ~h1
← ~h1:SPORE.command.list commands=["clock.get_time", "filesystem.read", ...] ok capture=SPORE.hub

SPORE.command.list node=com.example.clock ~h2
← ~h2:SPORE.command.list commands=["clock.get_time", "clock.set_timer"] ok capture=SPORE.hub

SPORE.command.help command=clock.get_time ~h3
← ~h3:SPORE.command.help commandinfo={...} ok capture=SPORE.hub
```

**Errors** — `SPORE.error.list` lists all standard protocol error codes, or a specific node's custom errors when `node=` is passed. `SPORE.error.help` describes any error code — standard or custom.

```
SPORE.error.list ~h1
← ~h1:SPORE.error.list errors=["RouteNotFound", "RouteNotConnected", ...] ok capture=SPORE.hub

SPORE.error.list node=com.example.clock ~h2
← ~h2:SPORE.error.list errors=["clock.err.invalid_timezone", ...] ok capture=SPORE.hub

SPORE.error.help errorcode=RouteNotFound ~h3
← ~h3:SPORE.error.help errorinfo={...} ok capture=SPORE.hub
```

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

**Discovery** — `SPORE.topic.list` and `SPORE.topic.help` work the same way as their command equivalents: list all topics (or scope to a node), and describe a specific topic.

**Hub lifecycle events** — the hub publishes these topics itself. Nodes interested in the node lifecycle subscribe to them.

| Topic | Published when |
|---|---|
| `SPORE.node.installed` | A manifest is registered with the hub |
| `SPORE.node.uninstalled` | A node is unregistered |
| `SPORE.node.spawned` | The hub starts a node process |
| `SPORE.node.killed` | The hub stops a node process |

Each carries a single `node` field with the affected node ID:

```
publish SPORE.node.spawned node=com.example.clock cast=dev.sporeos.SPORE capture=com.example.monitor
```

---

## Permissions

> **Open issue:** The enforcement model — the policy matrix, how requests are surfaced to users, and how grants are stored — is not yet fully specified. See [OPEN.md](OPEN.md).

Permissions govern which nodes may call which subjects or subscribe to which topics. The core flow is: a node requests access, the hub presents it to the user, and the result is granted or denied.

```
# Node requests access at runtime:
SPORE.permission.request command=filesystem.read ~h1
← ~h1:SPORE.permission.request granted ok capture=SPORE.hub

# Granting and revoking can also be done directly:
SPORE.permission.grant node=com.example.agent command=filesystem.read ~h2
← ~h2:SPORE.permission.grant ok capture=SPORE.hub

SPORE.permission.revoke node=com.example.agent command=filesystem.read ~h3
← ~h3:SPORE.permission.revoke ok capture=SPORE.hub
```

Both `command=` and `topic=` are accepted by grant, revoke, and request. `SPORE.permission.list` returns current grants, optionally filtered by node, command, or topic.

---

## Security

The security group manages the hub's keyring of trusted public keys and handles signing and verifying files (manifests, binaries). All subjects in this group are `protected` — accessible only to hub-whitelisted nodes.

**Keyring** — the keyring is the hub's set of trusted public keys used to verify node manifests and binaries at install time. `SPORE.security.keyring.grant` and `SPORE.security.keyring.revoke` add and remove keys; `SPORE.security.keyring.list` and `SPORE.security.keyring.info` inspect them.

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

