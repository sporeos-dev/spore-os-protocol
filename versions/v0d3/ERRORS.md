<!--
Copyright 2026 Matt Harrison
SPDX-License-Identifier: Apache-2.0
-->

# Spore OS Protocol — Errors

An error response uses `error`, `code=`, and `what=`. It can come from the hub
or a node and never includes `ok`.

```
~h1:time.get_time error code=Missing.Node what="no node connected for this subject" capture=SPORE.hub
~h2:time.get_time error code=Generic.Node what="internal clock failure" capture=com.example.clock
```

`code=` identifies the error and component as `Code.Module`, such as
`NotPermitted.Permission`. `what=` gives a human-readable description.

Only a topic's declaring node may publish to it; other publishers receive
`RouteNotAllowed`.

## Codes

| Code | Meaning |
|---|---|
| `Generic` | General error. |
| `ConnectionFailure` | Connection could not be established or was lost. |
| `HandshakeDenial` | Node handshake refused. |
| `InitializationFailure` | Component failed to initialize. |
| `Malformed` | Message or value could not be parsed. |
| `Missing` | Expected resource, argument, or route was not found. |
| `NotApplicable` | Operation does not apply. |
| `NotImplemented` | Subject is declared but not implemented. |
| `NotPermitted` | Caller is not permitted to perform the action. |
| `Timeout` | Operation did not complete in time. |
| `UserDenial` | User denied the operation. |
| `HandleInUse` | Handle reused while still in flight. |
| `ReservedLanguage` | Reserved keyword or namespace used where prohibited. |

## Modules

| Module | Component |
|---|---|
| `Bus` | Message routing bus. |
| `Hub` | Hub process. |
| `Hyphae` | User-space layer. |
| `Manifest` | Manifest parsing or validation. |
| `Message` | Wire message parsing. |
| `Node` | Node lifecycle or connection. |
| `Permission` | Permission management. |
| `Policy` | Policy enforcement. |
| `Registry` | Node registry. |
| `Spore` | Hub API handler. |

## Pipes

A stage error stops the pipe and is returned to the caller. A forwarded
field or flag that duplicates an explicit argument or flag also stops the
pipe before that stage executes. A stage timeout stops the pipe with
`Timeout.Pipe`; see [GRAMMAR.md](GRAMMAR.md#piping).

Error events are also sent to [witnesses](ROLES.md#witness).
