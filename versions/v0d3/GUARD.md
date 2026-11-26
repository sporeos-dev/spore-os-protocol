<!--
Copyright 2026 Matt Harrison
SPDX-License-Identifier: Apache-2.0
-->

# Spore OS Protocol — Guard

A guard allows or denies delivery on a route. It adds checks to
[permissions](SECURITY.md#permissions-api) for Spore traffic, not direct
OS access. Guards can cover requests, replies, roles, and other hooks.

## Example: Deny a file read

```text
Caller -> Hub:   filesystem.read path="/outside/report.txt" ~h1
Hub -> Guard:    policy.check_read path="/outside/report.txt" ~g1
Guard -> Hub:    ~g1:policy.check_read allowed=false ok
Hub -> Caller:   ~h1:filesystem.read error code=NotPermitted.Policy what="path outside allowed directory" capture=SPORE.hub
```

The hook call is illustrative. Guards can use ordinary request/response;
attachment rules are open. Blocking a reply does not undo the operation.
