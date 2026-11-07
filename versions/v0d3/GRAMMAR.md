<!--
Copyright 2026 Matt Harrison
SPDX-License-Identifier: Apache-2.0
-->

# Spore OS Protocol — Wire Grammar
**Version:** v0d3

Messages are newline-delimited plaintext. See [ERRORS.md](ERRORS.md) for errors.

## Request

```text
capability [key=value ...] [flag ...] ~handle
```

The hub adds `cast=` and routes the request to the node exposing the capability.

## Response

```text
~handle:capability [key=value ...] ok capture=node_id
```

Every request gets one response. On success, the hub adds `ok` and `capture=`.
Commands with no outputs still receive `ok`.
A successful cancellation has both `ok` and `cancelled`.

## Broadcast

```text
publish topic [key=value ...] [flag ...]
```

Broadcasts have no handle or response. The hub adds `cast=` and sends the
message to current subscribers. Only the node declaring a topic may publish to
it. Delivery is fire-and-forget: there is no waiting, retrying, or buffering.
Subscriptions do not persist across disconnects; nodes must subscribe again.
Broadcasts also appear in the [witness stream](ROLES.md#witness).

## Parts

### Capability

A capability is a single token; dots are a naming convention. Requests must
match a manifest API name exactly. Introspection can describe partial names or
prefixes, but they are not executable capabilities.

### Prefixes

`publish` starts a broadcast; `witness` starts a witness message. See
[ROLES.md](ROLES.md#witness) and [RESERVED.md](RESERVED.md#grammar).

### Handle

The caller supplies `~handle` to match a request with its response. Handles are
global: reusing one while a request is in flight returns `HandleInUse`. The
response echoes it as `~handle:capability`. The token after `~` is arbitrary;
it has no built-in meaning.

### Arguments and Flags

Arguments use `key=value`; flags are bare tokens. A flag is true when present
and false when absent. Booleans instead use an explicit `true` or `false`.

| Value | Example |
|---|---|
| String | `name="hello world"` |
| Boolean | `enabled=true` |
| Array | `items=[a, b, c]` |
| Object | `point={x: 1, y: 2}` |
| Block | `content=<<` |

Single-quoted strings are literal. Double-quoted strings recognize `\\`, `\"`,
`\n`, `\r`, and `\t`; other backslash sequences remain unchanged.

A block starts with `key=<<` and ends at a line beginning with `>>`. Its content
is raw. Tokens after `>>` are parsed as arguments and may open another block;
the message completes when no blocks remain open. `<<` must be a named value,
and `>>` must be the first token on its line. The handle can appear on the
opening line or a closing line.

The hub injects `cast=` on forwarded requests and broadcasts. For direct calls
it identifies the caller; for delegated calls it is `actor:origin`. The actor
receives the response, while ordinary authorization uses the origin. Nodes
cannot supply `cast=` themselves.

## Piping

A pipe sequences calls from left to right:

```text
dialog.file_picker ~h1 | SPORE.node.install
```

Only the first stage has the caller's handle. The hub uses internal handles for
intermediate stages and returns the final response under the original handle.
Each node receives a normal call with resolved arguments.

The hub forwards response fields and non-protocol flags that match the next
stage's inputs; unmatched fields and protocol fields are not forwarded. Stages
can also have explicit arguments and a `timeout=` override.
If a stage returns `ok cancelled`, the hub stops the pipe and returns that
response under the original caller's handle. Later stages are not executed.

Intermediate stages use `cast=SPORE.pipe:<original_caller>`; the final stage
uses `cast=<original_caller>`. Ordinary authorization uses the original caller;
protected operations check the actor. See [ERRORS.md](ERRORS.md#pipes) for
failures.
