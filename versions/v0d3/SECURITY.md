<!--
Copyright 2026 Matt Harrison
SPDX-License-Identifier: Apache-2.0
-->

# Spore OS Protocol — Security

## Trust and Risk

Nodes have **trust**; capabilities have **risk**. The hub uses both
to decide whether a capability can be accessed.

### Trust Levels

Trust is declared in the manifest and assigned at installation. The user can
override it, except for `system`.

| Level | Access |
|---|---|
| `system` | Hub-internal nodes on a hardcoded whitelist only. |
| `developer` | Bypasses binary checksum verification, not capability policy. For local development only. |
| `trusted` | Policy-controlled; announced as `standard` in manifest and upgraded to `trusted` by user. |
| `standard` | Policy-controlled; announced as `standard` in manifest, requires successful signature verification. |
| `untrusted` | Policy-controlled; announced as `standard` in manifest, downgraded on failure to verify signature. |

### Risk Levels

| Level | Default access |
|---|---|
| `benign` | Unimplemented; safe to use in any reasonable circumstance. |
| `standard` | Unimplemented; policy-controlled; generally safe to use. |
| `personal` | Unimplemented; personal data. |
| `secret` | Unimplemented; credentials or other sensitive material. |
| `admin` | Control-plane actions; fresh OS authentication for each request. |
| `protected` | System-critical capabilities; hardcoded hub whitelist only. |

## Installation and Handshake

Installation requires user confirmation and checksums the manifest. It also
checksums the binary, except for `developer` nodes. During the handshake, a
checksum mismatch denies the connection; changes to a checksummed file require
reinstallation.

For user-space verification, `OK` may be sent before the binary check
finishes; permissions are not granted until it succeeds. See
[HANDSHAKE.md](HANDSHAKE.md).

Signature verification and the keyring are planned; checksums alone do not
verify a publisher.

## Permissions API

Permissions apply to API subjects and topics. Requests are confirmed through
the shell. Manifest permission requests are prompted during installation;
approved grants are saved in `permissions.yaml`.

| Subject | Use |
|---|---|
| `SPORE.permission.request capability=filesystem.read` | Request a one-use permit for the caller. |
| `SPORE.permission.grant node=com.example.agent capability=filesystem.read` | Grant access. |
| `SPORE.permission.revoke node=com.example.agent capability=filesystem.read` | Revoke access. |

Requests use the original caller; a supplied `node=` is ignored. On approval,
the hub returns an opaque `permission.token` bound to that caller and
capability. It expires after 30 seconds and is consumed when it authorizes a
request, even if the operation later fails. Pass it as
`permission.token=<opaque-token>` on a capability call or topic subscription.
If access is already allowed, the token is not consumed. Permits cannot
authorize commands registered as `admin` or `protected`.

Topic risk handling not yet implemented.
