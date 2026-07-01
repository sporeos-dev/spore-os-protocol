<!--
Copyright 2026 Matt Harrison
SPDX-License-Identifier: Apache-2.0
-->

# Spore OS Hub — Normative Contract
**Version:** v0d2  
**Protocol spec:** [SPEC.md](SPEC.md)  
**Date:** 2026-06-30  
**Status:** Development

## Overview

The normative `SPORE.*` subjects and topics a conforming hub must expose. The hub is a Spore OS node (`dev.sporeos.SPORE`, `system` trust) whose manifest follows the standard format defined in [SPEC.md §8](SPEC.md). Implementations may expose additional subjects beyond those listed here. For protocol fundamentals, see [SPEC.md](SPEC.md).

> **Known gaps:** Several subjects in `spored` are not yet adjudicated as part of this contract. See [OPEN.md](OPEN.md).

---

## Hub Manifest

```yaml
id: dev.sporeos.SPORE
name: Spore OS Hub
app: spore
schema: SPORE/v0d2
trust: system
description: Central router and manifest registry.

api:
  - name: SPORE.node.list
    description: List all known nodes and their state (installed or connected).
    usage:
      - SPORE.node.list
    outputs:
      - name: nodes
        type: array
        description: Array of installed node IDs.

  - name: SPORE.node.install
    description: Register a node's manifest with the hub. Makes the node discoverable and callable.
    usage:
      - SPORE.node.install path=/path/to/manifest.yaml
      - SPORE.node.install path=(dialog.file_picker)
    inputs:
      - name: path
        type: string
        required: true
        description: Path to the manifest YAML file. Supports inline call substitution (e.g. path=(dialog.file_picker)).

  - name: SPORE.node.uninstall
    description: Unregister a node from the hub. Removes it from the manifest registry.
    usage:
      - SPORE.node.uninstall node=com.example.old_node
      - SPORE.node.uninstall path=/path/to/manifest.yaml
    inputs:
      - name: node
        type: string
        required: false
        description: Reverse-domain ID of the node to uninstall.
      - name: path
        type: string
        required: false
        description: Path to the manifest YAML file.
    notes:
      - "Either node or path must be provided (at least one is required)."

  - name: SPORE.topic.subscribe
    description: Subscribe the calling node to a topic. Delivered messages arrive prefixed with `publish` while the node is connected.
    usage:
      - SPORE.topic.subscribe topic=SPORE.node.spawned ~h1
    inputs:
      - name: topic
        type: string
        required: true
        description: The topic subject to subscribe to. Must be declared in an installed node's manifest.

  - name: SPORE.topic.unsubscribe
    description: Unsubscribe the calling node from a topic.
    usage:
      - SPORE.topic.unsubscribe topic=SPORE.node.spawned ~h1
    inputs:
      - name: topic
        type: string
        required: true
        description: The topic subject to unsubscribe from.

topics:
  - name: SPORE.node.installed
    description: Published when a node's manifest is registered with the hub.
    risk: standard
    usage:
      - publish SPORE.node.installed node=com.example.clock
    outputs:
      - name: node
        type: string
        description: The node ID of the installed node.

  - name: SPORE.node.uninstalled
    description: Published when a node is unregistered from the hub.
    risk: standard
    usage:
      - publish SPORE.node.uninstalled node=com.example.clock
    outputs:
      - name: node
        type: string
        description: The node ID of the uninstalled node.

  - name: SPORE.node.spawned
    description: Published when the hub spawns a node process.
    risk: standard
    usage:
      - publish SPORE.node.spawned node=com.example.clock
    outputs:
      - name: node
        type: string
        description: The node ID of the spawned node.

  - name: SPORE.node.killed
    description: Published when the hub kills a node process.
    risk: standard
    usage:
      - publish SPORE.node.killed node=com.example.clock
    outputs:
      - name: node
        type: string
        description: The node ID of the killed node.
```

---

## Core Witness Nodes

These nodes ship with the hub and declare `witness: true` in their manifests.

| Node | Purpose | Autostart |
|---|---|---|
| `SPORE.witness.console` | Active debugging and REPL companion. Outputs to stdout with filtering flags. | No |
| `SPORE.witness.log` | Persistent record. Writes a rolling log file, unfiltered. | Yes |

---

*Spore OS Hub Contract v0d2 — June 2026*
