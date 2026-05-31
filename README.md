
> For project overview, contributing guidelines, code of conduct, security policy,
> and licensing information, see the
> [sporeos-dev organization README](https://github.com/sporeos-dev/.github).

# Spore OS Protocol

A local IPC protocol for application-to-application communication over Unix domain sockets. The core idea: **applications as functions**.

Nodes expose capabilities. The hub routes calls between them.

## Versions

| Versions | Focus | Details |
| --- | --- | --- |
| v1 | Request / response | A node sends a request. Another node replies. |
|  | Core funtionality | Hub, nodes, manifests, communication, etc. |
|  | Witness | A node that listens for logging and auditability. |
|  | Core nodes | Command line, shell REPL, witness, logger, dialogs. |
|  | Go | Client library reference written in Go. |

## Manifest Schema

Add to the top of any node manifest for validation and autocompletion in VS Code:

```yaml
# yaml-language-server: $schema=https://raw.githubusercontent.com/sporeos-dev/spore-os-protocol/main/versions/v1/schema/spore.manifest.v1d0.schema.yaml
```
