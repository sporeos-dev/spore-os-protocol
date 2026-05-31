
> For project overview, contributing guidelines, code of conduct, security policy,
> and licensing information, see the
> [sporeos-dev organization README](https://github.com/sporeos-dev/.github).

# Spore OS Protocol

A local IPC protocol for application-to-application communication over Unix domain sockets. The core idea: **applications as functions**.

Nodes expose capabilities. The hub routes calls between them.

## Versions

| Version | Status | Description |
| --- | --- | --- |
| v1 | Stable | Request/response, hub, manifests, witness, core nodes, Go client library. |
| v2 | Coming soon | — |

## Manifest Schema

Add to the top of any node manifest for validation and autocompletion in VS Code:

```yaml
# yaml-language-server: $schema=https://raw.githubusercontent.com/sporeos-dev/spore-os-protocol/main/versions/v1/spore.manifest.v1.schema.yaml
```
