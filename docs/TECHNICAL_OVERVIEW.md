# AI AutoLEAD – ChemDiary REST API: Technical Overview (Draft)

Draft of 26 Nov 2026.

This overview describes a simple REST API that GBG calls to create a ChemDiary experiment from the AutoLEAD template and fill its blocks, so the technical team can review the structure before the spikes. Field names come from the [Field Mapping Matrix](FIELD_MAPPING_MATRIX.md) and are not yet confirmed against the ChemDiary data model; mandatory flags and allowed values are open.

## 1. Purpose, scope and design principles

- **Scope:** ChemDiary integration only. One caller (GBG), one target (ChemDiary).
- **Runtime:** Python FastAPI in its own container (`autolead-chemdiary-api`), deployed on SCALe, built from the shared AutoLEAD middleware template.
- **Simple:** A few endpoints, plain JSON over HTTPS, no workflow logic in the API. GBG decides what to send and when.
- **Authentication:** GBG is the only consumer, so a simple API key is enough.
- **One call per template block:** The API makes the several ChemDiary calls behind it.
- **PUT per block:** Re-sending a block replaces it, so retries are safe.
- **Validate first, write second:** A rejected call writes nothing.
- **Idempotent:** Every write carries a key; a repeat returns the stored result.
- **Per-field results:** If a write fails partway, GBG sees which fields were written and which failed.

## 2. Endpoint structure and headers

Version in the path (`/v1`), bumped when the template changes.

| Method and path | Template block | Phase | Notes |
| --- | --- | --- | --- |
| POST /v1/experiments | Create from the AutoLEAD template | 1 | Returns the experiment ID |
| PUT /v1/experiments/{id}/reaction-scheme | Reaction Scheme (Reactants and Products) | 1 | |
| PUT /v1/experiments/{id}/procedure | Procedure | 1 | Rich text |
| PUT /v1/experiments/{id}/placeholder-procedure | Placeholder worksheet procedure | 1, to confirm | Only if used |
| PUT /v1/experiments/{id}/analytic | Analytic | 1 | Found/not found, chromatogram |
| PUT /v1/experiments/{id}/samples | Samples Table | 1, to confirm | Drops out if auto-filled |
| PUT /v1/experiments/{id}/containers | Container table | 1, to confirm | Drops out if auto-filled |
| PUT /v1/experiments/{id}/conclusion | Conclusion | 1, to confirm | May stay human-entered |
| GET /v1/experiments/{id} | Status and written data | 1 | For debugging |
| GET /health | Liveness | 1 | For the container platform |

Headers on every call:

| Header | Purpose |
| --- | --- |
| X-API-Key | API key for GBG, stored as a SCALe secret |
| Idempotency-Key | One per write, for example gbg-run-0001-procedure |
| X-Correlation-Id | Same ID in every log line for the run |
| Content-Type: application/json | Plain JSON |

## 3. Field reference per endpoint

Fields are the column names in the AutoLEAD template, as listed in the Field Mapping Matrix. "Mandatory" and types are to be confirmed from the data model. Fields marked calculated may be derived by ChemDiary from the structure, in which case GBG would not send them.

**POST /v1/experiments**

| Field | Mandatory | Notes |
| --- | --- | --- |
| gbgRunId | Yes | GBG's own run ID, used for correlation |
| template | Yes | "AutoLEAD" |
| labHeadRef | To confirm | Lab head reference agreed with the patent department |

**PUT /v1/experiments/{id}/reaction-scheme: reactants[]**

| Field | Mandatory | Notes |
| --- | --- | --- |
| Rxn ID | To confirm | Key shared with products and samples |
| Reactant, Role, Limit? | To confirm | Role is likely a dropdown |
| Barcode, Location, Reference | To confirm | |
| Material ID | To confirm | GBG knows the building block ID |
| MF, FM, EM, MW | To confirm | Calculated? |
| Eq | To confirm | |
| Sample Mass, Moles, Molarity, Vol, d | To confirm | Amounts used in the run |
| CAS Number, MFCD, IUPAC Name | To confirm | May need another source than GBG |
| Structural Comment, % Wt, Comment | To confirm | |

**PUT /v1/experiments/{id}/reaction-scheme: products[]**

| Field | Mandatory | Notes |
| --- | --- | --- |
| Rxn ID, Product ID, Product | To confirm | |
| MF, FM, EM | To confirm | Calculated? |
| Actual Mass, Actual Mol | To confirm | |
| Theo Mass, Theo Mol, Yield | To confirm | Calculated? |
| Purity | To confirm | Analytics result; returned in project Phase 2 |
| Structural Comment, IUPAC Name, CAS | To confirm | |
| Biomolecule 1 ID, Biomolecule 2 ID | To confirm | |

**PUT /v1/experiments/{id}/procedure** and **/placeholder-procedure**

| Field | Mandatory | Notes |
| --- | --- | --- |
| content | To confirm | Rich text; or fixed template text with variables? |

**PUT /v1/experiments/{id}/analytic**

| Field | Mandatory | Notes |
| --- | --- | --- |
| productFound | To confirm | Fills "[add found/not found]" |
| chromatogram | To confirm | File reference or upload; depends on spike 1 |

**PUT /v1/experiments/{id}/samples: rows[]** (drops out if auto-filled)

| Field | Mandatory | Notes |
| --- | --- | --- |
| ID, Rxn ID | To confirm | |
| Chemical Name, Structural comment | To confirm | |
| Corporate ID, Corporate ID status | To confirm | Linked to compound registration |
| Amount, Purity, Batch Comment | To confirm | |
| MF, FM, Status, Material ID | To confirm | |
| Site, Project Code, Origin Code, Department | To confirm | Possibly constants per lab or project |

**PUT /v1/experiments/{id}/containers: rows[]** (drops out if auto-filled)

| Field | Mandatory | Notes |
| --- | --- | --- |
| Sample ID (link) | To confirm | Links to the Samples Table |
| Container Type, Container Barcode, Container ID | To confirm | |
| Concentration, Amount, Solvent/Buffer | To confirm | |
| Project Code, IDG 1, IDG2, IDG3, IDG4 | To confirm | Meaning of the IDG fields to confirm |

**PUT /v1/experiments/{id}/conclusion**

| Field | Mandatory | Notes |
| --- | --- | --- |
| content | To confirm | May stay human-entered |

## 4. Examples and error model

Illustrative only; values are placeholders.

**Create:** POST /v1/experiments

```json
{ "gbgRunId": "gbg-run-0001", "template": "AutoLEAD", "labHeadRef": "<tbc>" }
```

```json
{ "experimentId": "<ChemDiary ID>", "status": "created" }
```

**Fill the reaction scheme:** PUT /v1/experiments/{id}/reaction-scheme

```json
{
  "reactants": [
    { "Reactant": "<name>", "Role": "<value>", "Material ID": "<id>", "Eq": 1.0 }
  ],
  "products": [ { "Product": "<name>", "Product ID": "<id>" } ]
}
```

```json
{ "status": "complete", "written": ["reactants[0]", "products[0]"], "failed": [] }
```

**Rejected call (nothing written):**

```json
{
  "status": "rejected",
  "errors": [
    { "field": "reactants[0].Role", "code": "MISSING_MANDATORY", "message": "Role is required" }
  ]
}
```

**Partial result (a write failed midway):**

```json
{
  "status": "partial",
  "written": ["reactants[0]"],
  "failed": [ { "field": "products[0]", "code": "ELN_ERROR", "retryable": true } ]
}
```

| Status | When |
| --- | --- |
| 200 / 201 | Written |
| 400 / 422 | Bad JSON, or a missing or invalid field; nothing was written |
| 401 / 403 | Missing or wrong API key |
| 404 | Unknown experiment ID |
| 409 | Same idempotency key with a different payload |
| 502 / 503 | ChemDiary is unreachable; GBG may retry with the same key |

**Complete reaction-scheme payload with every template column**

This shows one reactant and one product with all columns from the template. Values are placeholders. Columns marked calculated in section 3 may be left out if ChemDiary derives them.

```json
{
  "reactants": [
    {
      "Rxn ID": "<rxn id>",
      "Reactant": "<name>",
      "Role": "<dropdown value>",
      "Limit?": true,
      "Barcode": "<barcode>",
      "Location": "<location>",
      "Reference": "<reference>",
      "Material ID": "<building block id>",
      "MF": "<formula>",
      "FM": "<formula mass>",
      "EM": "<exact mass>",
      "MW": "<molecular weight>",
      "Eq": 1.0,
      "Sample Mass": { "value": 0.0, "unit": "<unit>" },
      "Moles": { "value": 0.0, "unit": "<unit>" },
      "Molarity": { "value": 0.0, "unit": "<unit>" },
      "Vol": { "value": 0.0, "unit": "<unit>" },
      "d": 0.0,
      "CAS Number": "<cas>",
      "MFCD": "<mfcd>",
      "Structural Comment": "<text>",
      "IUPAC Name": "<name>",
      "% Wt": 0.0,
      "Comment": "<text>"
    }
  ],
  "products": [
    {
      "Rxn ID": "<rxn id>",
      "Product ID": "<id>",
      "Product": "<name>",
      "MF": "<formula>",
      "FM": "<formula mass>",
      "EM": "<exact mass>",
      "Actual Mass": { "value": 0.0, "unit": "<unit>" },
      "Purity": 0.0,
      "Yield": 0.0,
      "Structural Comment": "<text>",
      "Theo Mass": { "value": 0.0, "unit": "<unit>" },
      "Theo Mol": { "value": 0.0, "unit": "<unit>" },
      "Actual Mol": { "value": 0.0, "unit": "<unit>" },
      "IUPAC Name": "<name>",
      "CAS": "<cas>",
      "Biomolecule 1 ID": "<id>",
      "Biomolecule 2 ID": "<id>"
    }
  ]
}
```

**Fill the analytic block:** PUT /v1/experiments/{id}/analytic

```json
{ "productFound": true, "chromatogram": "<file reference or upload, to confirm>" }
```

**Fill the procedure:** PUT /v1/experiments/{id}/procedure

```json
{ "content": "<procedure text, rich text or plain text, to confirm>" }
```

## 5. Best practices and spikes

**REST best practices applied**

1. **Resource-oriented URLs:** plural nouns, no verbs, version in the path (`/v1`).
2. **Right HTTP method:** POST creates, PUT replaces a block, GET reads. GET never changes anything.
3. **Meaningful status codes** and one error shape everywhere, with a machine-readable code, a field path and a `retryable` flag.
4. **Idempotency-Key on every write:** store the key, a payload hash and the result; a repeat returns the stored result, and the same key with a different payload returns 409.
5. **Strict validation with Pydantic models:** reject unknown fields, check mandatory fields and allowed values, and do it before any ChemDiary write. Return 422 with the field paths.
6. **OpenAPI is the contract:** FastAPI generates it from the models; review it with BioZero and add contract tests in CI.
7. **API key handling:** send it in the `X-API-Key` header over TLS only, read it from a SCALe secret, compare in constant time, never log it, and support rotation.
8. **Consistent data types:** ISO 8601 dates, explicit units on every quantity, IDs as strings.
9. **Resilient calls to ChemDiary:** short timeouts and bounded retries with backoff, then a clear 502 or 503 with `retryable`.
10. **Observability:** structured logs with the correlation ID, endpoint and outcome; never log payload values or the key. Provide `/health`.
11. **Stateless container, configuration from environment variables**, as for the other AutoLEAD APIs built from the same template.
12. **Backward-compatible changes only within a version;** a template change bumps the version.
13. **Test doubles:** a mock ChemDiary and a sample GBG client, so the API can be tested without the lab.

**Spikes that decide the design**

| # | Spike | Question | Output |
| --- | --- | --- | --- |
| 0 | Access (prerequisite, start now) | Do we have the ChemDiary dev environment, a service account, repo access and the SCALe non-prod namespace? | Everything the spikes need |
| 1 | ChemDiary API walking skeleton | From a FastAPI call, can we create an experiment from the template and fill one block, then every block, including tables and the chromatogram file? | Call sequence per block; list of gaps |
| 2 | Pipeline and deployment skeleton | Can a hello-world container build on the internal runners, push to the registry and run on SCALe non-prod, reachable from GBG's network? | Working pipeline and network path |
| 3 | Template behaviour | Do API writes trigger the automations (Samples and Container tables), and which fields are calculated? | Final endpoint and payload list |
| 4 | GBG interface | Can GBG send an API key header and wait for a synchronous reply with the experiment ID? What are the timeouts and payload sizes? | Agreed contract with BioZero |
| 5 | State and idempotency | Is there a persistent store on SCALe? If not, can we keep GBG's run ID in the experiment and look it up? | Idempotency design |

Start spikes 1 and 2 in parallel, run spike 4 as a conversation with BioZero meanwhile, and do spike 3 after spike 1.
