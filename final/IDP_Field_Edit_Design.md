# Field edit design — path addressing, field spec, projections, downstream validation

> **Status:** Patch spec against `IDP_LLD.md` / `IDP_API_Reference.md` §2.5  
> **Created:** 2026-08-14  
> **Applies to:** `GET .../details`, `GET .../extracted-payload`, `POST .../fields`, and `fss_payment_data_ingest_details`  
> **Does not change:** header (read-only), handoff BPMN, extraction HTTP contract

This document **supersedes** client-sent `path[]` on GET/POST. YAML still holds `path`; the wire is `{ fieldGroup, fieldName, occurrenceIndex }`. Batching, audit grain, and `Confidence`-never-overwritten stay.

A per-entity **update class** that hardcodes `details[].data[]` is the trap. One edit service + one walker + **per-entity field spec**.

**Companion:** run [IDP_Field_Edit_REPO_SCAN_PROMPT.md](./IDP_Field_Edit_REPO_SCAN_PROMPT.md) in the live Java workspace **before** coding. Apply scan corrections here, then implement.

---

## 1. Why this patch

YAML `path` can describe `fx.rate` and (later) nested arrays. The **HTTP client must not send those keys** — that is how stored JSON is malformed. The wire is `fieldGroup` + `fieldName` + `occurrenceIndex` (slot only). Indonesia has at most one repeating segment (`details[]`).

A per-entity **update class** that hardcodes `details[].data[]` is the trap. One edit service + one walker + **per-entity field spec**.

---

## 2. Decisions (locked)

| # | Decision |
|---|----------|
| D1 | Wire address is `{ fieldGroup, fieldName, fieldValue }` plus `occurrenceIndex` **only when the field-spec path has a repeating segment**. The client **never** sends `path` or JSON keys. |
| D2 | `fieldGroup` is the first-level key under `initiationDetail` (or `initiationDetail` for instruction-root). **Walk `path` lives only in YAML.** |
| D3 | Resolution is rooted at `initiationDetail`. `header` cannot be expressed. |
| D4 | Server builds the walk: field-spec `path` + client `occurrenceIndex` applied to the one `repeating: true` segment. A client-supplied `path` is **ignored and rejected** (`400`). |
| D5 | Triples in `data[]` are found by `fieldName` (`Name`), never by slot in `data[]`. |
| D6 | One `ExtractionFieldEditService`. One `FieldSpecAccessor`. New country = field-spec YAML (+ `EntityPaymentMapper` for handoff). New Java accessor **only** if the value model is not `{ Name, Value, Confidence }` and not a scalar. |
| D7 | Client never sends table columns (`clientName`, `trnTyp`, `transactionRef`) and never sends `old_value`, `Confidence`, or `path`. |
| D8 | Promoted columns are field-spec `column:` projections. Same projector on **extract insert** and **field save**. |
| D9 | Repeating JSON fields that map to a **single** column must set `occurrence` (usually `0`). |
| D10 | Persist only after **shape** validation (JSON → `StoredExtractedData`). **Downstream DTO** validation (`EntityPaymentMapper` → IAP/`Payment`/`IAPSSTMRequestDTO`) on **Submit**; optional on Save draft. |
| D11 | Downstream validation uses the **same mapper as handoff**, not `objectMapper.readValue(json, IAPSSTMRequestDTO.class)`. |
| D12 | Failed validation rolls back JSON, columns, and audit (one transaction). |
| D13 | Details GET `fields[]` uses the **same** `{ fieldGroup, fieldName, occurrenceIndex }` as POST. Path is not on the wire. |
| D14 | Save draft sends only dirty instructions / dirty fields from that GET list. Untouched rows are omitted. The GET list size is not a payload problem. |
| D15 | Instruction-root scalars (`ActivityId`, `SubActivityId`) use `fieldGroup: initiationDetail` (node = the instruction object, not `initiationDetail.initiationDetail`). |
| D16 | Coupled copies: **`replicate-to`** on the primary. Each entry is the **same YAML object** as a field (`field-group`, `field-name`, `kind`, `path`, `triples-at`) so Spring binds it to a record — no `Group.Name` string split. Repeating `path` updates **all** occurrences. |
| D17 | `ActivityId` and `SubActivityId` are independent editable scalars. They are **not** copies of each other and there is **no** allowed-pairs table. |
| D18 | **Two GETs by `uploadId`.** `GET .../details` returns field-spec `fields[]` and UI summaries — **no** `extractedData`. `GET .../extracted-payload` returns stored JSON for external systems — **no** `fields[]`. The portal must not call the payload GET to build the grid. |

---

## 3. Request / response patch

### 3.0 Split GET by `uploadId` (do not overload `getDetail`)

Today `getDetail` mixes **form state** (grids, chips, Save draft) with **document payload** (`extractedData` / `header`). That forces the portal to download instruction JSON it must not walk, and forces integrators to understand `fields[]` / `occurrenceIndex` they do not own.

| API | Audience | Returns | Does not return |
|-----|----------|---------|-----------------|
| **Details** (form / UI) | Portal modal | File chip + left-panel summaries + `fields[]` | `extractedData`, `header` blob |
| **Extracted payload** (integration) | External / batch / ops | File meta + per-instruction stored JSON | `fields[]`, field-spec `kind` / `editable` |
| **List** `getUploads` | Portal table | Unchanged — no instruction JSON | — |

Same `uploadId`, same entity-scoped `404`. Different `@PreAuthorize` if integration is not `paymentMaker` (scan for an existing integration / API-user role; do not invent one in code until the scan names it).

Both GETs are read-only. **Save draft still `POST .../fields`.** The details GET is the only source the portal uses to build that POST.

#### 3.0.1 `GET .../extraction-uploads/details?uploadId={id}` — form / UI

Use this in the modal. Flatten via field spec `list()`. **No `extractedData`.**

`path` is field-spec-only (see below). Wire address is `fieldGroup` + `fieldName` + `occurrenceIndex`.

```json
{
  "uploadId": "b2c3d4e5-…",
  "fileId": "26IDJY00002",
  "fileName": "3897122.pdf",
  "entity": "ID",
  "status": "READY_FOR_REVIEW",
  "instructionCount": 2,
  "confidence": 91.2,
  "camundaTasks": [
    { "taskId": "…", "taskDefinitionKey": "Extraction_UserReview", "assignee": "user_id" }
  ],
  "instructions": [
    {
      "detailId": "0190f2a1-…0001",
      "instructionIndex": 0,
      "status": "READY_FOR_REVIEW",
      "version": 0,
      "trnTyp": "Redemption",
      "txRef": "3897122-001",
      "clientName": "PT BNI ASSET MANAGEMENT",
      "debitAmount": "5000000000",
      "confidenceScore": 97.23,
      "lowConfidenceFieldCount": 0,
      "errorDesc": null,
      "fields": [
        {
          "fieldGroup": "TransactionDetails",
          "fieldName": "ClntNm",
          "occurrenceIndex": null,
          "fieldValue": "PT BNI ASSET MANAGEMENT",
          "confidence": "93.23",
          "kind": "triple",
          "editable": true
        },
        {
          "fieldGroup": "DebitDetails",
          "fieldName": "DrAccNo",
          "occurrenceIndex": 0,
          "fieldValue": "30681655612",
          "confidence": "92.58",
          "kind": "triple",
          "editable": true
        }
      ]
    }
  ]
}
```

Left panel uses `trnTyp` / `txRef` / `clientName` / `debitAmount` / `confidenceScore` / `status` so it does not scan `fields[]`. Right panel binds **only** to `fields[]`.

| Details `fields[]` | Rule |
|-------------------|------|
| `fieldGroup`, `fieldName` | Field-spec key. Echo on POST. |
| `occurrenceIndex` | `null` if YAML `path` has no `repeating: true`; else 0-based slot. |
| `fieldValue` | Grid edits this only. |
| `confidence` | Read-only. |
| `kind` | `triple` \| `scalar` |
| `editable` | `false` if listed in any `replicate-to`. |

**POST `fields`:** echo `fieldGroup`, `fieldName`, `occurrenceIndex`; new `fieldValue`. Do not send `path`, `confidence`, `kind`, or JSON.

**Deprecate** UI use of `GET .../getDetail` once the portal is on `/details`. Keep `getDetail` as a thin alias to `/details` **without** `extractedData` during one release, or remove `extractedData` from `getDetail` in the same release so the old URL cannot leak the blob to the modal.

#### 3.0.2 `GET .../extraction-uploads/extracted-payload?uploadId={id}` — external / integration

For systems that need the **stored document** after extract or after maker edits. One call per upload, all instructions. This is **not** the form model.

```json
{
  "uploadId": "b2c3d4e5-…",
  "fileId": "26IDJY00002",
  "entity": "ID",
  "status": "READY_FOR_REVIEW",
  "instructionCount": 2,
  "instructions": [
    {
      "detailId": "0190f2a1-…0001",
      "instructionIndex": 0,
      "status": "READY_FOR_REVIEW",
      "trnTyp": "Redemption",
      "transactionRef": "3897122-001",
      "messageId": null,
      "extractedData": {
        "header": { "uniqueId": "3897122", "…": "…" },
        "initiationDetail": { "TxRef": "3897122-001", "TransactionDetails": [ "…" ], "DebitDetails": { "…" } }
      }
    }
  ]
}
```

| Include | Exclude |
|---------|---------|
| `extractedData` as stored (`header` + one `initiationDetail`) | `fields[]`, `kind`, `editable`, `occurrenceIndex` |
| `detailId`, `instructionIndex`, `status`, `trnTyp` | Field-spec flatten |
| `messageId` after handoff | Camunda task DTOs (portal-only) |

Consumers parse JSON with **their** POJOs (or the shared extraction types). They do not implement our field spec.

**Optional later:** `GET .../extracted-payload?uploadId=&instructionIndex=` if *N* is huge. v1 returns all instructions for the upload.

**Not for Save draft.** Editing this blob and PUTting it back is forbidden (same as posting `extractedData`).

`path` on the **details** wire stays field-spec-only. Repeating slots:

- `occurrenceIndex: null` — `ClntNm`, `SubActivityId`, `rate`
- `occurrenceIndex: 0` / `1` — debit/credit `details[]`

Grid key: `(fieldGroup, fieldName, occurrenceIndex)`.

### 3.1 `POST /api/fss/payments/gateway/v1/extraction-uploads/fields?uploadId={id}`

Unchanged: batched `instructions[]`, atomic, `detailId` ownership check, `READY_FOR_REVIEW` only, `@Version` → `409 CONCURRENT_EDIT`.

**Replace** each field item.

**POST body** — the field spec derives the walk. `path` in the body is `400 PATH_NOT_ALLOWED`.

```json
{
  "instructions": [
    {
      "detailId": "0190f2a1-7c3d-7e11-9b2a-000000000001",
      "fields": [
        {
          "fieldGroup": "TransactionDetails",
          "fieldName": "ClntNm",
          "occurrenceIndex": null,
          "fieldValue": "PT BNI ASSET MANAGEMENT - CORRECTED"
        },
        {
          "fieldGroup": "DebitDetails",
          "fieldName": "DrAccNo",
          "occurrenceIndex": 0,
          "fieldValue": "30681655612"
        },
        {
          "fieldGroup": "DebitDetails",
          "fieldName": "rate",
          "occurrenceIndex": null,
          "fieldValue": "1.25"
        }
      ]
    }
  ]
}
```

| Element | Rule |
|---------|------|
| `fieldGroup` | Field-spec group key. |
| `fieldName` | Field-spec field key (`Name` or scalar property). |
| `occurrenceIndex` | Required iff that field’s YAML `path` has `repeating: true`. Forbidden (`null`/omit) otherwise. Out of range → `400`. |
| `fieldValue` | New value only. |
| `path` | **Must not appear.** Server uses YAML `path` + `occurrenceIndex`. |

**POST response** — same `fields[]` shape as GET (no `path`).

Indonesia — GET and POST:

| Edit | `fieldGroup` | `fieldName` | `occurrenceIndex` |
|------|----------------|-------------|-------------------|
| Client name | `TransactionDetails` | `ClntNm` | `null` |
| Sub-activity | `initiationDetail` | `SubActivityId` | `null` |
| 1st debit account | `DebitDetails` | `DrAccNo` | `0` |
| 2nd debit amount | `DebitDetails` | `DrAmt` | `1` |
| FX rate | `DebitDetails` | `rate` | `null` (`path` in YAML is `{ key: fx }`, not repeating) |

Future two repeating lists: `occurrenceIndexes: [0, 1]` in YAML segment order — still no keys on the wire.

### 3.2 Errors (additions)

| Code | Condition |
|------|-----------|
| `400 PATH_NOT_ALLOWED` | Body contains `path` |
| `400` | Unknown `fieldGroup`/`fieldName`; `occurrenceIndex` present on a non-repeating field; missing/out-of-range on a repeating field |
| Unchanged | empty `instructions[]`, `403` ownership, `404` upload, `409` status / concurrent edit |

---

## 4. Field spec (groups + `path` + `replicate-to`)

This section is the **shape to implement**. Bind it with `@ConfigurationProperties(prefix = "extraction")`. Do not invent a second addressing model in Java.

### 4.0 YAML tree (what each level is)

```text
extraction:
  field-specs:                          # Map<String, EntityFieldSpec>
    ID:                                    # entity code (same as upload.entity)
      groups:                              # Map<String, GroupDef>  — key = JSON key under initiationDetail
                                           # exception: key "initiationDetail" = the instruction object itself
        <groupKey>:
          kind: …                          # instruction-root | triple-list | object
          fields:                          # Map<String, FieldDef> — key = fieldName (JSON Name or property)
            <fieldName>:
              kind: …                      # triple | scalar
              path: [ { key, repeating } ] # walk from the group node to the triples/scalar parent
              triples-at: …                # only kind: triple, and only when triples are not the group itself
              column: …                    # optional ingest-table projection
              occurrence: …                # optional; with column, which repeating index feeds the column
              replicate-to: [ FieldTarget ]# optional; same shape as a walker call, not a string
```

**JSON the walker starts from** (stored `extracted_data`):

```text
initiationDetail                          ← instruction-root group
  ActivityId, SubActivityId, …            ← scalars, path: []
  TransactionDetails: [ { Name, Value, Confidence }, … ]   ← triple-list, path: []
  DebitDetails:                           ← object
    details[]:                            ← path segment repeating: true
      data: [ { Name, Value, Confidence } ]  ← triples-at: data
    fx.rate                               ← scalar, path: [{ key: fx }]  (when present)
  CreditDetails:                          ← same as DebitDetails
```

`header` is never a group.

#### Group (`groups.<groupKey>`)

| `kind` | YAML `groupKey` | JSON node | Typical `path` on fields |
|--------|-----------------|-----------|---------------------------|
| `instruction-root` | `initiationDetail` | The `initiationDetail` **object** (do **not** do `node.get("initiationDetail")` again) | `[]` — field is a property on that object |
| `triple-list` | `TransactionDetails` | `initiationDetail.TransactionDetails` which **is** `List<{Name,Value,Confidence}>` | always `[]`; `triples-at` omitted (the node is the list) |
| `object` | `DebitDetails`, `CreditDetails` | `initiationDetail.DebitDetails` object | non-empty: walk into `details[]`, `fx`, … |

Missing `kind` on `DebitDetails` is invalid — defaulting it to `triple-list` would look for `{Name,Value}` on the DebitDetails object and fail.

#### Field (`fields.<fieldName>`)

The map key **is** `fieldName`. For triples it equals JSON `Name` (`ClntNm`, `Transaction-Id`). For scalars it equals the JSON property (`ActivityId`, `rate`).

| `kind` | JSON | `path` | `triples-at` |
|--------|------|--------|----------------|
| `triple` on `triple-list` group | `{ Name, Value, Confidence }` in that array | **must be `[]`** | **omit** |
| `triple` on `object` group | same triple, nested | `[{ key: details, repeating: true }]` | `data` (property on each `details[i]` that holds the triple list) |
| `scalar` | plain string/number property | `[]` on instruction-root; `[{ key: fx }]` for `DebitDetails.fx.rate` | **omit** |

`path` is always a **list of segments**, never a string. Each segment:

```yaml
- key: details          # JSON property name
  repeating: true       # omit or false = object; true = JSON array, request supplies index
```

| `repeating` | GET/POST | Replicate (no client index) |
|-------------|-----------------|------------------------------|
| omitted / `false` | omit `occurrenceIndex` | set that one object (`fx.rate`) |
| `true` | `occurrenceIndex` = array slot | set **every** array element |

`column` / `occurrence` are projection-only. They do not change the JSON walk. `occurrence: 0` means only `details[0]` updates `debit_amount`.

Quote YAML keys that contain `-`: `"Transaction-Id"`.

#### `replicate-to` (list of `FieldTarget`)

Not `Group.Name`. Same properties the walker needs, because this entry is **not** nested under the target group:

```yaml
replicate-to:
  - field-group: DebitDetails      # → FieldTarget.fieldGroup
    field-name: DrNarrLn1          # → FieldTarget.fieldName
    kind: triple
    path:
      - key: details
        repeating: true
    triples-at: data
```

Must match that target field’s own `kind` / `path` / `triples-at` (duplicated so save does not parse or look up by string).

`field-group` + `field-name` + `kind` + `path` + `triples-at` bind to **one** Java type used for spec fields **and** replicate (see records below). When applying a spec `FieldDef`, the walker is called with `fieldGroup = groupKey`, `fieldName = fields map key`.

#### Java bind (1:1 with YAML)

```java
@ConfigurationProperties(prefix = "extraction")
public record ExtractionProperties(Map<String, EntityFieldSpec> fieldSpecs) {}

public record EntityFieldSpec(Map<String, GroupDef> groups) {}

public record GroupDef(
    String kind,                       // instruction-root | triple-list | object
    Map<String, FieldDef> fields       // key = fieldName
) {}

public record PathSegment(String key, Boolean repeating) {}

public record FieldTarget(
    String fieldGroup,
    String fieldName,
    String kind,
    List<PathSegment> path,
    String triplesAt
) {}

public record FieldDef(
    String kind,
    List<PathSegment> path,
    String triplesAt,
    String column,
    Integer occurrence,
    List<FieldTarget> replicateTo
) {}
```

Boot maps `field-group` → `fieldGroup`, `triples-at` → `triplesAt`, `replicate-to` → `replicateTo`, `repeating: true` → `Boolean.TRUE`. Empty `path: []` → empty list, not null (use `@DefaultValue` / empty list in the binder if needed).

Lookup on POST: `groups.get(fieldGroup).fields().get(fieldName)` then `setValue(groupKey, fieldKey, def)`. Replicate: skip lookup; `setValue(target)` for each `replicateTo` element.

### 4.1 Canonical ID YAML

Nested group YAML. Replication is `List<FieldTarget>` on the primary.

```yaml
extraction:
  field-specs:
    ID:
      groups:
        initiationDetail:
          kind: instruction-root
          fields:
            ActivityId:
              kind: scalar
              path: []
            SubActivityId:
              kind: scalar
              path: []
              replicate-to:
                - field-group: TransactionDetails
                  field-name: TrnTyp
                  kind: triple
                  path: []
                - field-group: DebitDetails
                  field-name: DrNarrLn1
                  kind: triple
                  path:
                    - key: details
                      repeating: true
                  triples-at: data
                - field-group: CreditDetails
                  field-name: CrNarrLn1
                  kind: triple
                  path:
                    - key: details
                      repeating: true
                  triples-at: data
        TransactionDetails:
          kind: triple-list
          fields:
            ClntNm:
              kind: triple
              path: []
              column: client_name
            TrnTyp:
              kind: triple
              path: []
              column: trn_typ
            TrnRef:
              kind: triple
              path: []
              column: transaction_ref
            BankChg:
              kind: triple
              path: []
            Multi:
              kind: triple
              path: []
            NonS2B:
              kind: triple
              path: []
            ScaRef:
              kind: triple
              path: []
            ValDt:
              kind: triple
              path: []
        DebitDetails:
          kind: object
          fields:
            DrAccNo:
              kind: triple
              path:
                - key: details
                  repeating: true
              triples-at: data
            DrAmt:
              kind: triple
              path:
                - key: details
                  repeating: true
              triples-at: data
              column: debit_amount
              occurrence: 0
            DrAccCcy:
              kind: triple
              path:
                - key: details
                  repeating: true
              triples-at: data
            DrAccNm:
              kind: triple
              path:
                - key: details
                  repeating: true
              triples-at: data
            DrNarrLn1:
              kind: triple
              path:
                - key: details
                  repeating: true
              triples-at: data
            "Transaction-Id":
              kind: triple
              path:
                - key: details
                  repeating: true
              triples-at: data
            rate:
              kind: scalar
              path:
                - key: fx
        CreditDetails:
          kind: object
          fields:
            CrAccNo:
              kind: triple
              path:
                - key: details
                  repeating: true
              triples-at: data
            CrAmt:
              kind: triple
              path:
                - key: details
                  repeating: true
              triples-at: data
            CrAccCcy:
              kind: triple
              path:
                - key: details
                  repeating: true
              triples-at: data
            CrAccNm:
              kind: triple
              path:
                - key: details
                  repeating: true
              triples-at: data
            CrNarrLn1:
              kind: triple
              path:
                - key: details
                  repeating: true
              triples-at: data
            "Transaction-Id":
              kind: triple
              path:
                - key: details
                  repeating: true
              triples-at: data
```

Do **not** mix flow-style `{ kind: triple, path: [{ key: details, repeating: true }] }` with block style in production YAML if the binder is picky — the block form above is the one to copy.

Walker: `setValue(stored, FieldTarget, value)`. For `repeating: true` with no request index (replicate), every array index.

```java
for (FieldTarget target : primary.replicateTo()) {
    accessor.setValue(stored, target, newValue);
}
```

MY (triples sit on each `lines[]` element — no `data` wrapper):

```yaml
MY:
  groups:
    Debit:
      kind: object
      fields:
        AccNo:
          kind: triple
          path:
            - key: lines
              repeating: true
          triples-at: "."          # the array element itself is the triple list or a single triple — treat "." as “node is the list”
```

POST: `{ fieldGroup: Debit, fieldName: AccNo, occurrenceIndex: 0 }` (YAML `path` is not on the wire).

### 4.2 `replicate-to` rules

Fixture: `SubActivityId`, `TrnTyp`, `DrNarrLn1`, `CrNarrLn1` all `"Redemption"`. Do **not** replicate to `ActivityId`.

| Rule | Behaviour |
|------|-----------|
| Client POST | May send `SubActivityId`, `ActivityId`, `ClntNm`, … Must **not** send a replicate target → `400 REPLICATE_TARGET_NOT_EDITABLE`. Match on `fieldGroup` + `fieldName`. |
| After apply | Loop `replicateTo`; `setValue` each `FieldTarget`. Repeating `path` → all indexes. Triple: `Value` only. |
| Projector | Runs on changed fields, so `trn_typ` updates when `TrnTyp` was replicated. |
| Audit | One row per changed field (primary + targets). |
| GET | Targets `editable: false`. Primary `editable: true`. Index targets at startup from `replicateTo` lists. |
| Later split | Delete that list item from `replicate-to`. |

**Wrong:** `TransactionDetails.TrnTyp` strings. **Wrong:** `if ("SubActivityId")`. **Wrong:** portal sends four values. **Right:** bound `List<FieldTarget>` and the existing walker.

---



## 5. Ingest table projections

`extracted_data` is the source of truth. Columns are copies for list/audit **without** re-parsing JSON on every read.

The **request never names columns**. After a real value change, `IngestDetailProjector` runs for field-spec entries that have `column:`.

| JSON field (ID) | Typical column | Notes |
|-----------------|----------------|-------|
| `TrnTyp` | `trn_typ` | LLD: only promoted column; keep even if you drop the others |
| `ClntNm` | `client_name` | Only if the entity has this column |
| `TrnRef` or `TxRef` | `transaction_ref` | Map **one** source; do not map `DrAccNo` here |
| `DrAmt` | `debit_amount` | Requires `occurrence: 0` |

**Same projector** on `ExtractionTriggerHandler` insert and on field save. Two writers cause drift.

**Do not** re-parse the whole JSON to refill every column on every save. Only columns whose **changed** fields declare `column:`.

**Do not** promote a repeating field without `occurrence`. Editing `details[1].DrAmt` must not overwrite `debit_amount` unless that is the defined rule.

Physical setter map is **fixed** to columns that exist on `PaymentDataIngestDetailsEntity`. The field spec may only reference those keys. A new DB column is a migration; mapping `CustNm` → existing `client_name` is field-spec-only.

If you follow LLD strictly (no `client_name` / `transaction_ref` columns): omit those `column:` entries; the **details** GET still derives `clientName` / `txRef` from JSON.

---

## 6. Save transaction (mandatory order)

One `@Transactional` per batch:

1. Load details; verify `upload_id`, `READY_FOR_REVIEW`, `@Version`.
2. Validate every address against the field spec; reject POST of a `replicate-to` target (`fieldGroup` + `fieldName`). Apply client edits via walker (`Value` / scalar only).
3. **replicate-to:** if a primary changed, `for (FieldTarget t : primary.replicateTo()) accessor.setValue(stored, t, newValue)` (all indexes when `repeating: true`).
4. **Gate A — shape:** `ExtractedDataCodec.fromJson` / POJO round-trip of the merged document. Fail → rollback.
5. Diff vs stored JSON (includes replicate-to targets); skip no-ops (no audit, no projection).
6. Projector for changed fields with `column:` (includes `trn_typ` when `TrnTyp` was written by replicate).
7. **Gate B — downstream** (Submit always; Save draft only if product requires handoff-ready drafts): `EntityPaymentMapper.toDownstream(stored, meta)` then Bean Validation. **No** `PaymentsCoreClient`. Fail → rollback. Map violations back to `{ fieldGroup, fieldName, occurrenceIndex }` (never DTO property names).
8. Persist `extracted_data` + columns + `updated_by` / `updated_at`.
9. `fss_payment_field_audit` (one row per real change, including replicate-to) + one `FIELD_EDIT` action if any change.
10. Return summaries + `changedFieldCount` + refreshed `fields[]`.

If **no** field changed: `200`, all `changedFieldCount: 0`, no audit row.

---

## 7. Downstream validation

`extracted_data` is **not** `IAPSSTMRequestDTO` / `SSTMIAPRequestDTO`. Direct `readValue(json, IAPSSTMRequestDTO.class)` will fail on a valid instruction.

```text
merged JSON → StoredExtractedData          // Gate A
           → EntityPaymentMapper           // ID: triples → Payment / PaymentData / SSTM DTO
           → jakarta.validation.Validator  // Gate B
```

Split mapper I/O:

- `mapToPayment` / `toDownstream` — structural, no remote calls → used at persist
- `enrichPayment` — value date, account names → handoff / `Initialize_IAP_*` only

New country: implement mapper + field spec. Edit service unchanged.

Suggested error body:

```json
{
  "errorCode": "DOWNSTREAM_MAPPING_FAILED",
  "violations": [
    {
      "fieldGroup": "DebitDetails",
      "fieldName": "DrAccNo",
      "occurrenceIndex": 0,
      "message": "must not be blank"
    }
  ]
}
```

---

## 8. Audit schema patch

`fss_payment_field_audit`:

| Column | Change |
|--------|--------|
| `field_group` | Unchanged |
| `field_name` | Unchanged |
| `occurrence_index` | Wire `occurrenceIndex`; `NULL` when the field is not repeating. **Keep** — this is the client index, not a JSON key. |
| `field_path` | Optional **server-computed** snapshot of YAML walk + index (forensics). Never from the request. |
| `old_value` / `new_value` | Unchanged; server-computed |

If V6 already has `occurrence_index`, keep it. Do not treat client `path` as an audit column.

---

## 9. Implementation blueprint (reduce guesswork)

Follow LLD §9.5.2: snippets below are **Java 17**. If the scan finds 8 or 11, apply the downgrade table in §9.5.2 **before** coding — do not mix `record` into an 8/11 module.

### 9.1 Language features by release (use these, not others)

| Release | Use in this patch | Do not use |
|---------|-------------------|------------|
| **All (8+)** | `enum` for `GroupKind` / `FieldKind`; `EnumMap`; `Objects.requireNonNull`; `Optional` for `occurrenceIndex` / `column`; `Collections.unmodifiableMap` after field-spec load; constructor injection; `final` classes | `Instant.now()` (inject `Clock`); `UUID.randomUUID()` (LLD `IdGenerator`); `Map.get` without a spec key |
| **11** | `var` for **locals only**; `List.of` / `Map.of` for small immutable literals; `String.isBlank()` on `fieldValue` | records, text blocks, sealed |
| **17** | `record` for DTOs, `PathSegment`, `FieldTarget`, `FieldDef`, `GroupDef`; switch **expression** on `GroupKind` (no `default` if exhaustive) | sealed exceptions unless the gateway already uses them |

**Downgrade (8/11):** `record FieldTarget(...)` → `public final class FieldTarget` with the **same** accessor names (`fieldGroup()`, not `getFieldGroup()`). `List.of` → `Collections.unmodifiableList(Arrays.asList(...))`. Switch **expression** → switch **statement** plus `default: throw new IllegalStateException("Unhandled " + kind)`.

### 9.2 Patterns (one each — do not add more)

| Pattern | Type | Why |
|---------|------|-----|
| **Registry** | `EntityFieldSpecRegistry` | `entity` → frozen `EntityFieldSpec`. Same idea as LLD `AbstractKeyedRegistry`. Unknown entity → `UnknownEntityException`. |
| **Strategy** | `FieldKindWriter` (`TRIPLE` / `SCALAR`) | `setValue` does not `if ("triple")`. Register in `EnumMap<FieldKind, FieldKindWriter>`. |
| **Facade** | `ExtractionFieldEditService` | Only public write API for Save draft. Controller does not call the walker. |
| **Adapter** | `FieldDefSupport.toTarget(groupKey, fieldName, def)` | `FieldDef` → `FieldTarget` for `setValue`. |
| **Guard** | `ReplicateTargetIndex` | Startup: every `replicate-to` `(fieldGroup, fieldName)` is not client-writable. |

Do **not** add `IdFieldEditService`, a visitor over JSON, or a chain-of-responsibility for save steps. Save order is private methods on the facade, in §6 order.

### 9.3 Types (names are binding)

```java
public enum GroupKind { INSTRUCTION_ROOT, TRIPLE_LIST, OBJECT }
public enum FieldKind { TRIPLE, SCALAR }

// Bind YAML kind: instruction-root → INSTRUCTION_ROOT (custom converter or @JsonProperty)

public final class FieldAddress {          // wire + audit identity
    private final String fieldGroup;
    private final String fieldName;
    private final Integer occurrenceIndex; // null if not repeating
    // equals/hashCode on all three
}

public interface ExtractedDataFieldAccessor {
    List<EditableFieldDto> list(String extractedJson, EntityFieldSpec spec);
    String setValue(String extractedJson, FieldTarget target, Integer occurrenceIndex, String newValue);
    // occurrenceIndex ignored when target.path has no repeating:true
    // occurrenceIndex null + repeating:true + replicate = ALL indexes
}
```

`FieldSpecAccessor` is the **only** class allowed to use Jackson `JsonNode` / `ObjectNode`. Keys come from `PathSegment.key()` or `fieldName`, never from the HTTP body except as field-spec lookup. Payment mapping stays on POJOs (`ExtractedDataCodec`) per LLD §6.3.

**JsonNode walk (must implement exactly):**

1. `ObjectNode root = (ObjectNode) codec.readTree(extractedJson)`.
2. `JsonNode instruction = root.get("initiationDetail")` — missing → `400`.
3. Resolve group node:
   - `INSTRUCTION_ROOT` → `instruction` itself.
   - else → `instruction.get(fieldGroup)`; null → `400 UNKNOWN_FIELD_GROUP`.
4. For each `PathSegment` in spec `path` (not the request):
   - `repeating == false` / null → `node = node.get(segment.key())`.
   - `repeating == true` → `node = node.get(segment.key())` must be array; if `occurrenceIndex != null` use that index; if null **and** replicate-all, iterate `0..size-1` and apply the rest of the walk per element.
5. `SCALAR` → `ObjectNode.put(fieldName, newValue)` on the parent object. Do not create missing `fx` unless the field spec later says so; v1 missing parent → `400 FIELD_ADDRESS_NOT_FOUND`.
6. `TRIPLE` → locate list: `triples-at` null or omitted on `TRIPLE_LIST` group → the group node is the array; `triples-at: data` → `node.get("data")`; `triples-at: "."` → current node is the list (or wrap single object). Find element with `Name` equal to `fieldName`. Duplicate `Name` → `400`. Set **`Value` only**. Do not touch `Confidence`. Optionally set `isEdited: true` if the POJO has that field.
7. `root.toString()` (or codec) back to `extracted_data`. Preserve sibling properties.

### 9.4 Field-spec load (startup)

`EntityFieldSpecRegistry` `@PostConstruct` (or constructor after bind):

1. Fail start if `groups` empty, `kind` missing, `DebitDetails` without `OBJECT`.
2. Count `repeating: true` per field: ID allows 0 or 1. More than one → fail start (v1) until `occurrenceIndexes` exists.
3. Build `Set<FieldAddress>` of replicate targets (`occurrenceIndex` unused; match group+name).
4. Wrap `groups` / `fields` in `unmodifiableMap`.
5. Do not reload YAML per request.

### 9.5 Save facade (method contracts)

```java
public interface ExtractionFieldEditService {
    FieldEditResult save(UUID uploadId, String actor, FieldEditRequest request);
}

public final class FieldEditRequest { // or record
    // instructions: List<InstructionEdits>
    // InstructionEdits: detailId, List<FieldAssignment>
    // FieldAssignment: fieldGroup, fieldName, Integer occurrenceIndex, fieldValue
    // REJECT if JSON has "path" — use @JsonIgnoreProperties(ignoreUnknown = false) or a setter that throws
}

public final class FieldEditResult {
    UUID uploadId;
    Instant savedAt;              // Clock.instant()
    List<InstructionSaveResult> instructions;
    // InstructionSaveResult: detailId, status, changedFieldCount, fields (same as details GET)
}
```

`save` is `@Transactional`. Load rows with `@Lock` none — rely on `@Version`. Per instruction: decode → apply assignments → replicate → Gate A (`codec.fromJson`) → diff → projector → persist → audit.

**Diff:** compare `Value` / scalar string with `Objects.equals`. Trim policy: do **not** trim unless the field spec says so (v1: no trim).

**Replicate:** after successful client `setValue` on a field whose `replicateTo` is non-empty, loop `FieldTarget` with `occurrenceIndex = null` so repeating targets update **all** legs.

**Projector:**

```java
public interface IngestDetailProjector {
    void apply(PaymentDataIngestDetailsEntity row, FieldAddress address, FieldDef def, String newValue);
}
```

If `def.column()` empty, return. If `def.occurrence() != null` and `address.occurrenceIndex()` not equal, return. Else `switch (column)` on a **fixed** enum/`if` of known columns (`trn_typ`, `client_name`, `transaction_ref`, `debit_amount`). Unknown `column` in YAML → fail **startup**, not save.

### 9.6 Read path

| Method | SQL | CPU |
|--------|-----|-----|
| `GET .../details` | Load meta + detail rows (need `extracted_data`) | `list()` per row; do not return raw JSON |
| `GET .../extracted-payload` | Same load | `extractedData` string/parsed object; **do not** call `list()` |
| `GET .../getUploads` | Meta only | no details |

Do not `SELECT *` on a combined query used for both — two repository methods or two DTO projections.

### 9.7 Controller mapping

```
GET  /api/fss/payments/gateway/v1/extraction-uploads/details?uploadId=
GET  /api/fss/payments/gateway/v1/extraction-uploads/extracted-payload?uploadId=
POST /api/fss/payments/gateway/v1/extraction-uploads/fields?uploadId=
```

`getDetail` → delegate to details assembler, **strip** `extractedData` if the old method remains.

### 9.8 Package (under existing ingestion package)

```
...ingestion.edit.ExtractionFieldEditService
...ingestion.spec.FieldSpecAccessor
...ingestion.edit.FieldKindWriter
...ingestion.edit.TripleFieldWriter
...ingestion.edit.ScalarFieldWriter
...ingestion.edit.IngestDetailProjector
...ingestion.edit.ReplicateTargetIndex
...ingestion.spec.EntityFieldSpecRegistry
...ingestion.spec.GroupKind
...ingestion.spec.FieldKind
...ingestion.spec.FieldDef
...ingestion.spec.FieldTarget
...ingestion.spec.PathSegment
...ingestion.dto.FieldAssignmentDto
...ingestion.dto.EditableFieldDto
...ingestion.dto.UploadDetailsResponse   // GET details
...ingestion.dto.ExtractedPayloadResponse
```

### 9.9 Logging

`INFO`: save with `uploadId`, instruction count, `changedFieldCount` sum. Never log `fieldValue` / `extracted_data`. `WARN`: `REPLICATE_TARGET_NOT_EDITABLE`, `PATH_NOT_ALLOWED`.

---

## 10. What not to do (examples)

**Do not send columns on the request**

```json
{ "detailId": "…", "clientName": "PT BNI…", "trnTyp": "Subscription" }
```

**Do not mix JSON edit and a column**

```json
{ "fields": [{ "fieldName": "ClntNm", "fieldValue": "A" }], "clientName": "B" }
```

**Do not replace whole `extracted_data`**

```json
{ "extractedData": { "header": {}, "initiationDetail": {} } }
```

**Do not deserialize extraction JSON as the IAP DTO**

```java
objectMapper.readValue(mergedJson, IAPSSTMRequestDTO.class);
```

**Do not refill every column from a full JSON parse on a one-field save**

```java
row.setClientName(read(json, "ClntNm"));
row.setTrnTyp(read(json, "TrnTyp")); // untouched field rewritten
```

**Do not map repeating `DrAmt` to `debit_amount` without `occurrence: 0`**

Second-leg edit would overwrite the instruction summary column.

**Do not send YAML `path` on GET/POST.** ID still uses `occurrenceIndex` for `details[]`. Nested keys stay in YAML.

**Do not walk `extractedData` on the client.** Bind the grid to details GET `fields[]`; echo `fieldGroup`, `fieldName`, `occurrenceIndex` only.

**Do not POST extracted-payload JSON.** Save is field assignments.

**Do not key a grid cell by `fieldName` alone.** Two debit legs share `DrAccNo`; the key is `(fieldGroup, fieldName, occurrenceIndex)`.

---

## 11. Test matrix (minimum)

| # | Case | Expect |
|---|------|--------|
| 1 | `ClntNm` save | JSON `Value` changed; `Confidence` unchanged; `client_name` updated iff the field spec has `column`; audit 1 row |
| 2 | `DrAccNo` `occurrenceIndex` 0 | Triple under `details[0].data`; no `trn_typ` write |
| 3 | `DrAmt` `occurrenceIndex` 1 with YAML `occurrence: 0` on column | JSON updated; `debit_amount` unchanged |
| 4 | `rate` (`occurrenceIndex` null) | Scalar `DebitDetails.fx.rate`; 400 if `fx` missing |
| 5 | Replay identical save | `changedFieldCount: 0`; no audit |
| 6 | Merge that fails Jackson/`StoredExtractedData` | 400; no persist |
| 7 | Submit with blank `DrAccNo` | `DOWNSTREAM_MAPPING_FAILED`; no persist |
| 8 | `detailId` from another upload | 403; no persist |
| 9 | Stale `@Version` | 409; no persist |
| 10 | MY field spec `lines` without new Java walker | AccNo edit works if YAML registered |
| 11 | Details GET vs fixture | `ClntNm` `occurrenceIndex` null; `DrAccNo` `0`; no `path`; **no** `extractedData` |
| 11b | Extracted-payload GET | each instruction has `extractedData.header` + `initiationDetail`; **no** `fields[]` |
| 12 | Save echoes GET addresses only | Round-trip: GET → change `fieldValue` → POST without `path` |
| 13 | Two credit legs | Two `CrAccNo` entries differ only by `occurrenceIndex` |
| 13a | POST with `path` | `400 PATH_NOT_ALLOWED`; no persist |
| 14 | POST only `SubActivityId` = `Subscription` | `TrnTyp`, every `DrNarrLn1`/`CrNarrLn1` updated via `replicate-to`; `trn_typ` column updated; `ActivityId` unchanged; triple `Confidence` unchanged |
| 15 | POST `TrnTyp` while it is a `replicate-to` target | `400 REPLICATE_TARGET_NOT_EDITABLE` |
| 16 | Two debit legs + `replicate-to` `DrNarrLn1` | both legs’ narration match `SubActivityId` |

---

## 12. LLD / API sections this document patches

When applying to the pack (after repo scan Deliverable E):

| Location | Change |
|----------|--------|
| API §2.4 | Split: `GET .../details` (UI) vs `GET .../extracted-payload` (integration). `getDetail` deprecated or aliased to `/details` **without** blob |
| API §2.5 | No client `path`. `{ fieldGroup, fieldName, occurrenceIndex, fieldValue }`; YAML supplies the walk |
| LLD §5.5 | Walker `list`/`setValue` + projector + gates A/B |
| LLD §5.1 field audit | Keep `occurrence_index`; optional server-computed `field_path` |
| LLD §6.5 | Addressing relative to `initiationDetail` via `path` |
| LLD §9.1 / §9.3 | `FieldSpecAccessor`, projector, mapper validate |
| LLD §4.1 new-entity checklist | Field-spec YAML instead of a new `*ExtractedDataFieldAccessor` (unless value model differs) |
| UX field grids | Bind to GET `fields[]`; Save draft filters dirty cells; do not parse `extractedData` |

---

## 13. Implementation order

0. Repo scan ([IDP_Field_Edit_REPO_SCAN_PROMPT.md](./IDP_Field_Edit_REPO_SCAN_PROMPT.md)); resolve Blockers; apply Deliverable E into this file if the real DTO/column names differ.  
1. DDL / entity: `field_path`; confirm which ingest columns exist.  
2. Field-spec properties + ID YAML from `after_ocr-llm-output.json`.  
3. `FieldSpecAccessor.list` + `.setValue` + tests against the fixture.  
4. Details GET assembler: `instructions[].fields[]` via `list()`; omit `extractedData`.  
4b. Extracted-payload GET: return stored JSON only; no field-spec flatten.  
5. `FieldEditRequest` / `EditableFieldDto` + controller.  
6. Projector + wire into extract insert and save.  
7. Gate A in edit service.  
8. Gate B via existing mapper type found in scan (Submit first).  
9. Portal: **details** GET only; grid = `fields[]`; Save draft echoes dirty editable fields.  
10. Do not send `path` on GET/POST. Keep `occurrenceIndex` for repeating field-spec fields.  
11. Honour `replicate-to` in the edit service (loop `List<FieldTarget>`, existing `setValue`).
