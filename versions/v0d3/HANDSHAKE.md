<!--
Copyright 2026 Matt Harrison
SPDX-License-Identifier: Apache-2.0
-->

# Spore OS Protocol — Handshake

A node must be installed before it connects. It opens the hub's Unix domain
socket and sends its node ID followed by a newline:

```text
com.example.clock
```

The hub looks up the installed node and responds:

```text
OK
```

If the node is unknown or verification fails, the hub rejects the connection
with `FAIL`, optionally followed by a space and a human-readable reason:

```text
FAIL
FAIL node not installed
```

The hub completes verification before sending `OK`. If verification requires
user-space access, the hub may ask an already-connected Hyphae agent; failure
to verify denies the handshake. See [SECURITY.md](SECURITY.md#installation-and-handshake).

After `OK` the node may send requests. Once its setup is done (immediately, if it has none) it sends `READY` on a line by itself, and the hub holds a lazy launch's call until then.

## Spawning

The hub can start an installed node:

```text
SPORE.node.spawn node=com.example.clock ~h1
```

The started process connects and performs the same handshake. The spawn
response is successful only after the hub announces that the node has
connected. If the handshake fails or the connection is not observed in time,
spawn returns an error. A timeout does not stop the process, which may still
connect later.
