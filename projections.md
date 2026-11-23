# Projections

Tools return small, flat JSON objects, not FHIR resources. This keeps model
context small and returns only the fields a task needs (HIPAA minimum
necessary). Every SDK must produce byte-for-byte the same projection for the
same FHIR input; `fixtures/projections/*.json` is the test.

## Shared rules

- **Absent means omitted.** A field with no source value is left out of the object. Never `null`, never `""`.
- **display(cc)** for a CodeableConcept: `cc.text`, else the first `coding[].display`, else the first `coding[].code`.
- **codes(cc)**: every coding as `"SYSTEM|code"`, in source order. SYSTEM is the short name below, else the system URL unchanged. Omitted when empty.

| System URL | Short |
|---|---|
| `http://snomed.info/sct` | `SNOMED` |
| `http://loinc.org` | `LOINC` |
| `http://hl7.org/fhir/sid/icd-10-cm` | `ICD-10-CM` |
| `http://www.nlm.nih.gov/research/umls/rxnorm` | `RXNORM` |
| `http://hl7.org/fhir/sid/cvx` | `CVX` |

- **status(cc)** for `clinicalStatus` / `verificationStatus`: the first `coding[].code`.
- **ref(r)**: `r.display`, else `r.reference`.
- **Dates and times** are copied as given. No timezone conversion.
- **Quantity text**: `value` then a space then `unit` (or `code` when `unit` is absent). Numbers print as the shortest JSON form (`7.2`, not `7.20`).

## Objects

**patient**: Patient
- `id`
- `name`: the `official` name (else the first name). Uses `text` if present, else the given names and the family name joined by spaces.
- `birth_date`: `birthDate`
- `gender`
- `mrn`: the first `identifier` whose `type.coding[].code` is `MR`
- `deceased`: `true` only when `deceasedBoolean` is true or `deceasedDateTime` is present

**condition**: Condition
- `id`
- `display`: display(code)
- `codes`: codes(code)
- `clinical_status`: status(clinicalStatus)
- `onset`: `onsetDateTime`, else `onsetPeriod.start`
- `recorded`: `recordedDate`

**medication**: MedicationRequest
- `id`
- `display`: display(`medicationCodeableConcept`), else `medicationReference.display`
- `codes`: codes(`medicationCodeableConcept`)
- `status`
- `dosage`: `dosageInstruction[0].text`
- `authored_on`: `authoredOn`
- `requester`: ref(requester)

**allergy**: AllergyIntolerance
- `id`
- `display`: display(code)
- `clinical_status`: status(clinicalStatus)
- `criticality`
- `reactions`: display of every `reaction[].manifestation[]`, in order, deduplicated
- `recorded`: `recordedDate`

**observation**: Observation
- `id`
- `display`: display(code)
- `loinc`: the first `http://loinc.org` coding's `code`
- `value`:
  - Quantity text for `valueQuantity`
  - display for `valueCodeableConcept`
  - `valueString` as is
  - with no `value[x]` and every `component` carrying a valueQuantity: the component values joined by `/`, a space, then the first component's unit (blood pressure: `"128/82 mmHg"`)
- `value_number`: `valueQuantity.value` when present
- `unit`: `valueQuantity.unit` (else `.code`) when present
- `interpretation`: display(`interpretation[0]`)
- `effective`: `effectiveDateTime`, else `effectivePeriod.start`, else `issued`
- `category`: the first `category[].coding[].code`

**encounter**: Encounter
- `id`
- `status`
- `class`: `class.code` (R4 Coding)
- `type`: display(`type[0]`)
- `start`: `period.start`
- `end`: `period.end`
- `reason`: display(`reasonCode[0]`)
- `location`: ref(`location[0].location`)
- `provider`: ref of the first `participant[].individual`

**appointment**: Appointment
- `id`
- `status`
- `start`, `end`
- `type`: display(`appointmentType`), else display(`serviceType[0]`)
- `description`
- `practitioner`: ref of the first `participant[].actor` whose `reference` starts with `Practitioner/`
- `location`: ref of the first `participant[].actor` whose `reference` starts with `Location/`

**document**: DocumentReference
- `id`
- `title`: `description`, else `content[0].attachment.title`, else display(type)
- `type`: display(type)
- `date`
- `author`: ref(`author[0]`)
- `content_type`: `content[0].attachment.contentType`
- `status`

**document_text**: DocumentReference plus its content
- `id`, `title` (as in document)
- `text`: the attachment's plain text. Inline `data` is base64-decoded; otherwise the SDK fetches the `url` (a Binary, `Accept: text/plain` or `application/fhir+json` with `data`). HTML is reduced to text by replacing each tag with a space, decoding `&amp; &lt; &gt; &quot; &#39; &nbsp;`, then collapsing runs of whitespace to one space and trimming. Plain text is only trimmed.
- `truncated`: `true` when the text was cut to `max_chars` (cut at a character boundary, never mid-codepoint)

**coverage**: Coverage
- `id`
- `status`
- `payer`: ref(`payor[0]`)
- `plan`: the `value` of the `class[]` entry whose `type.coding[].code` is `plan`, else its `name`
- `member_id`: `subscriberId`, else the first `identifier[].value`
- `relationship`: display(relationship)
- `period_start`, `period_end`

## Lists

A list tool returns `{ "items": [...], "total": n, "truncated": bool }`:
- `total` is the number of items returned.
- `truncated` is `true` when the SDK stopped paging at `limit` while more remained.
- Order follows the server's order, except observations, encounters and documents, which are sorted newest first by `effective` / `start` / `date`. A stable sort keeps the server order for ties; missing dates sort last.
