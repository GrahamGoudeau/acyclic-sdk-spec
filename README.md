# acyclic-sdk-spec

This is the shared contract for the Acyclic EHR agent SDKs ([JS](https://github.com/GrahamGoudeau/acyclic-sdk-js), [Go](https://github.com/GrahamGoudeau/acyclic-sdk-go), [Rust](https://github.com/GrahamGoudeau/acyclic-sdk-rs)) and the [sandbox](https://github.com/GrahamGoudeau/acyclic-sandbox). Every SDK ships a copy and must pass the same fixtures.

| File | What it defines |
|---|---|
| [tools.json](tools.json) | The 13 tools: names, descriptions, JSON Schema inputs, read or write |
| [projections.md](projections.md) | How each FHIR resource becomes a small flat result |
| [fixtures/projections/](fixtures/projections) | Input resources and expected results; the conformance test |
| [telemetry.md](telemetry.md) | Span names, the attribute allow-list, never-recorded data, audit record shape |
| [vendors.md](vendors.md) | How Epic, Oracle Health, athenahealth, MEDITECH and eClinicalWorks differ, with public sources |

## Changing the spec

1. Edit the file here, and add or adjust a fixture for any projection change.
2. Bump `version` in `tools.json`.
3. Sync each SDK (`npm run sync-spec` in JS, copy `spec/` in Go and Rust) and make its tests pass.

Apache-2.0. Copyright 2026 Acyclic LLC.
