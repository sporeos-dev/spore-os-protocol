<!--
Copyright 2026 Matt Harrison
SPDX-License-Identifier: Apache-2.0
-->

# Spore OS Protocol — Mutate

A mutator changes a message before delivery. Like a [guard](GUARD.md),
it can target requests, replies, roles, and other hooks.

## Example: Normalize a temperature input

```text
Caller -> Hub:       thermostat.set temperature=68 unit=F ~h1
Hub -> Mutator:      temperature.normalize temperature=68 unit=F ~m1
Mutator -> Hub:      ~m1:temperature.normalize temperature=20 unit=C ok
Hub -> Thermostat:   thermostat.set temperature=20 unit=C cast=com.example.panel ~h1
Thermostat -> Hub:   ~h1:thermostat.set ok
Hub -> Caller:       ~h1:thermostat.set ok capture=com.example.thermostat
```

The hook call is illustrative. Mutators can use ordinary request/response;
attachment and ordering rules are open.
