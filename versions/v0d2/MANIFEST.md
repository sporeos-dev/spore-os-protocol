<!--
Copyright 2026 Matt Harrison
SPDX-License-Identifier: Apache-2.0
-->

# Spore OS Protocol — Manifest Reference
**Version:** v0d2  
**Schema:** `SPORE/v0d2`  
**Spec:** [SPEC.md](SPEC.md)

This document is the full reference for the Spore OS manifest format — the YAML file that is every node's complete contract. It covers all fields, their types, and examples across metadata, API subjects, topics, custom errors, and file layout conventions.

---

## 1. Structure

A manifest is a YAML file with the following top-level shape:

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

---

## 2. Metadata Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | ✅ | Reverse-domain unique identifier |
| `name` | string | ✅ | Human-readable name |
| `app` | string | ✅ | How to invoke the application (e.g., `./clock`, `./clock.exe`, `python -m myclock`, `node app.js`). Use `"n/a"` for system nodes. Relative paths are relative to the manifest directory. |
| `schema` | string | ✅ | Manifest schema version (`SPORE/v0d2`) |
| `version` | string | | Node API version (`MAJOR.MINOR.PATCH`). Informational. |
| `description` | string | ✅ | Short description of the node's purpose |
| `trust` | string | | Trust level of the node: `system`, `developer`, `trusted`, `standard`, `untrusted`. Assigned at install time. Default: `untrusted`. See [SPEC.md §11](SPEC.md). |
| `autostart` | boolean | | If `true`, the hub spawns this node automatically when the hub starts in a headless manner (no UI or interactive context). Default: `false`. Only meaningful if `app` is set to a real executable path. |
| `witness` | boolean | | If `true`, this node receives all witness traffic from the hub (see [SPEC.md §9](SPEC.md)). Default: `false`. |

---

## 3. API

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

**Input type examples:**

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

---

## 4. Custom Errors

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

## 5. Topics

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
| `risk` | string | ✅ | Risk level of the topic (see [SPEC.md §11](SPEC.md)) |
| `outputs` | array | | Fields present in the published message. Describes what is broadcast — there is no input from subscribers. |

By convention, topic names use past tense to distinguish them from commands (e.g. `SPORE.node.spawned` vs `SPORE.node.spawn`). This is convention only — the hub does not enforce naming style.

---

## 6. Manifest Location Convention

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

*Spore OS Protocol v0d2 — June 2026*
