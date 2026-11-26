<!--
Copyright 2026 Matt Harrison
SPDX-License-Identifier: Apache-2.0
-->

# Spore OS Protocol — Request

A capability call gets one `ok` or `error` reply under the same handle,
even with no outputs. Live [I/O](ROLES.md#io) is separate.

## Example: Get the time

```text
clock.get_time timezone=UTC ~h1
~h1:clock.get_time time="2026-03-13T14:32:00Z" ok capture=com.example.clock
```

See [Grammar](GRAMMAR.md#request) and [Piping](GRAMMAR.md#piping).
