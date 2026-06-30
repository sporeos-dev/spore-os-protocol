<!--
Copyright 2026 Matt Harrison
SPDX-License-Identifier: Apache-2.0
-->

# Spore OS Protocol — Overview
**Version:** v0d2  
**Full spec:** [SPEC.md](SPEC.md)

---

## What it is

A local IPC protocol over Unix domain sockets. Applications expose capabilities; a hub routes calls between them. Think of every app as a function you can call by name.

---

## Core pieces

**Hub** — a local daemon. Holds a manifest registry and routes messages. Knows about nodes even when they aren't running.

**Node** — any process (CLI, GUI, daemon, script). Its manifest is installed with the hub separately; when the process runs it connects and is then callable by other nodes.

**Manifest** — a YAML file that is the node's full contract. Declares its ID, its API (callable subjects), and any custom errors.

---

## Manifest shape

```yaml
id: com.example.clock
name: Clock
app: ./clock
schema: SPORE/v0d2
trust: standard
description: Provides time functions.
api:
  - name: get_time
    description: Returns the current time.
    usage:
      - clock.get_time
      - clock.get_time timezone=America/New_York
    inputs:
      - name: timezone
        type: string
        required: false
        description: IANA timezone. Defaults to UTC.
    outputs:
      - name: time
        type: string
        description: Current time in ISO 8601 format.
```

---

## Installing a node

Before a node can connect, its manifest must be registered with the hub:
```
SPORE.node.install path=/path/to/clock.manifest.spore.yaml ~h1
← ~h1:SPORE.node.install ok capture=SPORE.hub
```
The hub now knows the node exists and can route to its subjects — even before the node process is running.

---

## Connecting

When the node process starts, it opens the hub socket and sends its ID:
```
com.example.clock
```
Hub replies `OK`. Node is now live and callable.

Installed but not connected: the hub knows the node, but the process isn't running — calls to its subjects return `RouteNotFound`.  
Connected: the node is running and has completed the handshake.

---

## Message channels

There are three distinct channels on the protocol:

**Request / response** — the primary channel. A caller sends a subject with a handle; the hub routes it to the owning node; the node replies. Every call gets exactly one response.

**Broadcast** — a publish/subscribe channel. A node publishes to a declared topic; the hub fans out the message to all current subscribers. No handle, no response. Fire-and-forget.

**Witness** — a passive observation channel. The hub sends a copy of every message (calls, responses, hub events, and broadcast) to any node registered as a witness. Fire-and-forget — the hub never waits on witnesses and witnesses never reply.

These are independent. A node can be a caller, a receiver, a publisher, a subscriber, a witness, or any combination.

---

## Wire format

**Call:**
```
subject [key=value ...] ~handle
```

**Response:**
```
~handle:subject [fields] ok capture=node_id
```

**Error:**
```
~handle:subject error code=SomeCode what="description" node_error capture=node_id
```
Every error carries exactly one origin flag: `spore_error` (hub), `node_error` (receiver), `cast_error` (caller's fault), or `capture_error` (bad reply from responder).

Every call gets exactly one response. `ok`, `error`, `custom_error`, or `cancelled` — always exactly one.

**Publish (broadcast):**
```
publish subject [key=value ...]
```
No handle. The hub injects `cast=` (publisher ID) and fans out with `capture=` (subscriber ID) to each subscriber.

### Example exchange
```
clock.get_time timezone=UTC ~h1
← ~h1:clock.get_time time="2026-03-13T14:32:00Z" ok capture=com.example.clock
```

---

## Inline calls

The hub resolves inner calls before forwarding the outer one. Two forms:

**Key binding** — result of the inner call is bound to a named argument:
```
SPORE.node.install path=(dialog.file_picker)
```

**Spread** — all fields from the inner call's response are spread into the outer call:
```
filesystem.list (dialog.dir.open) recursive
```

If the inner call returns `cancelled` or `error`, the outer call is never forwarded — the result is returned to the original caller immediately.

---

## Witness

Declare `witness: true` in the manifest. The node then receives every message on the wire — prefixed with `witness ` so it can distinguish them from normal calls:
```
witness clock.get_time cast=com.example.shell ~h1 spore_incoming spore_time=1744732800000
witness ~h1:clock.get_time time="2026-03-13T14:32:00Z" ok capture=com.example.clock spore_outgoing spore_time=1744732800001
```
Used for logging, debugging, monitoring. The hub never blocks on witnesses.

---

## Broadcast

Declare topics in the manifest under `topics`. Publish to them at any time — only the declaring node may publish. Subscribers receive delivered messages via `SPORE.topic.subscribe`.

```
# Publisher sends:
publish clock.ticked time="2026-06-30T12:00:00Z"

# Hub delivers to each subscriber:
publish clock.ticked time="2026-06-30T12:00:00Z" cast=com.example.clock capture=com.example.monitor
```

Topic names by convention use past tense (e.g. `SPORE.node.spawned` vs `SPORE.node.spawn`). Nodes subscribe on connect, unsubscribe on disconnect. Unsubscribed nodes silently miss published messages — the hub does not buffer.

---

## Trust & Risk

Every node has a **trust level**; every API entry and topic has a **risk level**.

**Trust levels:** `system` · `developer` · `trusted` · `standard` · `untrusted`

**Risk levels:** `benign` · `standard` · `personal` · `secret` · `protected`

`benign` is open to all callers by default. `protected` is governed by a hardcoded hub whitelist and cannot be unlocked by policy. `personal` and `secret` require explicit user grants; the hub warns on first request. `developer` trust bypasses all policy — the hub warns when a `developer`-trust node connects.

Trust is declared in the manifest and may be overridden by the user at install time, except `system` (hardcoded whitelist only).

---

## Hub API (built-in subjects)

| Subject | What it does |
|---|---|
| `SPORE.node.list` | List all known nodes |
| `SPORE.node.install path=...` | Register a manifest |
| `SPORE.node.uninstall node=...` | Remove a node |
| `SPORE.topic.subscribe topic=...` | Subscribe to a topic |
| `SPORE.topic.unsubscribe topic=...` | Unsubscribe from a topic |

---

## Rules worth knowing

- `SPORE.*` is protocol-reserved — no third-party node may use that namespace.
- Manifests are read-only at runtime. Changes require a node restart.
- Every call must include a `~handle`. The hub rejects calls without one.
- All messages are newline-delimited plaintext. No binary framing.
- Only the node that declares a topic may publish to it. Others receive `RouteNotAllowed`.
- Broadcast subscriptions are not persistent — nodes must re-subscribe on each connect.
- `publish` messages carry no handle and receive no response.
