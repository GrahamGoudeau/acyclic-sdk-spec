# Telemetry

The SDKs depend on no telemetry library. They call a `Telemetry` hook with
spans, logs and audit records whose names follow the OpenTelemetry semantic
conventions, so an adapter of about 20 lines connects them to any OTel SDK.

## Spans

| Span name | Kind | When |
|---|---|---|
| `invoke_agent acyclic` | internal | one agent run |
| `chat {model}` | client | one model call |
| `execute_tool {tool}` | internal | one tool call |
| `{METHOD} {ResourceType}` (e.g. `GET Observation`) | client | one FHIR HTTP request |
| `POST token` | client | one token request |

Attributes, all optional unless marked. **Only these keys may appear.**

| Key | Example | Notes |
|---|---|---|
| `gen_ai.operation.name` | `invoke_agent`, `chat`, `execute_tool` | required on GenAI spans |
| `gen_ai.provider.name` | `openai`, `anthropic`, `aws.bedrock` | |
| `gen_ai.request.model` | `deepseek/deepseek-v4.1-flash-1017` | |
| `gen_ai.usage.input_tokens` / `gen_ai.usage.output_tokens` | `812` | |
| `gen_ai.response.finish_reasons` | `["tool_calls"]` | |
| `gen_ai.tool.name` | `get_labs_vitals` | |
| `gen_ai.tool.call.id` | `call_3` | |
| `gen_ai.conversation.id` | run id | random, not derived from a patient |
| `http.request.method` | `GET` | |
| `http.response.status_code` | `200` | |
| `server.address` | `fhir.epic.com` | host only |
| `url.template` | `/api/FHIR/R4/Observation` | path with ids replaced by `{id}`; **never** a query string |
| `fhir.resource_type` | `Observation` | |
| `fhir.result_count` | `14` | entries returned |
| `acyclic.vendor` | `epic` | |
| `acyclic.patient_ref` | `p_5f1c…` | HMAC-SHA256(salt, patient id), hex, first 16 chars, prefixed `p_` |
| `acyclic.retry_count` | `1` | |
| `error.type` | `429`, `OperationOutcome:not-found`, `timeout` | |

## Never recorded

The patient-data rules are not optional and have no off switch. The SDKs
**must not** put any of these into spans, logs or errors they raise:

- query strings or search parameter values (names and birth dates are PHI)
- request or response bodies
- resource ids other than through `acyclic.patient_ref`
- prompts, model output and tool arguments or results
- tokens, client assertions and keys

`debug` mode adds tool argument **keys** (not values) and response sizes. It
does not lift the rules above. Use the SDK's `onToolResult` callback to see
contents in your own code.

## Logs

Logs are events with `level` (`debug`, `info`, `warn`, `error`), a `name`
(`fhir.request`, `token.refresh`, `agent.step`, `agent.stop`, `write.denied`,
`write.approved`) and attributes from the table above.

## Audit records

A separate stream for compliance retention, one record per FHIR request. The
shape follows FHIR AuditEvent loosely and flattened:

```json
{
  "time": "2026-11-23T15:04:05.123Z",
  "action": "R",
  "outcome": "0",
  "vendor": "epic",
  "resource_type": "Observation",
  "patient_ref": "p_5f1c0a9be2d4c311",
  "agent_run": "run_8f2a…",
  "tool": "get_labs_vitals",
  "purpose_of_use": "TREAT",
  "client_id": "your-client-id",
  "status": 200
}
```

- `action` is FHIR's code: `C` create, `R` read or search.
- `outcome` is `0` success, `4` minor failure (4xx), `8` serious failure (5xx or transport).
- `purpose_of_use` comes from config; the default is `TREAT`.
