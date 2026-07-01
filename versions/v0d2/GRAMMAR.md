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

**Cancelled response:**
```
~handle:subject cancelled capture=node_id
```
A cancelled response indicates a graceful non-result — the call completed without error, but produced no output because the user or system chose not to proceed (e.g. a user dismissed a dialog). The responding node writes `cancelled`; the hub injects `capture=` as it does for all responses. No data fields are present on a cancelled response.

Exactly one of `ok`, `error`, `custom_error`, or `cancelled` is present on every response.

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

1. If the inner response has exactly one data field (excluding protocol fields `ok`, `cancelled`, `error`, `custom_error`, `capture`, `cast`, `code`, `what`), that field is used regardless of its name.
2. If the inner response has multiple data fields, the hub looks for a field whose name matches the outer key. If found, it is used.
3. If neither condition is satisfied, the hub returns an error to the original caller — the field cannot be resolved.

```
SPORE.node.install path=(dialog.file_picker)
SPORE.node.install path=(dialog.file_picker filters=[*.yaml])
filesystem.read path=(shortpath project=notes)
```

**Form 2 — Spread: `(subject [args...])`**

The hub executes the inner call and spreads all data fields from its response directly into the outer call as if they had been written there by the caller. Protocol fields (`ok`, `cancelled`, `error`, `custom_error`, `capture`, `cast`, `code`, `what`) are stripped and never spread. Only one spread form may appear per outer call.

```
# Spreads all data fields from dialog.dir.open into the outer call
filesystem.list (dialog.dir.open) recursive
```

**Cancelled handling (both forms):**

If the inner call responds with `cancelled`, the hub halts immediately and does not forward the outer call. The `cancelled` response is returned to the original caller as the result of the entire expression. `cancelled` indicates the inner call completed gracefully but produced no usable output — this is not an error.

**Error handling (both forms):**

If the inner call responds with `error` or `custom_error`, the hub halts and returns that error to the original caller. The outer call is never forwarded.

**Nesting:**

Inline calls may nest — an inner call may itself contain an inline call. Readability should guide usage; keep nesting shallow.

---

## 5. Errors

Error responses follow the same grammar as success responses. The presence of `error` or `custom_error` marks the response as an error; `code=` carries a standard or custom identifier; `what=` carries a human-readable description.

- **`error`** — used for standard protocol errors and generic node errors. The `code=` value should be one of the standard error codes defined below.
- **`custom_error`** — used by a node when the error corresponds to a named entry in its manifest `errors` field (see [MANIFESTS.md §4](MANIFESTS.md)). The `code=` value is node-defined.

Every error response also carries exactly one **error origin flag** indicating where the error originated (see Section 6).

```
# Hub-produced routing error:
~h1:time.get_time error code=RouteNotConnected what="No node connected for this subject" spore_error capture=SPORE.hub

# Hub-produced argument validation error:
~h1:time.get_time error code=RequiredArgumentMissing what="Missing required argument: timezone" cast_error capture=SPORE.hub

# Node-produced generic error:
~h1:time.get_time error code=Runtime what="Internal clock failure" node_error capture=com.example.clock

# Node-produced custom_error (declared in manifest):
~h1:time.get_time custom_error code=clock.err.invalid_timezone what="Unknown timezone: Fakezone" node_error capture=com.example.clock
```

The caller checks for `ok`, `error`, `custom_error`, or `cancelled` on every response — exactly one will be present. When `error` or `custom_error` is present, `code=` and `what=` will always be set.

**Standard error codes:**

These are protocol-defined codes used with the `error` flag. Nodes should prefer these over inventing equivalent codes.

*Internal / meta:*

| Code | Meaning |
|---|---|
| `UnknownFailure` | An error occurred but the cause cannot be determined |
| `SporeFailure` | A failure internal to the Spore OS hub or runtime. **Hub-only** — the hub blocks this code if a node emits it. |
| `ProtocolFailure` | A violation of the Spore OS wire protocol |
| `ConnectionFailure` | The connection between parties could not be established or was lost |

*General:*

| Code | Meaning |
|---|---|
| `Generic` | A general-purpose error with no more specific category |
| `Fatal` | An unrecoverable error; the node or operation cannot continue |
| `Timeout` | An operation did not complete within the allowed time |
| `Busy` | The node or resource is temporarily unavailable |
| `ResourcesExhausted` | A resource limit has been reached (memory, file handles, etc.) |
| `Deprecated` | The called subject or argument is deprecated and no longer supported |
| `Runtime` | An error occurred during execution not attributable to caller inputs |
| `Logic` | An internal logic error or invariant violation in the node |

*Route:*

| Code | Meaning |
|---|---|
| `RouteNotFound` | No installed node declares this subject — no manifest in the registry owns it |
| `RouteNotConnected` | The subject is declared in an installed node's manifest, but that node is not currently connected |
| `RouteNotAvailable` | A matching node exists but cannot handle the call right now |
| `RouteNotAllowed` | The caller is not permitted to call this subject |
| `RouteNotImplemented` | The subject is declared in the manifest but not yet implemented |

*Message:*

| Code | Meaning |
|---|---|
| `MessageNotValid` | The message is structurally valid but semantically rejected |
| `MessageMalformed` | The message could not be parsed |

*Arguments:*

| Code | Meaning |
|---|---|
| `RequiredArgumentMissing` | A required argument was not provided |
| `ArgumentInvalidType` | An argument value is the wrong type |
| `ArgumentConflict` | Two or more arguments are mutually exclusive but both were provided |
| `ArgumentOutOfRange` | An argument value is outside the accepted range |
| `ArgumentUnrecognized` | An argument was provided that the subject does not accept |
| `ArgumentDuplicated` | The same argument key appears more than once |
| `ReservedKeyword` | A protocol-reserved keyword was used where it is not permitted (e.g. as an argument name) |

*Flags:*

| Code | Meaning |
|---|---|
| `FlagConflict` | Two or more flags are mutually exclusive but both were provided |
| `FlagUnrecognized` | A flag was provided that the subject does not accept |
| `FlagDuplicated` | The same flag appears more than once |

*Handles:*

| Code | Meaning |
|---|---|
| `HandleMissing` | A call was sent without a required `~handle` |
| `HandleInUse` | A handle was reused before the previous call with that handle completed |
| `HandleExpired` | A response arrived for a handle that is no longer active |

Node-defined custom error codes should be declared in the manifest `errors` list so callers know what to expect (see [MANIFESTS.md §4](MANIFESTS.md)).

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
| `error` | flag | Written by the responding node or hub. Marks this response as a standard protocol error. `code=` will be one of the standard error codes. |
| `custom_error` | flag | Written by the responding node. Marks this response as a node-defined custom error declared in the manifest `errors` field. |
| `cancelled` | flag | Written by the responding node. Marks this response as a graceful non-result — the call completed without error but produced no output (e.g. a user dismissed a dialog). No data fields will be present. `ok` is never present alongside `cancelled`. |
| `code` | string | Error identifier. Always present with `error` or `custom_error`. Standard codes are unnamespaced (e.g. `RouteNotFound`); custom codes follow the `nodename.err.reason` convention. |
| `what` | string | Human-readable error description. Always present with `error` or `custom_error`. |

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
