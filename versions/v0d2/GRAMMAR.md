<!--
Copyright 2026 Matt Harrison
SPDX-License-Identifier: Apache-2.0
-->

# Spore OS Protocol — Wire Grammar
**Version:** v0d2  
**Schema:** `SPORE/v0d2`  
**Spec:** [SPEC.md](SPEC.md)

This document defines the exact syntax of all messages on the Spore OS wire — message shapes, handles, argument syntax, pipelines, error grammar, and reserved keywords.

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
| Block value | `key=<<` | `content=<<` |

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

**Block Values:**

A `key=<<` argument signals that the value for that key spans multiple lines. Block content begins on the line immediately following the opening call line and continues until a `>>` line is encountered.

The `>>` line terminates the current block. Any remaining tokens on the `>>` line are parsed as call arguments — flags, key-value pairs, and further `key=<<` openings are all valid. The message is complete when a `>>` line is parsed and no open block remains.

```
filesystem.write path=/tmp/out.txt content=<< ~w1
line one
line two
>>
```

Multiple blocks in a single call — each `>>` closes the current block and argument parsing resumes on the same line:

```
diff.compare before=<<
original line one
original line two
>> after=<<
new line one
new line two
>> ~d1
```

Flags and other arguments may appear on `>>` lines alongside new block openings:

```
something.something before=<<
one line
two line
>> verbose continue=<<
three line
four line
>> ~w1
```

Rules:

- `<<` is only valid as the value of a named key — `key=<<`. A bare `<<` not bound to a key is a parse error.
- `>>` must be the first token on its line. Tokens following `>>` on the same line are parsed as call arguments.
- Block content is **raw** — no argument parsing and no escape processing. Blank lines within a block are preserved.
- A `>>` line that leaves no open blocks completes the message and triggers dispatch.
- The `~handle` may appear anywhere in the argument list — on the opening line, on any `>>` line, or as the last token on the final `>>`.

---

## 4. Pipelines

A pipeline sequences calls hub-side with `|`. The hub executes stages from left to right. The response data from one stage is forwarded as arguments to the next, and each receiving node sees only a normal call with resolved arguments.

```
subject [key=value ...] [flag ...] ~handle [| subject [key=value ...] [flag ...]] ...
```

```
dialog.file_picker ~h2 | SPORE.node.install
dialog.dir.open ~h3 | filesystem.list recursive
source.open ~h4 | transform.normalize | destination.write
```

The first stage is the only stage written with a caller-supplied `~handle`; later stages must not include one. The handle identifies the entire pipeline. The hub generates internal handles for intermediate stages; their format and observability are implementation details. The caller receives only the final stage's response, under the original handle.

Intermediate stages may be routed internally with `cast=<hub pipeline identity>` rather than the original caller, since the hub itself is correlating that stage's response — this is a routing implementation detail. The final stage is always forwarded with `cast=<original_caller>`, identical to a non-piped call. Regardless of the wire-level `cast=` used for internal routing, trust and permission evaluation for every stage is always performed against the original caller (see [SPEC.md §11.6](SPEC.md)).

For each subsequent stage, the hub forwards data fields and non-protocol flags from the preceding successful response when their names match inputs declared by that stage. Fields and flags that do not match an input are ignored. Protocol fields (`ok`, `error`, `capture`, `cast`, `code`, `what`) are never forwarded.

Each stage may include its own explicit arguments and flags. If a forwarded field or flag duplicates an explicit argument or flag, the hub returns an error to the original caller and does not execute that stage.

If any stage responds with `error`, the hub halts the pipeline and returns that error to the original caller. Later stages are never executed.

The hub imposes a default timeout on each pipeline stage while waiting for that stage's response; exceeding it halts the pipeline with `Timeout.Pipe`. A stage may override this with the reserved `timeout=` argument (see Section 6) — useful for stages that depend on user interaction (e.g. a file picker dialog) and may legitimately take longer than the default.

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

**Conventional argument** — not reserved from manifest use, but carries protocol-defined meaning where it appears:

| Keyword | Type | Meaning |
|---|---|---|
| `body` | string | The primary content of a message. Required on all `witness` lines sent by nodes, where additional key-value pairs may follow as structured metadata. Also carries the target or partial expression passed to the introspection subjects (see [HUB.md](HUB.md)). |

Exactly one of `spore_incoming`, `spore_outgoing`, `spore_event`, or `spore_node` is present on every witness copy.

**Pipeline stage control arguments** — reserved only within a pipeline stage (see Section 4); the hub consumes them and never forwards them to the receiving node:

| Keyword | Type | Meaning |
|---|---|---|
| `timeout` | integer or `false` | Overrides the hub's default per-stage wait for that stage's response. An integer is the number of seconds to wait before halting the pipeline with `Timeout.Pipe`. `false` disables the timeout for that stage and waits indefinitely. |

**Caller behavior flags** — added by the caller to the outgoing call. The hub forwards the call unchanged; receiver-side client libraries must ignore them. On the return path, the hub acts on any caller behavior flags before delivering the response:

| Keyword | Type | Meaning |
|---|---|---|
| `json` | flag | Request JSON-formatted output. The hub serializes the response to JSON before delivering it to the caller. Receiver-side client libraries must ignore this flag. |

**Reserved syntax tokens:**

| Token | Role |
|---|---|
| `~` prefix | Marks a caller-supplied correlation handle |
| `~handle:subject` | Response binding — correlates a response to its originating call and subject |
| `|` | Pipeline separator, resolved hub-side (see Section 4) |
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
