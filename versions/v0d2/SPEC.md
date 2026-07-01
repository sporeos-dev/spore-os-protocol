<!--
Copyright 2026 Matt Harrison
SPDX-License-Identifier: Apache-2.0
-->

# Spore OS Protocol Specification
**Version:** v0d2  
**Schema:** `SPORE/v0d2`  
**Date:** 2026-06-30  
**Status:** Development

## 1. Overview

Spore OS is a **local IPC protocol**. It provides a socket interface that lets applications discover each other and exchange messages — without knowing anything about each other's implementation.

The core idea is **treating applications as functions**: a node exposes capabilities, and the hub routes calls between them. A node can be a CLI tool, a background daemon, a GUI application, or a shell script.

Spore OS is not a framework, not a service mesh, and is explicitly not performance-sensitive. It is a simple, human-readable message bus for local processes.

---

## 2. Core Concepts

### 2.1 Hub

The hub is the local daemon. It:

- Routes messages between nodes based on **manifest knowledge**, not message syntax
- Maintains a **manifest registry** — nodes are known to the hub when installed, whether or not they are currently running
- Fans out **broadcast messages** to subscribed nodes on declared topics

The hub has its own manifest (`dev.sporeos.SPORE`) which defines its API. The hub manifest is part of this specification.

### 2.2 Nodes

A node is any process that connects to the hub via a Unix domain socket and presents a manifest. Nodes are:

- **Implementation-agnostic** — a node can be anything: a script, a CLI, a GUI app, a daemon
- **Self-describing** — every node has a manifest that is its full contract

Nodes exist in one of two states:

| State | Meaning |
|---|---|
| **Installed** | The hub has the manifest; the node is not running |
| **Connected** | The node is running and has completed handshake |

### 2.3 Manifest

The manifest is the **contract and API** for a node. It is the authoritative description of everything a node can do. The hub routes every message by resolving it against the manifest registry.

The manifest is **read-only at runtime**. Changes require a node restart.

### 2.4 The `SPORE.` Namespace

`SPORE.` is protocol-reserved. No third-party node may register an `id` beginning with `SPORE.`. The hub rejects any handshake presenting a `SPORE.` ID unless it is a recognised hub-internal node.

### 2.5 Trust and Risk

Every node has a **trust level** declared in its manifest. Every API entry and topic has a **risk level**. Together, these form the access control model for capability access.

- **Trust** is assigned to a node at install time. It describes how much authority that node is granted.
- **Risk** is declared by the node author on each capability. It describes how sensitive the capability is.

The hub enforces the relationship between them according to system policy and user preferences. See Section 11 for the full model.

---

## 3. Transport

### 3.1 Socket

Spore OS uses **Unix domain sockets** (UDS) for local communication.

- **macOS / Linux:** Unix domain socket at a conventional path
- **Windows:** Unix domain socket via the Windows UDS implementation

The canonical socket path for each platform is defined by the platform installer. All implementations (hub, CLI, nodes) on a given host must use the same path. No implementation should hardcode a socket path independently of the installer's definition.

### 3.2 Encoding

All messages are **newline-delimited plaintext strings**. No binary framing. Type information declared in the manifest is advisory — it helps implementors know what to expect, but the hub does not validate or coerce types.

### 3.3 Manifest Format

Manifests are authored in **YAML**. The hub reads the YAML file directly.

---

## 4. Handshake

**Installation** and **connection** are separate concerns.

**Installation** registers a node's manifest with the hub out-of-band — typically via a CLI command that reads the node's YAML manifest file. Installation persists across hub restarts. The hub can route to an installed node's capabilities even when the node is not running.

**Connection** is the runtime identification of a running node. When a node starts, it opens a connection to the hub socket and sends:

```
com.example.clock
```

The hub responds:

```
OK
```

The hub looks up the installed manifest for `com.example.clock` and marks the node as connected. If the node ID is not known, the hub responds with an error.

If a node disconnects after handshake, it reverts to installed-but-not-connected in the registry.

---

## 5. Addressing

All messages use dot-notation subjects. A subject is a single string token — the dots are a naming convention, not syntax the hub parses structurally.

```
subject [key=value ...] [~handle]
```

The subject on the wire must match the `name` field of an API entry in a node's manifest exactly as declared. The hub routes by exact match against registered command names.

### 5.1 Routing

The hub resolves the subject against its manifest registry and routes to the connected node that exposes it.

```
clock.get_time              # matches manifest api.name exactly
SPORE.node.list             # hub internal
```

---

## 6. Wire Grammar

### 6.1 Message Shapes

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

### 6.2 The `~handle`

`~handle` is a caller-supplied correlation token used to match a response to its originating call. **Every call must include a handle.** The hub rejects calls without one.

Rules:

- The **caller** generates the handle and appends it to the outgoing call
- The hub echoes it back on the response as `~handle:subject` — the `:subject` portion echoes what was called, making every response self-describing
- Handles are **global** — `~w1` is a single handle regardless of which node sent it. If a handle is already in use by another in-flight call, the hub rejects the call with `HandleInUse`. Nodes are responsible for choosing handles that are sufficiently unique to avoid collisions.
- Handle tokens are **arbitrary** — any string following `~` is valid. There is no semantic meaning encoded in the prefix or format. Choose handles that are convenient and sufficiently unique for your use case.

### 6.3 Argument Syntax

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

### 6.4 Inline Call Substitution

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

### 6.5 Errors

Error responses follow the same grammar as success responses. The presence of `error` or `custom_error` marks the response as an error; `code=` carries a standard or custom identifier; `what=` carries a human-readable description.

- **`error`** — used for standard protocol errors and generic node errors. The `code=` value should be one of the standard error codes defined below.
- **`custom_error`** — used by a node when the error corresponds to a named entry in its manifest `errors` field (see Section 8.4). The `code=` value is node-defined.

Every error response also carries exactly one **error origin flag** indicating where the error originated (see Section 6.6).

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

Node-defined custom error codes should be declared in the manifest `errors` list so callers know what to expect (see Section 8.4).

> Error events are also emitted on the witness stream as `spore_event` messages for passive observation (see Section 9).

### 6.6 Reserved Keywords

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
| `()` wrapper | Inline call substitution, resolved hub-side (see Section 6.4) |
| `witness` | Reserved line prefix. Witness copies sent to witness nodes are prefixed with this token. It is not a subject and cannot be used as one. |
| `publish` | Reserved line prefix. Broadcast messages sent by publishing nodes are prefixed with this token. It is not a subject and cannot be used as one. |

**Reserved namespace:**

- `SPORE.*` — all hub and protocol-internal subjects and error codes. No third-party node may register an `id` beginning with `SPORE.`.

**Reserved prefix:**

- `spore_` — all argument names beginning with `spore_` are reserved by the protocol. No node manifest may use a `spore_`-prefixed name as an input or output. The hub rejects manifests that violate this rule.
- `witness` — reserved as a line-prefix token on the wire. No subject may be named `witness` or begin with `witness ` (with a trailing space). The hub rejects manifests declaring such a subject.
- `publish` — reserved as a line-prefix token on the wire. No subject may be named `publish` or begin with `publish ` (with a trailing space). The hub rejects manifests declaring such a subject.

---

## 7. Call / Response

A node calls another node's API subject. Each API entry is a `query` (returns data) or a `command` (triggers behaviour).

### 7.1 Query

Queries return data. The response arrives as `~handle:subject` with result fields and a hub-injected `capture=`.

```
clock.get_time ~h1
← ~h1:clock.get_time time="2026-03-13T14:32:00Z" ok capture=com.example.clock

filesystem.read path=/documents/notes.txt ~r1
← ~r1:filesystem.read content="Q1 planning notes..." ok capture=com.example.filesystem

SPORE.node.list ~meta
← ~meta:SPORE.node.list nodes=["com.example.clock", "com.example.filesystem"] ok capture=SPORE.hub
```

### 7.2 Command

Commands trigger behaviour. Unlike queries, a command may or may not return data — this is declared in the manifest's `outputs` field.

A command with no declared `outputs` returns an `ok` response — a response with no data fields, only the subject echo, `ok`, and `capture=`. The `ok` flag confirms the command completed without error.

```
# Command with no outputs (void return):
logger.append entry="started" ~l1
← ~l1:logger.append ok capture=com.example.logger

# Command that returns data:
filesystem.write path=/tmp/out.txt content="hello" ~w1
← ~w1:filesystem.write ok capture=com.example.filesystem

# Error on a command:
logger.append entry="started" ~l2
← ~l2:logger.append error code=logger.err.disk_full what="No space remaining" capture=com.example.logger
```

### 7.3 Composition

The output of one call feeds the next. The orchestrating node sequences calls itself; individual nodes are unaware of each other. The hub-level mechanism for this is inline call substitution (see Section 6.4); beyond that, composition is a node implementation concern.

---

## 8. Manifest Specification

### 8.1 Structure

```yaml
id: com.example.clock
name: Clock
version: 1.0.0
schema: SPORE/v0d2
description: Provides time-related functions.
trust: standard
api: []       # Direct call/response subjects
topics: []    # Topics this node publishes
errors: []    # Custom node-defined errors this node may emit
witness: false # If true, this node receives all witness traffic from the hub
```

### 8.2 Metadata Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | ✅ | Reverse-domain unique identifier |
| `name` | string | ✅ | Human-readable name |
| `app` | string | ✅ | How to invoke the application (e.g., `./clock`, `./clock.exe`, `python -m myclock`, `node app.js`). Use `"n/a"` for system nodes. Relative paths are relative to the manifest directory. |
| `schema` | string | ✅ | Manifest schema version (`SPORE/v0d2`) |
| `version` | string | | Node API version (`MAJOR.MINOR.PATCH`). Informational. |
| `description` | string | ✅ | Short description of the node's purpose |
| `trust` | string | | Trust level of the node: `system`, `developer`, `trusted`, `standard`, `untrusted`. Assigned at install time. Default: `untrusted`. See Section 11. |
| `autostart` | boolean | | If `true`, the hub spawns this node automatically when the hub starts in a headless manner (no UI or interactive context). Default: `false`. Only meaningful if `app` is set to a real executable path. |
| `witness` | boolean | | If `true`, this node receives all witness traffic from the hub (see Section 9). Default: `false`. |

### 8.3 API

Defines every callable subject this node exposes.

```yaml
api:
  - name: get_time
    description: Returns the current time.
    usage:
      - clock.get_time
      - clock.get_time timezone=America/New_York
    inputs:
      - name: timezone
        type: string
        required: false
        description: IANA timezone. Defaults to UTC.
    outputs:
      - name: time
        type: string
        description: Current time in ISO 8601 format.

  - name: file_picker
    description: Opens a file picker dialog.
    usage:
      - dialog.file_picker
      - dialog.file_picker filters=[*.yaml]
    inputs:
      - name: filters
        type: array
        required: false
        description: File type filters (e.g., "*.yaml", "*.txt").
    outputs:
      - name: path
        type: string
        description: The selected file path.

  - name: set_timer
    description: Sets a one-shot timer. Returns no data — an acknowledgment is sent on completion.
    usage:
      - clock.set_timer duration=1h
      - clock.set_timer duration=30m label="break"
    inputs:
      - name: duration
        type: string
        required: true
        description: e.g. 1h, 30m, 45s
      - name: label
        type: string
        required: false
        description: Optional label for the timer.
```

**API entry fields:**

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | ✅ | Subject name |
| `description` | string | ✅ | What this entry does |
| `usage` | array | ✅ | Example calls demonstrating usage |
| `inputs` | array | | Input arguments. Omit if none. |
| `outputs` | array | | Success-path data fields returned by this entry. Omit if the call returns no data (`ok` is still sent by the hub). Protocol-level outcomes — `cancelled`, `error`, `custom_error` — are not declared here; they are implied for any entry. If `outputs` is not included in the manifest, it is treated as an empty list. |
| `notes` | array | | Optional clarifying notes (list of strings). Useful for documenting constraints that don't fit neatly into structured fields — for example, "either `node` or `path` is required, but not both" — or other implementation guidance for callers. |

**Input entry fields:**

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | ✅ | Argument name |
| `type` | string | ✅ | Type: `string`, `integer`, `float`, `boolean`, `array`, `object`, `flag`, etc. |
| `required` | boolean | | Whether this argument must be provided. Default: false. Should be false for `type: flag` (flags are always optional). |
| `description` | string | ✅ | What this argument does. |

**Output entry fields:**

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | ✅ | Output field name |
| `type` | string | ✅ | Type hint (same as inputs) |
| `description` | string | ✅ | What this output field contains. |

**Input Type Examples:**

```yaml
inputs:
  - name: path
    type: string
    required: true
    description: File path to read.
    
  - name: recursive
    type: flag
    description: Include subdirectories.
    
  - name: followSymlinks
    type: boolean
    description: Whether to follow symbolic links.
    
  - name: limit
    type: integer
    required: false
    description: Maximum number of results to return.
```

Wire calls:
```
filesystem.read path=/tmp recursive              # flag present
filesystem.open path=/tmp followSymlinks=true    # boolean explicit
filesystem.list path=/home limit=100 recursive   # both types together
```

### 8.4 Custom Errors

Nodes may declare their custom error codes in the top-level `errors` field. Each entry gives callers structured documentation about what the error means and when it may occur. When emitting a declared error, the node uses the `custom_error` flag instead of `error`.

```yaml
errors:
  - name: clock.err.invalid_timezone
    description: The provided timezone string is not a valid IANA timezone identifier.
    examples:
      - "Timezone string is misspelled (e.g. 'Amercia/New_York')"
      - "Timezone is not present in the IANA timezone database"
      - "An empty string was provided for the timezone argument"

  - name: clock.err.timer_limit_reached
    description: The node has reached its maximum number of concurrent active timers.
    examples:
      - "Calling set_timer when the node is already tracking the maximum allowed timers"
```

**Custom error entry fields:**

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | ✅ | The error code, by convention `nodename.err.reason` |
| `description` | string | ✅ | What this error means |
| `examples` | array of strings | | Potential causes or scenarios that trigger this error |

A node emitting a declared custom error puts `custom_error` (not `error`) on the wire response, pairs it with the matching `code=`, and includes a human-readable `what=`:

```
~h1:clock.get_time custom_error code=clock.err.invalid_timezone what="Unknown timezone: Fakezone" capture=com.example.clock
```

The `error` flag remains available for ad-hoc or unexpected errors that were not anticipated at manifest-authoring time.

---

## 8.5 Topics

A node that publishes topics declares them in the top-level `topics` field. Topics are broadcast (push-only) — distinct from `api` entries, which are request/response.

```yaml
topics:
  - name: clock.ticked
    description: Published every minute.
    risk: benign
    usage:
      - publish clock.ticked time="2026-06-30T12:00:00Z"
    outputs:
      - name: time
        type: string
        description: Current time in ISO 8601 format.
```

**Topic entry fields:**

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | ✅ | Subject name for the topic |
| `description` | string | ✅ | What this topic represents |
| `usage` | array | ✅ | Example `publish` calls demonstrating the message shape |
| `risk` | string | ✅ | Risk level of the topic (see Section 11) |
| `outputs` | array | | Fields present in the published message. Describes what is broadcast — there is no input from subscribers. |

By convention, topic names use past tense to distinguish them from commands (e.g. `SPORE.node.spawned` vs `SPORE.node.spawn`). This is convention only — the hub does not enforce naming style.

---

## 8.6 Manifest Location Convention

Manifests and their applications follow a co-location convention:

- Manifest file is named `<app_name>.manifest.spore.yaml` (where `app_name` is derived from the `name` field)
- Manifest lives in the same directory as the application
- The `app` field in the manifest specifies how to invoke the application, typically as a relative path

Example directory structure:
```
<nodes-dir>/clock/
  clock                      (executable)
  clock.manifest.spore.yaml

<nodes-dir>/dialog/
  dialog                      (executable or script)
  dialog.manifest.spore.yaml
```

The actual location of `<nodes-dir>` is platform-specific and defined by the platform installer.

When installing a node, the installer:
1. Reads the manifest file (via path selection or dialog)
2. Loads and validates the manifest
3. Registers the manifest with the hub
4. The hub uses the `app` field to launch the application when messages arrive

---

## 9. Witness

### 9.1 Architecture

The hub dispatches a copy of every message — incoming and outgoing — to all registered witness nodes as a fire-and-forget side channel. The hub does not wait for witnesses and does not care if dispatch fails.

- Zero witnesses is a valid operational state.
- Multiple witnesses receive identical raw copies independently.
- Witnesses are responsible for all filtering, formatting, prettification, and storage. The hub makes no decisions about these.

A node registers as a witness by declaring `witness: true` in its own manifest. The hub sends it all witness traffic — there is no per-node scoping.

### 9.2 What Witnesses Receive

Every witness copy is prefixed with the reserved token `witness` followed by a space, then the wire-format message body. This prefix allows a witness node's client library to distinguish witness copies from normal calls and responses on the same socket connection — without inspecting content.

Examples:

```
witness clock.get_time cast=com.example.shell ~h1 spore_incoming spore_time=1744732800000
witness ~h1:clock.get_time time="2026-03-13T14:32:00Z" ok capture=com.example.clock spore_outgoing spore_time=1744732800001
```

The body after `witness ` carries one of four flags indicating what the copy represents:

| Flag | Meaning |
|---|---|
| `spore_incoming` | Raw message as received from a node, before any hub injection or routing |
| `spore_outgoing` | Message as delivered to a node, after hub injection (`cast=`, `capture=`, `ok`, etc.) |
| `spore_event` | A hub lifecycle or internal error event (not a routed message) |
| `spore_node` | A node-emitted observability message, not observed traffic (see Section 9.4) |

Exactly one of these four flags is present on every witness copy. All four types also carry `spore_time=` (Unix milliseconds since epoch).

> **Implementation note:** Implementations may include additional non-normative metadata fields on witness copies beyond those listed above (for example, a per-connection sequential message index). These are informational extensions and are not required by the protocol.

### 9.3 Hub Events as Messages

Hub lifecycle and error events (autostart failures, handshake failures, routing errors) are emitted as `spore_event` witness messages:

```
witness SPORE.hub.event level=warn what="Unable to complete handshake" error="..." spore_event spore_time=1744732800000
```

This allows a witness to correlate hub events with message traffic by timestamp.

### 9.4 Node-Emitted Witness Messages

A node may emit observability messages directly to the witness stream without those messages being part of any call or response. This is distinct from the hub observing traffic — the node is proactively publishing a diagnostic, audit, or lifecycle event of its own.

A node emits a node witness message by sending a line prefixed with the reserved `witness` token directly on its socket connection:

```
witness message="loaded config"
```

The hub intercepts this line before routing (it is never treated as a call or subject). It appends `cast=` identifying the emitting node and `spore_time=`, tags it `spore_node`, and dispatches to all registered witnesses. No response is sent back to the node — this is fire-and-forget.

```
witness message="loaded config" cast=com.example.mynode spore_node spore_time=1744732800000
```

The body after the `witness ` prefix is free-form. Nodes are responsible for choosing a format useful to their witnesses.

### 9.5 Hub Output

The hub emits no structured log of its own during normal operation. The sole exception is stderr for witness handshake failures — one line per failed witness, surfacing the error code and reason. All other observability flows through the witness stream.

### 9.6 Core Witness Nodes

The hub ships two built-in witness nodes. See [HUB.md](HUB.md) for details.

### 9.7 Failure Mode

Always passthrough — the hub never blocks a call due to a witness being absent or unreachable.

---

## 10. Broadcast

### 10.1 Architecture

Broadcast is a **publish/subscribe channel**. A node that declares topics in its manifest may publish to those topics at any time. The hub fans out each published message to all currently subscribed nodes.

- Only the node that declared a topic in its manifest may publish to it. Attempts by any other node to publish result in `RouteNotAllowed`.
- Subscriptions are managed at runtime via `SPORE.topic.subscribe` and `SPORE.topic.unsubscribe`.
- Nodes are responsible for subscribing on connect and unsubscribing on disconnect.
- If no subscribers are connected when a message is published, the message is silently dropped.
- Delivery is **fire-and-forget** — the hub does not wait for subscribers and does not retry.
- The hub does not buffer undelivered messages.

### 10.2 Wire Format

A broadcast message uses the reserved `publish` line prefix. No `~handle` is included — broadcast is one-way with no response:

```
publish subject [key=value ...] [flags]
```

**Example (publisher sends):**
```
publish SPORE.node.spawned node=com.example.clock
```

The hub injects `cast=` (publisher ID) before fanning out. Each subscriber receives a delivery with their own `capture=`:

```
publish SPORE.node.spawned node=com.example.clock cast=dev.sporeos.SPORE capture=com.example.monitor
```

`cast=` identifies the publishing node. `capture=` identifies this specific subscriber — each delivery carries the individual subscriber's ID.

### 10.3 Topic Ownership

A topic is declared in the publishing node's manifest under the `topics` field. Only that node may publish to it. The hub resolves topic ownership against the manifest registry, exactly as it resolves API subject ownership.

### 10.4 Subscriptions

Subscriptions are managed via hub API calls and are **not persistent** across disconnect/reconnect. A node is responsible for re-subscribing each time it connects:

```
SPORE.topic.subscribe topic=SPORE.node.spawned ~h1
← ~h1:SPORE.topic.subscribe ok capture=SPORE.hub

SPORE.topic.unsubscribe topic=SPORE.node.spawned ~h2
← ~h2:SPORE.topic.unsubscribe ok capture=SPORE.hub
```

### 10.5 Witness Integration

All broadcast traffic flows through the witness stream. Published messages appear as `spore_incoming` (the `publish` line from the publisher) and `spore_outgoing` (each fan-out delivery to a subscriber). Witness nodes see all published messages regardless of whether they have subscribed to a topic.

### 10.6 Trust and Risk

Topic subscriptions are subject to the same trust and risk policy as request/response calls. The hub enforces this at subscription time — `SPORE.topic.subscribe` returns `RouteNotAllowed` if the subscribing node's trust level does not satisfy the topic's risk policy.

---

## 11. Trust & Risk

### 11.1 Overview

Every node has a **trust level** declared in its manifest. Every API entry and topic has a **risk level**. Together, these define the security boundary for capability access.

Trust is assigned at install time and reflects how much authority the node is granted. Risk is declared by the node author and reflects how sensitive the capability is. The hub enforces the relationship between them according to system policy and user preferences.

### 11.2 Trust Levels

| Level | Meaning |
|---|---|
| `system` | Hub-internal nodes only. Governed by a hardcoded hub whitelist — cannot be assigned to third-party nodes at install. |
| `developer` | Full trust — bypasses all user preferences and policy. For local development only. The hub emits a warning whenever a `developer`-trust node connects. Do not use in production. |
| `trusted` | High trust. Access governed by policy; typically broader than `standard`. |
| `standard` | Regular trust. Expected for well-known, vetted nodes. |
| `untrusted` | All permissions must be explicitly granted. No implicit access beyond `benign` APIs. Default for nodes without an explicit trust assignment. |

Trust is set in the manifest by the node author. A user may override trust at install time — with the following constraints:

- **`developer`** — the hub warns that this level bypasses all policy and should not be used for production nodes.
- **`system`** — cannot be assigned via install; reserved for the hardcoded hub whitelist.

### 11.3 Risk Levels

| Level | Meaning |
|---|---|
| `benign` | No sensitive access. Open to all callers regardless of trust level. |
| `standard` | Minor capability. Access governed by policy. |
| `personal` | Access to personal data. Requires explicit user policy grant. Hub warns on first access request. |
| `secret` | Access to credentials, keys, or other sensitive material. Requires explicit user policy grant. Hub warns on first access request. |
| `protected` | System-critical capabilities. Access governed by a hardcoded hub whitelist. Cannot be unlocked by user preferences or policy. |

### 11.4 Default Policy

| Risk | Default |
|---|---|
| `benign` | **Open** — all callers may access, regardless of trust level |
| `standard` | Policy-defined — default behaviour is determined by the caller's trust level |
| `personal` | **Deny** by default. Requires explicit user grant. Hub warns on first grant request. |
| `secret` | **Deny** by default. Requires explicit user grant. Hub warns on first grant request. |
| `protected` | **Deny** to all except hub-whitelisted nodes. Cannot be overridden by user policy. |

> **Known gap:** Runtime enforcement of the middle tiers (`standard`, `personal`, `secret`) and the user preference mechanism are not yet fully defined. The model and manifest fields are established in v0d2; the enforcement matrix and permission API are open issues. See [OPEN.md](OPEN.md).

### 11.5 Developer Trust Warning

When a node with `developer` trust connects, the hub emits a warning on the witness stream:

```
witness SPORE.hub.event level=warn what="Node com.example.mynode has developer trust. All policy is bypassed. Do not use in production." spore_event spore_time=...
```

### 11.6 Protected Resources

`protected` risk capabilities and `system` trust assignments are governed by a whitelist hardcoded in the hub binary. This cannot be modified by config file or runtime call — the hub binary is the trust anchor for protected resources. Nodes not on the whitelist receive `RouteNotAllowed` regardless of their stated trust level.

### 11.7 Inline Call Trust Preservation

When the hub expands an inline call (see Section 6.4), the inner call is forwarded with `cast=<original_caller>`. The hub never substitutes its own identity (`cast=SPORE.hub`) for hub-generated inner calls. This ensures that inline call substitution cannot be used to escalate privileges to a higher trust level.

---

## 12. Hub Contract

The normative hub contract — the `SPORE.*` subjects and topics a conforming hub must expose — is defined in [HUB.md](HUB.md).

---

*Spore OS Protocol v0d2 — June 2026*
