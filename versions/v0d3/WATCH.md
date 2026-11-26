<!--
Copyright 2026 Matt Harrison
SPDX-License-Identifier: Apache-2.0
-->

# Spore OS Protocol — Watch

A watch observes a caller's selection and sends events to that caller.
A quiet watch can stay active. Either the caller or the provider can end it.

In these sketches, `attach` gets an `ok` reply but keeps its handle alive.
`emit` delivers an event; `detach` asks to stop; `end` releases the handle
and may carry a final event. The syntax is provisional.

## Example: Watch a directory

```text
→ watch attach filesystem.changes path="/notes" ~h1
← ~h1:filesystem.changes ok capture=com.example.filesystem
← watch emit ~h1 path="/notes/today.md" change=modified
→ watch detach ~h1
← watch end ~h1 detached
```

## Example: Set a one-shot alarm

```text
→ watch attach clock.alarm at="08:00" ~h2
← ~h2:clock.alarm ok capture=com.example.clock
← watch end ~h2 fired
```
