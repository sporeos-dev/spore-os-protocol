<!--
Copyright 2026 Matt Harrison
SPDX-License-Identifier: Apache-2.0
-->

# Spore OS Protocol — Roles

Nodes declare roles in their manifest.

## Raw

Not yet implemented; behavior is not yet specified.

## Hint

Not yet implemented; behavior is not yet specified.

## Witness

Witnesses receive copies of all messages and events, including broadcasts
without subscribing. Delivery is fire-and-forget; the hub never waits for
witnesses. Each witness handles its own filtering and storage.

Copies begin with `witness ` and include `spore_time=` and a
[witness flag](RESERVED.md#flags). Nodes may also send
`witness body="message"`; the hub adds `cast=`, `spore_time=`, and `spore_node`
and forwards it without a response.

The hub has no structured log; witness handshake failures go to stderr.
