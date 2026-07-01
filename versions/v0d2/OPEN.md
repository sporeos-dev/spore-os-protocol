<!--
Copyright 2026 Matt Harrison
SPDX-License-Identifier: Apache-2.0
-->

# Spore OS Protocol — Open Issues
**Version:** v0d2  
**Date:** 2026-06-30

These are areas of the protocol that are known to be incomplete or undefined in v0d2. They are not roadmap items — they are gaps in the current specification that a protocol reader should be aware of. Each item represents a known incompleteness that requires resolution before this area of the protocol can be considered normative.

---

## 1. Trust/Risk Enforcement: Middle Tiers

**Affected:** [SPEC.md §11.4](SPEC.md)

The trust/risk model is established in v0d2 and manifest fields are active. However, the runtime enforcement behaviour for the middle risk tiers is not fully specified.

**What is defined:**
- `benign` is open to all callers
- `protected` is governed by a hardcoded hub whitelist, cannot be overridden
- `personal` and `secret` are denied by default and require explicit user grant

**What is not yet defined:**
- The full trust/risk policy matrix (which trust levels may access which risk levels by default)
- How a node requests elevated access at runtime
- How a user grants or revokes access
- How user preferences are stored, expressed, and loaded by the hub

---

## 2. Permission Model

**Affected:** [HUB.md](HUB.md) — subjects not yet in hub contract

`spored` exposes a `SPORE.permission.*` API for querying, requesting, granting, and revoking permissions at runtime. These subjects are not yet part of the normative hub contract. Resolution depends on first defining the enforcement model (see item 1).

**Subjects pending specification:**

| Subject | Purpose |
|---|---|
| `SPORE.permission.list` | List current permission grants |
| `SPORE.permission.request` | Node requests access to a capability |
| `SPORE.permission.grant` | Grant a node access to a capability |
| `SPORE.permission.revoke` | Revoke a node's access to a capability |

---

## 3. Hub Contract Completeness

**Affected:** [HUB.md](HUB.md)

`spored` exposes subjects beyond those listed in [HUB.md](HUB.md). The following are implemented but not yet adjudicated as part of the normative hub contract:

| Subject | Notes |
|---|---|
| `SPORE.node.spawn` | Start a node process |
| `SPORE.node.kill` | Stop a node process |
| `SPORE.node.help` | Describe a node |
| `SPORE.command.list` | List all known subjects |
| `SPORE.command.help` | Describe a subject |
| `SPORE.error.list` | List all known error codes |
| `SPORE.error.help` | Describe an error code |
| `SPORE.topic.list` | List all known topics |
| `SPORE.topic.help` | Describe a topic |
| `SPORE.permission.*` | See item 2 |
| `SPORE.security.keyring.*` | Hub keyring management — likely hub-internal only |
| `SPORE.security.signature.*` | Signature verification — likely hub-internal only |
| `SPORE.developer.manifest.load` | Development-only manifest hot-reload |

The `SPORE.security.*` and `SPORE.developer.*` subjects are likely hub-internal administration API and may not belong in the normative contract. This needs to be decided explicitly.

---

## 4. Manifest Signing and Verification

**Affected:** [SPEC.md §8](SPEC.md) — no signing fields defined

`spored` supports verifying node manifests and binaries against a cryptographic signature using a keyring of trusted public keys. The manifest format does not currently define any fields for declaring a signature or checksum.

**What needs to be decided:**
- Whether the manifest schema should include a `signature` or `checksum` field
- Whether companion `.sig` files are a protocol-level concept or purely a hub implementation detail
- A normative statement on hub verification behaviour at install time (e.g. whether a hub may reject an unverified manifest)

---

## 5. User Preferences

**Affected:** [SPEC.md §11](SPEC.md)

The trust/risk model references "user preferences" and "user grants" without defining them. What needs to be specified:

- Whether user preferences are a protocol-level construct (e.g. a defined file format or API) or purely a hub implementation concern
- How preferences interact with the trust/risk enforcement matrix
- Whether preferences are editable at runtime or only at configuration time

This is closely coupled with items 1 and 2.
