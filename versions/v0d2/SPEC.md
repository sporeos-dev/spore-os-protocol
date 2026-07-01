<!--
Copyright 2026 Matt Harrison
SPDX-License-Identifier: Apache-2.0
-->

# Spore OS Protocol Specification
**Version:** v0d2  
**Schema:** `SPORE/v0d2`  
**Date:** 2026-06-30  
**Status:** Development

## 1. Overview

Spore OS is a **local IPC protocol**. It provides a socket interface that lets applications discover each other and exchange messages — without knowing anything about each other's implementation.

The core idea is **treating applications as functions**: a node exposes capabilities, and the hub routes calls between them. A node can be a CLI tool, a background daemon, a GUI application, or a shell script.

Spore OS is not a framework, not a service mesh, and is explicitly not performance-sensitive. It is a simple, human-readable message bus for local processes.

---

## 2. Core Concepts

### 2.1 Hub

The hub is the local daemon. It:

- Routes messages between nodes based on **manifest knowledge**, not message syntax
- Maintains a **manifest registry** — nodes are known to the hub when installed, whether or not they are currently running
- Fans out **broadcast messages** to subscribed nodes on declared topics

The hub has its own manifest (`dev.sporeos.SPORE`) which defines its API. The hub manifest is part of this specification.

### 2.2 Nodes

A node is any process that connects to the hub via a Unix domain socket and presents a manifest. Nodes are:

- **Implementation-agnostic** — a node can be anything: a script, a CLI, a GUI app, a daemon
- **Self-describing** — every node has a manifest that is its full contract

Nodes exist in one of two states:

| State | Meaning |
|---|---|
| **Installed** | The hub has the manifest; the node is not running |
| **Connected** | The node is running and has completed handshake |

### 2.3 Manifest

The manifest is the **contract and API** for a node. It is the authoritative description of everything a node can do. The hub routes every message by resolving it against the manifest registry.

The manifest is **read-only at runtime**. Changes require a node restart.

### 2.4 The `SPORE.` Namespace

`SPORE.` is protocol-reserved. No third-party node may register an `id` beginning with `SPORE.`. The hub rejects any handshake presenting a `SPORE.` ID unless it is a recognised hub-internal node.

### 2.5 Trust and Risk

Every node has a **trust level** declared in its manifest. Every API entry and topic has a **risk level**. Together, these form the access control model for capability access.

- **Trust** is assigned to a node at install time. It describes how much authority that node is granted.
- **Risk** is declared by the node author on each capability. It describes how sensitive the capability is.

The hub enforces the relationship between them according to system policy and user preferences. See Section 11 for the full model.

---

## 3. Transport

### 3.1 Socket

Spore OS uses **Unix domain sockets** (UDS) for local communication.

- **macOS / Linux:** Unix domain socket at a conventional path
- **Windows:** Unix domain socket via the Windows UDS implementation

The canonical socket path for each platform is defined by the platform installer. All implementations (hub, CLI, nodes) on a given host must use the same path. No implementation should hardcode a socket path independently of the installer's definition.

### 3.2 Encoding

All messages are **newline-delimited plaintext strings**. No binary framing. Type information declared in the manifest is advisory — it helps implementors know what to expect, but the hub does not validate or coerce types.

### 3.3 Manifest Format

Manifests are authored in **YAML**. The hub reads the YAML file directly.

---

## 4. Handshake

**Installation** and **connection** are separate concerns.

**Installation** registers a node's manifest with the hub out-of-band — typically via a CLI command that reads the node's YAML manifest file. Installation persists across hub restarts. The hub can route to an installed node's capabilities even when the node is not running.

**Connection** is the runtime identification of a running node. When a node starts, it opens a connection to the hub socket and sends:

```
com.example.clock
```

The hub responds:

```
OK
```

The hub looks up the installed manifest for `com.example.clock` and marks the node as connected. If the node ID is not known, the hub responds with an error.

If a node disconnects after handshake, it reverts to installed-but-not-connected in the registry.

---

## 5. Addressing

All messages use dot-notation subjects. A subject is a single string token — the dots are a naming convention, not syntax the hub parses structurally.

```
subject [key=value ...] [~handle]
```

The subject on the wire must match the `name` field of an API entry in a node's manifest exactly as declared. The hub routes by exact match against registered command names.

### 5.1 Routing

The hub resolves the subject against its manifest registry and routes to the connected node that exposes it.

```
clock.get_time              # matches manifest api.name exactly
SPORE.node.list             # hub internal
```

---

## 6. Wire Grammar

All messages on the Spore OS wire share a single unified grammar — calls, responses, errors, and broadcast. The grammar covers argument syntax (key-value pairs, flags, booleans, arrays, objects, and inline call substitution), the `~handle` correlation mechanism, standard error codes, and the full set of reserved protocol keywords and prefixes.

See [GRAMMAR.md](GRAMMAR.md) for the full wire grammar reference.

---

## 7. Call / Response

A node calls another node's API subject. Each API entry is a `query` (returns data) or a `command` (triggers behaviour).

### 7.1 Query

Queries return data. The response arrives as `~handle:subject` with result fields and a hub-injected `capture=`.

```
clock.get_time ~h1
← ~h1:clock.get_time time="2026-03-13T14:32:00Z" ok capture=com.example.clock

filesystem.read path=/documents/notes.txt ~r1
← ~r1:filesystem.read content="Q1 planning notes..." ok capture=com.example.filesystem

SPORE.node.list ~meta
← ~meta:SPORE.node.list nodes=["com.example.clock", "com.example.filesystem"] ok capture=SPORE.hub
```

### 7.2 Command

Commands trigger behaviour. Unlike queries, a command may or may not return data — this is declared in the manifest's `outputs` field.

A command with no declared `outputs` returns an `ok` response — a response with no data fields, only the subject echo, `ok`, and `capture=`. The `ok` flag confirms the command completed without error.

```
# Command with no outputs (void return):
logger.append entry="started" ~l1
← ~l1:logger.append ok capture=com.example.logger

# Command that returns data:
filesystem.write path=/tmp/out.txt content="hello" ~w1
← ~w1:filesystem.write ok capture=com.example.filesystem

# Error on a command:
logger.append entry="started" ~l2
← ~l2:logger.append error code=logger.err.disk_full what="No space remaining" capture=com.example.logger
```

### 7.3 Composition

The output of one call feeds the next. The orchestrating node sequences calls itself; individual nodes are unaware of each other. The hub-level mechanism for this is inline call substitution (see [GRAMMAR.md §4](GRAMMAR.md)); beyond that, composition is a node implementation concern.

---

## 8. Manifest Specification

The manifest is the full YAML contract for a node — its identity, API (callable subjects with inputs and outputs), topics for broadcast, and custom error declarations. Manifests are registered with the hub at install time and are read-only at runtime.

See [MANIFESTS.md](MANIFESTS.md) for the full manifest schema reference.

---

## 9. Witness

### 9.1 Architecture

The hub dispatches a copy of every message — incoming and outgoing — to all registered witness nodes as a fire-and-forget side channel. The hub does not wait for witnesses and does not care if dispatch fails.

- Zero witnesses is a valid operational state.
- Multiple witnesses receive identical raw copies independently.
- Witnesses are responsible for all filtering, formatting, prettification, and storage. The hub makes no decisions about these.

A node registers as a witness by declaring `witness: true` in its own manifest. The hub sends it all witness traffic — there is no per-node scoping.

### 9.2 What Witnesses Receive

Every witness copy is prefixed with the reserved token `witness` followed by a space, then the wire-format message body. This prefix allows a witness node's client library to distinguish witness copies from normal calls and responses on the same socket connection — without inspecting content.

Examples:

```
witness clock.get_time cast=com.example.shell ~h1 spore_incoming spore_time=1744732800000
witness ~h1:clock.get_time time="2026-03-13T14:32:00Z" ok capture=com.example.clock spore_outgoing spore_time=1744732800001
```

The body after `witness ` carries one of four flags indicating what the copy represents:

| Flag | Meaning |
|---|---|
| `spore_incoming` | Raw message as received from a node, before any hub injection or routing |
| `spore_outgoing` | Message as delivered to a node, after hub injection (`cast=`, `capture=`, `ok`, etc.) |
| `spore_event` | A hub lifecycle or internal error event (not a routed message) |
| `spore_node` | A node-emitted observability message, not observed traffic (see Section 9.4) |

Exactly one of these four flags is present on every witness copy. All four types also carry `spore_time=` (Unix milliseconds since epoch).

> **Implementation note:** Implementations may include additional non-normative metadata fields on witness copies beyond those listed above (for example, a per-connection sequential message index). These are informational extensions and are not required by the protocol.

### 9.3 Hub Events as Messages

Hub lifecycle and error events (autostart failures, handshake failures, routing errors) are emitted as `spore_event` witness messages:

```
witness SPORE.hub.event level=warn what="Unable to complete handshake" error="..." spore_event spore_time=1744732800000
```

This allows a witness to correlate hub events with message traffic by timestamp.

### 9.4 Node-Emitted Witness Messages

A node may emit observability messages directly to the witness stream without those messages being part of any call or response. This is distinct from the hub observing traffic — the node is proactively publishing a diagnostic, audit, or lifecycle event of its own.

A node emits a node witness message by sending a line prefixed with the reserved `witness` token directly on its socket connection:

```
witness message="loaded config"
```

The hub intercepts this line before routing (it is never treated as a call or subject). It appends `cast=` identifying the emitting node and `spore_time=`, tags it `spore_node`, and dispatches to all registered witnesses. No response is sent back to the node — this is fire-and-forget.

```
witness message="loaded config" cast=com.example.mynode spore_node spore_time=1744732800000
```

The body after the `witness ` prefix is free-form. Nodes are responsible for choosing a format useful to their witnesses.

### 9.5 Hub Output

The hub emits no structured log of its own during normal operation. The sole exception is stderr for witness handshake failures — one line per failed witness, surfacing the error code and reason. All other observability flows through the witness stream.

### 9.6 Core Witness Nodes

The hub ships two built-in witness nodes. See [HUB.md](HUB.md) for details.

### 9.7 Failure Mode

Always passthrough — the hub never blocks a call due to a witness being absent or unreachable.

---

## 10. Broadcast

### 10.1 Architecture

Broadcast is a **publish/subscribe channel**. A node that declares topics in its manifest may publish to those topics at any time. The hub fans out each published message to all currently subscribed nodes.

- Only the node that declared a topic in its manifest may publish to it. Attempts by any other node to publish result in `RouteNotAllowed`.
- Subscriptions are managed at runtime via `SPORE.topic.subscribe` and `SPORE.topic.unsubscribe`.
- Nodes are responsible for subscribing on connect and unsubscribing on disconnect.
- If no subscribers are connected when a message is published, the message is silently dropped.
- Delivery is **fire-and-forget** — the hub does not wait for subscribers and does not retry.
- The hub does not buffer undelivered messages.

### 10.2 Wire Format

A broadcast message uses the reserved `publish` line prefix. No `~handle` is included — broadcast is one-way with no response:

```
publish subject [key=value ...] [flags]
```

**Example (publisher sends):**
```
publish SPORE.node.spawned node=com.example.clock
```

The hub injects `cast=` (publisher ID) before fanning out. Each subscriber receives a delivery with their own `capture=`:

```
publish SPORE.node.spawned node=com.example.clock cast=dev.sporeos.SPORE capture=com.example.monitor
```

`cast=` identifies the publishing node. `capture=` identifies this specific subscriber — each delivery carries the individual subscriber's ID.

### 10.3 Topic Ownership

A topic is declared in the publishing node's manifest under the `topics` field. Only that node may publish to it. The hub resolves topic ownership against the manifest registry, exactly as it resolves API subject ownership.

### 10.4 Subscriptions

Subscriptions are managed via hub API calls and are **not persistent** across disconnect/reconnect. A node is responsible for re-subscribing each time it connects:

```
SPORE.topic.subscribe topic=SPORE.node.spawned ~h1
← ~h1:SPORE.topic.subscribe ok capture=SPORE.hub

SPORE.topic.unsubscribe topic=SPORE.node.spawned ~h2
← ~h2:SPORE.topic.unsubscribe ok capture=SPORE.hub
```

### 10.5 Witness Integration

All broadcast traffic flows through the witness stream. Published messages appear as `spore_incoming` (the `publish` line from the publisher) and `spore_outgoing` (each fan-out delivery to a subscriber). Witness nodes see all published messages regardless of whether they have subscribed to a topic.

### 10.6 Trust and Risk

Topic subscriptions are subject to the same trust and risk policy as request/response calls. The hub enforces this at subscription time — `SPORE.topic.subscribe` returns `RouteNotAllowed` if the subscribing node's trust level does not satisfy the topic's risk policy.

---

## 11. Trust & Risk

### 11.1 Overview

Every node has a **trust level** declared in its manifest. Every API entry and topic has a **risk level**. Together, these define the security boundary for capability access.

Trust is assigned at install time and reflects how much authority the node is granted. Risk is declared by the node author and reflects how sensitive the capability is. The hub enforces the relationship between them according to system policy and user preferences.

### 11.2 Trust Levels

| Level | Meaning |
|---|---|
| `system` | Hub-internal nodes only. Governed by a hardcoded hub whitelist — cannot be assigned to third-party nodes at install. |
| `developer` | Full trust — bypasses all user preferences and policy. For local development only. The hub emits a warning whenever a `developer`-trust node connects. Do not use in production. |
| `trusted` | High trust. Access governed by policy; typically broader than `standard`. |
| `standard` | Regular trust. Expected for well-known, vetted nodes. |
| `untrusted` | All permissions must be explicitly granted. No implicit access beyond `benign` APIs. Default for nodes without an explicit trust assignment. |

Trust is set in the manifest by the node author. A user may override trust at install time — with the following constraints:

- **`developer`** — the hub warns that this level bypasses all policy and should not be used for production nodes.
- **`system`** — cannot be assigned via install; reserved for the hardcoded hub whitelist.

### 11.3 Risk Levels

| Level | Meaning |
|---|---|
| `benign` | No sensitive access. Open to all callers regardless of trust level. |
| `standard` | Minor capability. Access governed by policy. |
| `personal` | Access to personal data. Requires explicit user policy grant. Hub warns on first access request. |
| `secret` | Access to credentials, keys, or other sensitive material. Requires explicit user policy grant. Hub warns on first access request. |
| `protected` | System-critical capabilities. Access governed by a hardcoded hub whitelist. Cannot be unlocked by user preferences or policy. |

### 11.4 Default Policy

| Risk | Default |
|---|---|
| `benign` | **Open** — all callers may access, regardless of trust level |
| `standard` | Policy-defined — default behaviour is determined by the caller's trust level |
| `personal` | **Deny** by default. Requires explicit user grant. Hub warns on first grant request. |
| `secret` | **Deny** by default. Requires explicit user grant. Hub warns on first grant request. |
| `protected` | **Deny** to all except hub-whitelisted nodes. Cannot be overridden by user policy. |

> **Known gap:** Runtime enforcement of the middle tiers (`standard`, `personal`, `secret`) and the user preference mechanism are not yet fully defined. The model and manifest fields are established in v0d2; the enforcement matrix and permission API are open issues. See [OPEN.md](OPEN.md).

### 11.5 Developer Trust Warning

When a node with `developer` trust connects, the hub emits a warning on the witness stream:

```
witness SPORE.hub.event level=warn what="Node com.example.mynode has developer trust. All policy is bypassed. Do not use in production." spore_event spore_time=...
```

### 11.6 Protected Resources

`protected` risk capabilities and `system` trust assignments are governed by a whitelist hardcoded in the hub binary. This cannot be modified by config file or runtime call — the hub binary is the trust anchor for protected resources. Nodes not on the whitelist receive `RouteNotAllowed` regardless of their stated trust level.

### 11.7 Inline Call Trust Preservation

When the hub expands an inline call (see [GRAMMAR.md §4](GRAMMAR.md)), the inner call is forwarded with `cast=<original_caller>`. The hub never substitutes its own identity (`cast=SPORE.hub`) for hub-generated inner calls. This ensures that inline call substitution cannot be used to escalate privileges to a higher trust level.

---

## 12. Hub Contract

The normative hub contract — the `SPORE.*` subjects and topics a conforming hub must expose — is defined in [HUB.md](HUB.md).

---

*Spore OS Protocol v0d2 — June 2026*
