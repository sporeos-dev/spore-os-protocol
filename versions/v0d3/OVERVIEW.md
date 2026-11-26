<!--
Copyright 2026 Matt Harrison
SPDX-License-Identifier: Apache-2.0
-->

# Spore OS Protocol — Overview
**Version:** v0d3

---

## What it is

A local-first IPC protocol. Applications expose capabilities; Spore OS routes calls. Capabilities work like functions; apps like objects; a PC like a single unit.

## APIs

- [Request](REQUEST.md) — call a capability and receive one final response.
- [Broadcast](BROADCAST.md) — publish a message to current topic subscribers.
- [Watch](WATCH.md) — open an ongoing subscription for selected events.
- [Guard](GUARD.md) — allow or deny delivery on a message route.
- [Mutate](MUTATE.md) — change a message before delivery.

Watch, Guard, and Mutate describe intended behavior; their wire syntax is
not yet specified.

## Roles

[Roles](ROLES.md) cover focused interactions: **I/O** routes live output and
hub confirmations to the initiating interface; **Raw** carries free-form
text; **Hint** provides capability help via `?`; **Test** describes expected
message exchanges; **Health** checks participating nodes; and **Witness**
observes protocol traffic.

---

## Wire format

```
clock.get_time timezone=UTC ~h1
← ~h1:clock.get_time time="2026-03-13T14:32:00Z" ok capture=com.example.clock
```

## References

- [Manifest](MANIFEST.md) — node contracts and capabilities.
- [Schema](spore.manifest.v0.schema.yaml) — manifest validation.
- [Handshake](HANDSHAKE.md) — connecting and spawning nodes.
- [Grammar](GRAMMAR.md) — calls, responses, broadcasts, and piping.
- [Errors](ERRORS.md) — error responses and codes.
- [Security](SECURITY.md) — trust, risk, and permissions.
- [Reserved](RESERVED.md) — protocol-reserved names and tokens.
