# Vendor facts

What each EHR's FHIR R4 API does differently, as the SDKs and the sandbox mocks implement it. Every rule cites public documentation, gathered November 2026. **Unconfirmed** marks behavior that public docs do not state; there the mock follows the FHIR spec or the closest documented sibling.

No partner agreement, login-gated document or NDA material was used.

## Epic

Sources: [fhir.epic.com Documentation](https://fhir.epic.com/Documentation) (oauth2, fhir, searchparameters, troubleshooting), API specs at `fhir.epic.com/Specifications?api=<id>`, and the public sandbox's unauthenticated `/metadata` and `/.well-known/smart-configuration`.

| Topic | Rule |
|---|---|
| Base URL | `https://{host}/{instance}/api/FHIR/R4/`. Sandbox: `https://fhir.epic.com/interconnect-fhir-oauth/api/FHIR/R4/` |
| Token | `POST {instance}/oauth2/token`, `client_credentials` with a signed JWT. `iss` = `sub` = client id, `aud` = token URL. `jti` is at most 151 characters. `exp` is at most 5 minutes ahead. RS384 is preferred; ES384 only with a JWKS URL |
| Token response | `{"access_token","token_type":"bearer","expires_in":3600,"scope"}`. A bad assertion returns 400 `{"error":"invalid_client","error_description":null}` |
| Format | XML unless `Accept` or `_format` asks for JSON. Always send `Accept: application/fhir+json` |
| Patient search | Needs an identifier, or given + family + birthdate (organizations can change this minimum) |
| Observation search | `patient` plus **either** `category` **or** `code`, never both |
| DocumentReference search | `patient` plus `category=clinical-note` or `type`. Each note links one Binary per format: `text/html` and `text/rtf` |
| AllergyIntolerance | With no allergies, Epic returns a "no known allergies" resource rather than an empty Bundle |
| Appointment search | `patient`, with `service-category=appointment` |
| Paging | Follow `link[next]`, shaped `?sessionID=16-<hex>`. An expired session returns error 4113 |
| Errors | OperationOutcome issues carry Epic codes in `details.coding` (4101 no results, 4111 missing parameter, 4112 invalid combination, 4122 unknown parameter). An empty search returns a Bundle with a 4101 warning entry (`search.mode: outcome`). Unknown parameters are ignored with a warning |
| 401 | Empty body plus `WWW-Authenticate: Bearer error="invalid_token"` |
| DocumentReference.Create | Attachment `contentType` must be `text/plain`. `type` is one of LOINC 18842-5, 11488-4, 34117-2, 11506-3, 28570-0, 34111-5, 34746-8. `context.encounter` is required. Returns 201 with a **relative** `Location`; a body comes back only with `Prefer: return=representation` |
| Rate limits | Not documented publicly. The mock applies a light sandbox limit |

## Oracle Health (Millennium)

Sources: [Millennium R4 docs](https://docs.oracle.com/en/industries/health/millennium-platform-apis/mfrap/r4_overview.html), the OpenAPI file at `.../mfrap/swagger.json`, the [authorization framework](https://docs.oracle.com/en/industries/health/millennium-platform-apis/millennium-authorization-framework/), and [oracle-samples/ignite-endpoints](https://github.com/oracle-samples/ignite-endpoints).

| Topic | Rule |
|---|---|
| Base URL | `https://fhir-ehr.cerner.com/r4/{tenant}/`. Sandbox tenant `ec2458f2-1e24-41c8-b71b-0e701af7583d`. Tenant ids are not always UUIDs |
| Token | `https://authorization.cerner.com/tenants/{tenant}/protocols/oauth2/profiles/smart-v1/token`. System accounts use `client_secret_basic` or `private_key_jwt` (RS384/ES384) |
| Token response | `token_type: "Bearer"`, `expires_in: 570` |
| Scopes | v1 (`system/Patient.read`) or v2 (`system/Patient.rs`). **No wildcard scopes** |
| Accept | Required. Non-JSON media types return 406 |
| Patient search | `given` requires `family` |
| Observation | `code` is ignored when `category` is present. `_count` defaults to 50, max 200. Sorted newest first |
| DocumentReference | `_count` defaults to 10, max 100. Content at `Binary/<id>` with type prefixes (`TR-` HTML, `XR-` PDF) |
| Appointment search | `date` is required with `patient`, with a prefix **and a time** (`ge2026-01-01T00:00:00Z`). Sorted by start ascending |
| Paging | `link[next]` with `-pageContext=<opaque>&-pageDirection=NEXT`. `total` is absent on Observation, DocumentReference and Appointment searches |
| Ids | Patient numeric. Condition `p<digits>`. Observation `L-` (labs) or `VS-` (vitals). Coverage `<patient>-<x>-<x>` |
| DocumentReference.Create | `status: current`, `docStatus: final`, LOINC `type`, `subject`, exactly one attachment with `contentType` **including a charset** (`text/plain;charset=utf-8`), and `context.period.end` with a time. Returns 201 with an **empty body**, an absolute `Location` and `ETag: W/"1"` |
| Rate limits | Not documented for R4 |

## athenahealth (athenaOne)

Sources: [docs.athenahealth.com](https://docs.athenahealth.com/api) (environment-urls, base-fhir-urls, token-endpoint, authorization-overview, best-practices, testing-sandbox, fhir-r4/*, api-ref/appointment, api-error-messages), plus unauthenticated `metadata` and `smart-configuration` from the preview host.

| Topic | Rule |
|---|---|
| Base URLs | FHIR: `{host}/fhir/r4`. Proprietary: `{host}/v1/{practiceid}`. Preview host `api.preview.platform.athenahealth.com`; sandbox practice 195900 |
| Token | `POST {host}/oauth2/v1/token`. Use HTTP Basic auth for the secret (a secret in the body fails) or `private_key_jwt` |
| Token response | The documented 2-legged sample is `{"access_token","expires_in":"300"}`: `expires_in` is a **string** and there is no `token_type` |
| Scopes | SMART v1 or v2 system scopes, with no wildcards. `athena/service/Athenanet.MDP.*` covers the proprietary API. There is no `system/Appointment.read` |
| Practice | Patient searches with a system scope need `ah-practice=Organization/a-1.Practice-{id}` unless `_id` is given |
| MedicationRequest | Needs `patient` **and** `intent` |
| Appointments | No FHIR Appointment search for system apps. The SDK reads `GET /v1/{practiceid}/patients/{patientid}/appointments` (**unconfirmed path**, from the API reference index). Dates are `mm/dd/yyyy`, times `HH:MM` local. Status codes: f booked, x cancelled, o open, 2 checked in, 3 checked out, 4 charge entered |
| Ids | Patient `a-{practice}.E-{enterpriseid}`. Others are `a-{practice}.{type}-{n}` (`Problem`, `encounter`, `vitalamb`, `medicationrequest`) |
| 401 (FHIR) | `OperationOutcome` whose `details` is a plain string (non-conformant) |
| Errors (proprietary) | `{"error","detailedmessage"}` |
| Rate limits | Preview: 15 calls per second and 50,000 per day. Production: 150 per second and 500,000 per day. Daily limits come in `X-RateLimit-Limit` and `X-RateLimit-Remaining` |
| Writes | FHIR write scopes are off by default for system apps. Booking goes through `PUT /v1/{practiceid}/appointments/{id}` (form-encoded) |
| Paging | `_count` plus a `cursor` token. Next-link format **unconfirmed** |

## MEDITECH Expanse

Sources: the public Greenfield CapabilityStatement and SMART configuration (`greenfield-prod-apis.meditech.com/v2/uscore/STU6/...`), MEDITECH's public brands bundle, and public PDFs on home.meditech.com. The detailed API reference needs a Greenfield login, so this profile has the **lowest fidelity**.

| Topic | Rule |
|---|---|
| Base URL | Production: `https://{hospital-host}/v1/uscore/R4`, one host per hospital with no tenant in the path. Greenfield: `/v2/uscore/STU6` |
| Token | At the host root: `/oauth/token`. Supports `client_secret_basic`, `client_secret_post` and `private_key_jwt`. Signing algorithms **unconfirmed** |
| Scopes | System scopes are **SMART v2 only** (`system/Patient.rs`) |
| Paging | `continue` token. DocumentReference has no `_count`. AllergyIntolerance and Coverage have no paging parameters |
| MedicationRequest status | `active`, `cancelled`, `completed` |
| Writes | Only Communication and QuestionnaireResponse create are declared. The SDK offers no note writes |
| Required parameters, error bodies, ids, rate limits | **Unconfirmed** |

## eClinicalWorks

Sources: [fhir.eclinicalworks.com/ecwopendev](https://fhir.eclinicalworks.com/ecwopendev/documentation) (documentation, getting-started, per-resource pages under `apidocs/sections/scopes/`) and the unauthenticated discovery documents.

| Topic | Rule |
|---|---|
| Base URL | `https://fhir4.eclinicalworks.com/fhir/r4/{practiceCode}` (6 uppercase letters). Staging: `https://staging-fhir.ecwcloud.com/fhir/r4/FFBJCD` |
| Token | `https://oauthserver.eclinicalworks.com/oauth/oauth2/token`, `private_key_jwt` with a JWKS URL. The docs say RS384, but their own examples use ES384 and RS256 |
| Token response | `token_type: "Bearer"`, `expires_in: 300` |
| MedicationRequest | Patient searches include `intent=order` |
| Appointment | **Not offered** |
| Paging | `page` parameter. There is no `_count` |
| Rate limits | 500 calls per minute per practice base URL. Going over blocks every request for the rest of the minute (429) |
| CORS | "We do not support CORS headers for the FHIR APIs." Call eCW from a server. The mock allows CORS only so the playground can run in a browser |
| Writes | Transaction Bundles POSTed to the base URL, contract required. The SDK offers no note writes |
| Ids | Opaque, 43 to 64 characters, sharing a per-practice prefix |
