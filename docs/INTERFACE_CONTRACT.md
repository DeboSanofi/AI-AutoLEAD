# AI AutoLEAD – ChemDiary Middleware Interface Contract (Draft)

Draft of 24 Nov 2026.

This draft proposes the API that GBG calls to create a ChemDiary experiment and fill its worksheets, with one call per workflow step. It is a starting point for the Tech Lead and BioZero to review; every field name and value is illustrative until the data model is confirmed.

## 1. Purpose and design principles

- **One call per step:** GBG sends one JSON document per worksheet; the middleware makes the several ChemDiary calls behind it.
- **No workflow logic in the middleware (Phase 1):** GBG decides what to send and when.
- **Idempotent writes:** A repeated call with the same key never duplicates data.
- **Clear failure reporting:** Because one call becomes many, the response says exactly what was written and what failed.
- **Scope:** ChemDiary integration only.

## 2. Proposed endpoints

One create endpoint plus one per template block that GBG fills.

| Endpoint (proposed) | Template block | Phase | Returns |
| --- | --- | --- | --- |
| POST /experiments | Create experiment from the AutoLEAD template | 1 | Experiment reference ID |
| PUT /experiments/{id}/reaction-scheme | Reaction Scheme: structure, Reactants rows, Products rows | 1 | Per-field result |
| PUT /experiments/{id}/procedure | Procedure | 1 | Per-field result |
| PUT /experiments/{id}/placeholder-procedure | Placeholder worksheet procedure | 1, to confirm | Per-field result |
| PUT /experiments/{id}/analytic | Analytic: found/not found, chromatogram file | 1 | Per-field result |
| PUT /experiments/{id}/samples | Samples Table rows | 1, unless auto-filled | Per-field result |
| PUT /experiments/{id}/containers | Container table rows | 1, unless auto-filled | Per-field result |
| PUT /experiments/{id}/conclusion | Conclusion | 1, to confirm | Per-field result |
| GET /experiments/{id} | Read status and written data | 1 | Experiment state |
| POST /experiments/{id}/analytical-requests | Add analytical tasks to samples; refresh results | Project Phase 2 | Request ID |

If ChemDiary fills the Samples Table and Container table automatically from the structure, those two endpoints drop out. The endpoint list follows the template snapshots and must be checked against the field mapping matrix.

## 3. Common elements

| Element | Proposal | Status |
| --- | --- | --- |
| Authentication | Service credential for GBG; method to be agreed with BioZero and security | Open |
| Identity in ChemDiary | A service account writes the data; confirm how the audit trail shows it | Open |
| Idempotency | Client-supplied key per call; repeats return the original result | Proposed |
| Experiment reference | GBG sends its own run ID on create; the middleware returns the ChemDiary ID | Proposed |
| Validation | Mandatory fields checked before any ChemDiary write; whole call rejected if missing | Proposed |
| Partial failure | Response lists each field as written or failed, so GBG can retry only the failures | Proposed |
| Errors | Standard HTTP status plus a machine-readable code and field path | Proposed |
| Return of the ID | Synchronous reply, or webhook plus callback | Open, see section 5 |
| Versioning | Version in the path, tied to the template version | Proposed |

## 4. Example payloads

Illustrative only. Field names and values are placeholders until the data model is confirmed.

**Create an experiment:** POST /experiments

```json
{
  "idempotencyKey": "gbg-run-0001-create",
  "template": "AutoLEAD",
  "gbgRunId": "gbg-run-0001",
  "labHeadRef": "<to be confirmed>"
}
```

```json
{
  "experimentId": "<ChemDiary reference ID>",
  "status": "created"
}
```

**Fill a worksheet:** PUT /experiments/{id}/procedure

```json
{
  "idempotencyKey": "gbg-run-0001-procedure",
  "content": "<procedure text, rich text or plain text, to confirm>"
}
```

**Fill the reaction scheme:** PUT /experiments/{id}/reaction-scheme

```json
{
  "idempotencyKey": "gbg-run-0001-reaction-scheme",
  "reactants": [
    { "Reactant": "<name>", "Role": "<dropdown value>", "Material ID": "<building block ID>", "Eq": 1.0 }
  ],
  "products": [
    { "Product": "<name>", "Product ID": "<ID>" }
  ]
}
```

**Fill the analytic block:** PUT /experiments/{id}/analytic

```json
{
  "idempotencyKey": "gbg-run-0001-analytic",
  "productFound": true,
  "chromatogram": "<file reference or upload, to confirm>"
}
```

**Response with a partial failure:**

```json
{
  "status": "partial",
  "written": ["reactants[0].Reactant"],
  "failed": [
    { "field": "reactants[0].Role", "code": "VALIDATION_ERROR", "message": "<reason>" }
  ]
}
```

## 5. Open decisions

| # | Decision | Options | Owner |
| --- | --- | --- | --- |
| 1 | How GBG receives the experiment ID | Synchronous reply to the create call; or webhook, with the middleware calling back later | Tech Lead, BioZero |
| 2 | Granularity of endpoints | One per template block; or one per workflow step | Tech Lead, Jan H. |
| 3 | Behaviour on missing mandatory fields | Reject the whole call; or write what is valid and report the rest | Tech Lead, business |
| 4 | Authentication method | To agree with BioZero and security | Tech Lead |
| 5 | Which worksheets GBG fills | Depends on the field mapping matrix | Jan H. |
| 6 | How to handle a half-filled experiment after a failure | Retry from GBG; or reconciliation job | Tech Lead |
