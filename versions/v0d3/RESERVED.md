<!--
Copyright 2026 Matt Harrison
SPDX-License-Identifier: Apache-2.0
-->

# Spore OS Protocol — Reserved Names

These names and tokens belong to the protocol. Reserved arguments and flags
cannot be used as manifest inputs or outputs, except where noted.

## Arguments

| Name | Purpose and use |
|---|---|
| `capture` | Hub-injected response field identifying the node that handled the call. |
| `cast` | Hub-injected call field identifying the caller. Delegated calls also identify the original requester. |
| `code` | Error identifier in `Code.Module` format. |
| `spore.time` | Witness field: Unix time in milliseconds, present on every witness copy. |
| `timeout` | Reserved **only in pipe stages**. A caller sets seconds or `false` to override the stage timeout; the hub consumes it without forwarding it. |
| `what` | Human-readable error description; required with `error`. |

Hub-injected `cast` and `capture` are not written by nodes or callers.

## Flags

| Name | Purpose and use |
|---|---|
| `cancelled` | Additional flag on a successful response when the operation was cancelled. |
| `error` | Error response flag, written by the responding node or hub. Requires `code` and `what`. |
| `ok` | Success response flag, injected by the hub. |
| `spore.event` | Witness flag: a hub lifecycle or internal error event. |
| `spore.error` | Witness flag: a hub or runtime error event. |
| `spore.incoming` | Witness flag: a message received from a node before hub injection or routing. |
| `spore.node` | Witness flag: a node-emitted observability message. |
| `spore.outgoing` | Witness flag: a message delivered to a node after hub injection. |

Every response has exactly one of `ok` or `error`; a successful cancellation
carries `ok cancelled`. Witness fields
are hub-injected on witness copies only; exactly one of `spore.incoming`,
`spore.outgoing`, `spore.event`, `spore.error`, or `spore.node` is present
on each copy.

## Grammar

| Token | Purpose and use |
|---|---|
| `publish` | Line prefix used by publishers for broadcasts. The hub rejects subjects named `publish` or beginning with `publish `. |
| `witness` | Line prefix used for witness traffic. The hub rejects subjects named `witness` or beginning with `witness `. |
| `:` | Separates actor and origin in a delegated `cast`; prohibited in node IDs, subject names, and topic names. |
| `~` | Prefix for caller-supplied handles; `~handle:subject` binds a response to a call. |
| `\|` | Pipe separator, resolved by the hub. |

## Namespaces and prefixes

| Name | Purpose and use |
|---|---|
| `SPORE.*` | Hub and protocol-internal subjects and error codes. Third-party node IDs cannot begin with `SPORE.`; the hub rejects such handshakes unless the node is recognized as hub-internal. |
| `spore.` | Reserved argument-name prefix. No manifest input or output may begin with it. |

See [GRAMMAR.md](GRAMMAR.md) for message syntax and piping, and
[ERRORS.md](ERRORS.md) for errors.
