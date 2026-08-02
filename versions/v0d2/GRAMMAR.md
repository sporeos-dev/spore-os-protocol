<!--
Copyright 2026 Matt Harrison
SPDX-License-Identifier: Apache-2.0
-->

# Spore OS Protocol — Wire Grammar
**Version:** v0d2  
**Schema:** `SPORE/v0d2`  
**Spec:** [SPEC.md](SPEC.md)

This document defines the exact syntax of all messages on the Spore OS wire — message shapes, handles, argument syntax, inline call substitution, error grammar, and reserved keywords.

---

## 1. Message Shapes

All messages — calls and responses — share a single unified grammar. Direction and role are determined by structural position, not different syntax.

**Call (node → hub → receiver):**
```
subject [key=value ...] [flag ...] ~handle
```
The caller sends a subject with optional arguments and a handle. Before forwarding to the receiver, the hub injects `cast=` identifying the originating caller.

**Response (receiver → hub → caller):**
```
~handle:subject [key=value ...] ok capture=node_id
```
Every call receives exactly one response. The `~handle:subject` prefix binds the response to its originating call. The hub injects `ok` and `capture=` on every success response before delivering to the caller.

**Error response:**
```
~handle:subject error code=CODE what="description" capture=node_id
```
An error response uses the same shape as a success response. The presence of the `error` flag is the sole distinguishing marker; `ok` is never present on an error. Either the responding node or the hub itself may produce an error response.

Exactly one of `ok` or `error` is present on every response.

**Full exchange example:**
```
# Caller sends:
time.get_time timezone=UTC ~h1

# Hub forwards to receiver (cast= injected):
time.get_time timezone=UTC ~h1 cast=com.example.agent

# Receiver replies:
~h1:time.get_time time="2026-03-13T14:32:00Z"

# Hub forwards to caller (capture= and ok injected):
~h1:time.get_time time="2026-03-13T14:32:00Z" ok capture=com.example.clock
```

---

## 2. The `~handle`

`~handle` is a caller-supplied correlation token used to match a response to its originating call. **Every call must include a handle.** The hub rejects calls without one.

Rules:

- The **caller** generates the handle and appends it to the outgoing call
- The hub echoes it back on the response as `~handle:subject` — the `:subject` portion echoes what was called, making every response self-describing
- Handles are **global** — `~w1` is a single handle regardless of which node sent it. If a handle is already in use by another in-flight call, the hub rejects the call with `HandleInUse`. Nodes are responsible for choosing handles that are sufficiently unique to avoid collisions.
- Handle tokens are **arbitrary** — any string following `~` is valid. There is no semantic meaning encoded in the prefix or format. Choose handles that are convenient and sufficiently unique for your use case.

---

## 3. Argument Syntax

| Type | Syntax | Example |
|---|---|---|
| Key-value | `key=value` | `timezone=Europe/London` |
| String with spaces | `"..."` | `message="hello world"` |
| Array | `[...]` | `[a, b, c]` |
| Object | `{...}` | `{key: value}` |
| Flag (bare token) | `flag_name` | `recursive` |
| Boolean | `key=true\|false` | `followSymlinks=true` |
| Inline call (key binding) | `key=(subject [args...])` | `path=(dialog.file_picker)` |
| Inline call (spread) | `(subject [args...])` | `(dialog.dir.open)` |

**Double-quoted string escapes:**

Double-quoted strings support backslash escapes. The following sequences are recognised:

| Sequence | Meaning |
|---|---|
| `\\` | Literal backslash (`\`) |
| `\"` | Literal double-quote (`"`) |
| `\n` | Line feed (newline, U+000A) |
| `\r` | Carriage return (U+000D) |
| `\t` | Horizontal tab (U+0009) |

Any `\` followed by a character not in the table above is preserved as-is (the backslash and the character are both kept in the value).

Single-quoted strings have **no escape processing** — every character including `\` is literal. Because `'...'` is opaque and `"..."` supports `\"`, both quote styles can now express any string value:

```
msg="say \"hi\""          # double-quote inside a double-quoted string
msg='say "hi"'            # double-quote inside a single-quoted string (no escaping needed)
content="line1\nline2"    # newline embedded in a value
path="C:\\Users\\test"    # backslash (Windows path)
```

**Flags vs. Booleans:**

- **Flag:** A bare token that changes behavior when present. Declared as `type: flag` in the manifest. On the wire, the presence of the token means true; absence means false. Never has a value.
  ```
  filesystem.read path=/tmp recursive verbose
  ```

- **Boolean:** An explicit setting with a declared value. Declared as `type: boolean` in the manifest. On the wire, requires an explicit value (`true` or `false`).
  ```
  filesystem.open path=/tmp followSymlinks=true cacheMetadata=false
  ```

Use flags for behavior toggles; use booleans for explicit settings.

---

## 4. Inline Call Substitution

There are two inline call forms. Both are resolved **hub-side** before the outer call is forwarded to its receiver. The receiving node sees only the resolved values — it is unaware substitution occurred. The caster writes the inline syntax; the hub expands it transparently.

The hub generates an internal handle for each inner call using the `~~` prefix followed by a 4-character random hex token (e.g. `~~a3f9`). These hub-internal handles are distinct from caller-supplied handles and are never visible to the caster or receiver.

**Form 1 — Key binding: `key=(subject [args...])`**

The hub executes the inner call and binds one field from its response to the named key in the outer call. Field resolution follows this priority order:

1. If the inner response has exactly one data field (excluding protocol fields `ok`, `error`, `capture`, `cast`, `code`, `what`), that field is used regardless of its name.
2. If the inner response has multiple data fields, the hub looks for a field whose name matches the outer key. If found, it is used.
3. If neither condition is satisfied, the hub returns an error to the original caller — the field cannot be resolved.

```
SPORE.node.install path=(dialog.file_picker)
SPORE.node.install path=(dialog.file_picker filters=[*.yaml])
filesystem.read path=(shortpath project=notes)
```

**Form 2 — Spread: `(subject [args...])`**

The hub executes the inner call and spreads all data fields from its response directly into the outer call as if they had been written there by the caller. Protocol fields (`ok`, `error`, `capture`, `cast`, `code`, `what`) are stripped and never spread. Only one spread form may appear per outer call.

```
# Spreads all data fields from dialog.dir.open into the outer call
filesystem.list (dialog.dir.open) recursive
```

**Error handling (both forms):**

If the inner call responds with `error`, the hub halts and returns that error to the original caller. The outer call is never forwarded.

**Nesting:**

Inline calls may nest — an inner call may itself contain an inline call. Readability should guide usage; keep nesting shallow.

---

## 5. Errors

Error responses follow the same grammar as success responses. The presence of `error` marks the response as an error; `code=` carries the error identifier in `Code.Module` dot-notation; `what=` carries a human-readable description.

Every error response also carries exactly one **error origin flag** indicating where the error originated (see Section 6).

```
# Hub routing error:
~h1:time.get_time error code=Missing.Node what="no node connected for this subject" spore_error capture=SPORE.hub

# Hub argument error:
~h1:time.get_time error code=Missing.Message what="handle missing: every call must include a ~handle" cast_error capture=SPORE.hub

# Node-produced error:
~h1:time.get_time error code=Generic.Node what="internal clock failure" node_error capture=com.example.clock
```

The caller checks for `ok` or `error` on every response — exactly one will be present. When `error` is present, `code=` and `what=` will always be set.

**Error code format: `Code.Module`**

`code=` combines the error kind and the originating component as `Code.Module`, e.g. `Missing.Node`, `NotPermitted.Permission`, `Malformed.Message`. The two parts are independently meaningful — `Code` describes what went wrong, `Module` describes where.

*Codes:*

| Code | Meaning |
|---|---|
| `Generic` | A general-purpose error with no more specific category |
| `ConnectionFailure` | The connection between parties could not be established or was lost |
| `HandshakeDenial` | The hub refused to complete a node handshake |
| `InitializationFailure` | A component failed to initialize |
| `Malformed` | A message or value could not be parsed |
| `Missing` | An expected resource, argument, or route was not found |
| `NotApplicable` | The operation is not applicable in this context |
| `NotImplemented` | The subject is declared but not yet implemented |
| `NotPermitted` | The caller is not permitted to perform this action |
| `Timeout` | An operation did not complete within the allowed time |
| `UserDenial` | A user explicitly denied the operation |
| `HandleInUse` | A handle was reused before the previous call with that handle completed |
| `ReservedLanguage` | A reserved keyword or namespace was used where it is not permitted |

*Modules:*

| Module | Component |
|---|---|
| `Bus` | Internal message routing bus |
| `Hub` | Hub process |
| `Hyphae` | Hyphae user-space layer |
| `Manifest` | Manifest parsing or validation |
| `Message` | Wire message parsing |
| `Node` | Node lifecycle or connection |
| `Permission` | Permission management |
| `Policy` | Policy enforcement |
| `Registry` | Node registry |
| `Spore` | Hub API handler |

> Error events are also emitted on the witness stream as `spore_event` messages for passive observation (see [SPEC.md §9](SPEC.md)).

---

## 6. Reserved Keywords

The following argument names are reserved by the protocol. They may not be used as `input` or `output` names in any node manifest. The hub rejects manifests containing reserved keywords as argument names.

**Hub-injected routing fields** — written by the hub only, never by nodes or callers:

| Keyword | Type | Injected on | Meaning |
|---|---|---|---|
| `cast` | string | Call → receiver | Node ID of the originating caller |
| `capture` | string | Response → caller | Node ID of the node that handled the call |

**Response status fields** — exactly one of `ok`, `error`, `custom_error`, or `cancelled` is present on every response:

| Keyword | Role | Meaning |
|---|---|---|
| `ok` | flag | Hub-injected. Marks this response as a success. |
| `error` | flag | Written by the responding node or hub. Marks this response as an error. `code=` and `what=` will be set. |
| `code` | string | Error identifier in `Code.Module` dot-notation (see §5). e.g. `Missing.Node`, `NotPermitted.Permission`. |
| `what` | string | Human-readable error description. Always present with `error`. |

**Error origin flags** — exactly one is present on every error response. Indicates where the error originated. Useful for debugging and tracing.

| Keyword | Meaning |
|---|---|
| `spore_error` | The error originates from the hub or Spore OS runtime itself |
| `node_error` | The error originates from the receiving node |
| `cast_error` | The error is attributable to the caller (e.g. bad arguments, missing handle) |
| `capture_error` | The error is attributable to the responder's reply (e.g. malformed response from node) |

**Witness fields** — hub-injected on witness copies only. Never present on messages routed to callers or receivers:

| Keyword | Type | Meaning |
|---|---|---|
| `spore_time` | integer | Unix time in milliseconds since epoch (1970-01-01T00:00:00Z). Present on every witness copy. |
| `spore_incoming` | flag | Marks this witness copy as a raw inbound message — received from a node before hub injection or routing. |
| `spore_outgoing` | flag | Marks this witness copy as a message sent to a node — after hub injection (`cast=`, `capture=`, `ok`, etc.). |
| `spore_event` | flag | Marks this witness copy as a hub lifecycle or error event, not a routed message. |
| `spore_node` | flag | Marks this witness copy as a node-emitted observability message — sent by the node itself, not observed by the hub from routed traffic. |

Exactly one of `spore_incoming`, `spore_outgoing`, `spore_event`, or `spore_node` is present on every witness copy.

**Caller behavior flags** — added by the caller to the outgoing call. The hub forwards the call unchanged; receiver-side client libraries must ignore them. On the return path, the hub acts on any caller behavior flags before delivering the response:

| Keyword | Type | Meaning |
|---|---|---|
| `json` | flag | Request JSON-formatted output. The hub serializes the response to JSON before delivering it to the caller. Receiver-side client libraries must ignore this flag. |

**Reserved syntax tokens:**

| Token | Role |
|---|---|
| `~` prefix | Marks a caller-supplied correlation handle |
| `~~xxxx` prefix | Hub-internal handle for inline call expansion. 4-character random hex suffix (e.g. `~~a3f9`). Never used by callers. |
| `~handle:subject` | Response binding — correlates a response to its originating call and subject |
| `()` wrapper | Inline call substitution, resolved hub-side (see Section 4) |
| `witness` | Reserved line prefix. Witness copies sent to witness nodes are prefixed with this token. It is not a subject and cannot be used as one. |
| `publish` | Reserved line prefix. Broadcast messages sent by publishing nodes are prefixed with this token. It is not a subject and cannot be used as one. |

**Reserved namespace:**

- `SPORE.*` — all hub and protocol-internal subjects and error codes. No third-party node may register an `id` beginning with `SPORE.`.

**Reserved prefix:**

- `spore_` — all argument names beginning with `spore_` are reserved by the protocol. No node manifest may use a `spore_`-prefixed name as an input or output. The hub rejects manifests that violate this rule.
- `witness` — reserved as a line-prefix token on the wire. No subject may be named `witness` or begin with `witness ` (with a trailing space). The hub rejects manifests declaring such a subject.
- `publish` — reserved as a line-prefix token on the wire. No subject may be named `publish` or begin with `publish ` (with a trailing space). The hub rejects manifests declaring such a subject.

---

*Spore OS Protocol v0d2 — June 2026*
