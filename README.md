# Spore Protocol

A local IPC protocol for application-to-application communication over Unix domain sockets. The core idea: **treat applications as functions**.

A node exposes named subjects. The hub routes calls between them.

## Contents

| | |
|---|---|
| [versions/v1/SPEC.md](versions/v1/SPEC.md) | Protocol specification |
| [versions/v1/schema/](versions/v1/schema/) | Manifest schema |

**Current:** `SPORE/v1d0` — Pillar 1 (Request/Response) + Pillar 4 (Witness)

## Manifest Schema

Add to the top of any node manifest for validation and autocompletion in VS Code:

```yaml
# yaml-language-server: $schema=https://raw.githubusercontent.com/sporeos-dev/spore-os-protocol/main/versions/v1/schema/spore.manifest.v1d0.schema.yaml
```
