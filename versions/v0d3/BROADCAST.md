<!--
Copyright 2026 Matt Harrison
SPDX-License-Identifier: Apache-2.0
-->

# Spore OS Protocol — Broadcast

Only the topic owner publishes. Delivery to current subscribers is
fire-and-forget: no handle, reply, retry, or buffering. Broadcasts also
reach witnesses. Subscribers must resubscribe after reconnecting.

## Example: Announce a clock tick

```text
publish clock.ticked time="2026-03-13T14:32:00Z"
```

See [Grammar](GRAMMAR.md#broadcast).
