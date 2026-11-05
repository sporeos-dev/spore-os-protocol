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

If the node is unknown or verification fails, the hub rejects the connection.
For user-space verification, `OK` can arrive before the binary check finishes;
the node receives permissions only after verification succeeds. See
[SECURITY.md](SECURITY.md#installation-and-handshake).

## Spawning

The hub can start an installed node:

```text
SPORE.node.spawn node=com.example.clock ~h1
```

The started process connects and performs the same handshake. A successful
spawn means the process was started; it does not guarantee that the handshake
has completed.
