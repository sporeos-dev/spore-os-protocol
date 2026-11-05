<!--
Copyright 2026 Matt Harrison
SPDX-License-Identifier: Apache-2.0
-->

# Spore OS Protocol — Manifest Reference
**Version:** v0d3  
**Schema:** [spore.manifest.v0.schema.yaml](spore.manifest.v0.schema.yaml)  

Every node has a manifest that tells Spore what it can do. Spore checksums it
when you install the node; to change it, reinstall the node.

## Root manifest

```yaml
id: com.example.clock
name: Clock
app: ./clock
version: 1.0.0
schema: SPORE/v0d3
description: Provides time-related capabilities.
trust: standard
launch: lazy
namespace: hyphae
roles:
  - raw
  - hint
  - witness

api: []
topics: []

permissions:
  - name: some.capability
    reasons:
      - Stated reason for the permission.
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `id` | string | Yes | Reverse-domain identifier; `:` is reserved. |
| `name` | string | Yes | Human-readable name. |
| `app` | string | Yes | Invocation command; relative paths are relative to the manifest directory. Use `n/a` for system nodes without a separate executable. |
| `schema` | string | Yes | Manifest schema version, e.g. `SPORE/v0d3`. |
| `description` | string | Yes | Node purpose. |
| `version` | string | No | Informational API version in `MAJOR.MINOR.PATCH` format. |
| `trust` | string | No | Trust level; defaults to `untrusted`. See [SECURITY.md](SECURITY.md#trust-levels). |
| `launch` | string | No | `auto` at hub start, `lazy` on first call, or `manual`; schema default: `manual`. |
| `namespace` | string | No | Spawn in `spore` (hub space), `hyphae` (user space), or `any` (unconstrained); schema default: `hyphae`. |
| `roles` | array | No | See [ROLES.md](ROLES.md). |
| `api` | array | No | Callable capabilities. |
| `topics` | array | No | Published topics. |
| `permissions` | array | No | Capability requests, each with a `name` and an array of `reasons`. See [SECURITY.md](SECURITY.md#permissions-api). |
| `notes` | array | No | Strings containing additional information. |

## Capabilities

`api` entries declare callable subjects; `topics` declare broadcasts. Names
must match the subject used on the wire. See [GRAMMAR.md](GRAMMAR.md#capability)
and [GRAMMAR.md](GRAMMAR.md#broadcast) for routing and topic ownership.

```yaml
...

api:
  - name: clock.get_time
    description: Returns the current time.
    risk: benign
    usage:
      - clock.get_time timezone=UTC
    inputs:
      - name: timezone
        type: string
        description: IANA timezone; defaults to UTC.
    outputs:
      - name: time
        type: string
        description: Current time in ISO 8601 format.

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

...
```

| Field | API | Topic | Notes |
|---|---|---|---|
| `name` | Required | Required | Subject name; `:` is reserved. |
| `description` | Required | Required | Capability purpose. |
| `usage` | Required | Required | Array of example calls or `publish` messages. |
| `risk` | Required | Required | Risk level; see [SECURITY.md](SECURITY.md#risk-levels). |
| `inputs` | Optional | — | Arguments accepted by a call. |
| `outputs` | Optional | Optional | Success data returned by a call or fields published on a topic. |
| `notes` | Optional | Optional | Array of additional guidance. |

Inputs and outputs are arrays of entries with required `name`, `type`, and
`description` fields and optional `notes`. Inputs may also specify `required`,
which defaults to `false`. Type names are advisory; the hub does not validate
or coerce values. A `flag` is a bare token and should not be required;
`boolean` takes an explicit value. See [GRAMMAR.md](GRAMMAR.md#arguments-and-flags).
