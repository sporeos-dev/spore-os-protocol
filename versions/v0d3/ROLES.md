<!--
Copyright 2026 Matt Harrison
SPDX-License-Identifier: Apache-2.0
-->

# Spore OS Protocol — Roles

Nodes declare roles in their manifest.

## I/O

Live output and hub confirmations follow the initiating interface.
Neither replaces the [final response](REQUEST.md). Wire syntax is open.

## Raw

Address a Raw node by ID to send free-form text, not arguments or flags.
Handle placement is open.

**Example: Send literal text to a bridge (handle omitted)**

```text
dev.sporeos.foundry.git status
```

## Hint

A trailing `?` asks for help, not execution. The hub can answer from the
manifest and ask the owning node for more detail.

```text
capability?
```

## Test

End-to-end tests describe messages to send and replies or events to
expect. The manifest format is open.

## Health

The hub gathers pass/fail self-checks from participating nodes and
distinguishes failure from no reply. Probe syntax is open.

## Witness

Witnesses get fire-and-forget copies of all messages and events without
subscribing. They handle their own filtering and storage.

Copies start with `witness ` and include `spore.time=` and a
[witness flag](RESERVED.md#flags).

```text
witness Something happened
```

