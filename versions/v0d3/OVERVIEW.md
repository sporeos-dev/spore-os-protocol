<!--
Copyright 2026 Matt Harrison
SPDX-License-Identifier: Apache-2.0
-->

# Spore OS Protocol — Overview
**Version:** v0d3

---

## What it is

A local-first IPC protocol. Applications expose capabilities; Spore OS routes calls. Capabilities work like functions; apps like objects; a PC like a single unit.

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
- [Roles](ROLES.md) — raw, hint, and witness roles.
- [Security](SECURITY.md) — trust, risk, and permissions.
- [Reserved](RESERVED.md) — protocol-reserved names and tokens.
