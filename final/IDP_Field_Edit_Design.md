# Field edit design — path addressing, catalog, projections, downstream validation

> **Status:** Patch spec against `IDP_LLD.md` / `IDP_API_Reference.md` §2.5  
> **Created:** 2026-08-14  
> **Applies to:** `GET .../getDetail`, `POST .../extraction-uploads/fields`, and `fss_payment_data_ingest_details`  
> **Does not change:** header (read-only), handoff BPMN, extraction HTTP contract

This document **supersedes** LLD/API use of a single `occurrenceIndex` as the inner address. The REST resource, batching, audit grain, and `Confidence`-never-overwritten rules stay.

**Companion:** run [IDP_Field_Edit_REPO_SCAN_PROMPT.md](./IDP_Field_Edit_REPO_SCAN_PROMPT.md) in the live Java workspace **before** coding. Apply scan corrections here, then implement.

---

## 1. Why this patch

`occurrenceIndex` encodes **one** repeating list under a first-level group (`DebitDetails.details[i].data[]`). That matches Indonesia `id-payment` today. It does **not** address:

- a nested object (`DebitDetails.fx.rate`)
- a second array under the same group (`charges[]` beside `details[]`)
- an array inside an array (`details[i].intermediaries[j]`)

Replacing `occurrenceIndex` after go-live would break the portal **and** `fss_payment_field_audit`. Ship a **path** now. Indonesia is just a one-segment path.

A per-entity **update class** that hardcodes `details[].data[]` is the same trap. One edit service + one walker + **per-entity catalog data**.

---

## 2. Decisions (locked)

| # | Decision |
|---|----------|
| D1 | Request address is `{ fieldGroup, path[], fieldName, fieldValue }`. **No** `occurrenceIndex`. |
| D2 | `fieldGroup` is the **first-level** key under `initiationDetail` only. `path[]` is the walk inside that group. |
| D3 | Resolution is rooted at `initiationDetail`. `header` cannot be expressed. |
| D4 | `path[]` is a list of `{ key, index? }`. `index` is present **iff** that key is a repeating array in the catalog. |
| D5 | Triples in `data[]` are found by `fieldName` (`Name`), never by slot in `data[]`. |
| D6 | One `ExtractionFieldEditService`. One `CatalogExtractedDataFieldAccessor`. New country = catalog YAML (+ `EntityPaymentMapper` for handoff). New Java accessor **only** if the value model is not `{ Name, Value, Confidence }` and not a scalar. |
| D7 | Client never sends table columns (`clientName`, `trnTyp`, `transactionRef`) and never sends `old_value` or `Confidence`. |
| D8 | Promoted columns are catalog `column:` projections. Same projector on **extract insert** and **field save**. |
| D9 | Repeating JSON fields that map to a **single** column must set `occurrence` (usually `0`). |
| D10 | Persist only after **shape** validation (JSON → `StoredExtractedData`). **Downstream DTO** validation (`EntityPaymentMapper` → IAP/`Payment`/`IAPSSTMRequestDTO`) on **Submit**; optional on Save draft. |
| D11 | Downstream validation uses the **same mapper as handoff**, not `objectMapper.readValue(json, IAPSSTMRequestDTO.class)`. |
| D12 | Failed validation rolls back JSON, columns, and audit (one transaction). |
| D13 | `GET getDetail` flattens **editable** fields with the **same** `{ fieldGroup, path, fieldName }` the save API accepts. The portal **echoes** those coordinates and sends a new `fieldValue`. It does **not** walk `extractedData` to invent `path[]`. |
| D14 | Save draft sends only dirty instructions / dirty fields from that GET list. Untouched rows are omitted. The GET list size is not a payload problem. |

---

## 3. Request / response patch

### 3.0 `GET .../getDetail?uploadId={id}` — flatten addresses (required)

`ingestDetails[]` stays one object per instruction (`detailId`, summary columns, status). **Add** `fields[]` on each item. The catalog walker **lists** every editable field (same spine as save).

The modal right-hand grid binds to `fields[]`, not to `extractedData`. `extractedData` may remain on the response for diagnostics / PDF correlation; **it is not an input to Save draft**. A new country changes catalog YAML; GET `fields[]` shape is unchanged.

Emit one `fields[]` entry per catalog field that exists in the stored JSON **or** is omitted by the LLM (then `fieldValue` is `""` / `null` so the maker can fill it). Do not emit `header`. Do not emit `Confidence1` / `TxRef` unless they are in the catalog as editable.

```json
{
  "uploadId": "b2c3d4e5-…",
  "entity": "ID",
  "instructionCount": 2,
  "ingestDetails": [
    {
      "detailId": "0190f2a1-7c3d-7e11-9b2a-000000000001",
      "instructionIndex": 0,
      "status": "READY_FOR_REVIEW",
      "version": 0,
      "trnTyp": "Redemption",
      "txRef": "3897122-001",
      "clientName": "PT BNI ASSET MANAGEMENT",
      "debitAmount": "5000000000",
      "confidenceScore": 97.23,
      "lowConfidenceFieldCount": 0,
      "fields": [
        {
          "fieldGroup": "TransactionDetails",
          "path": [],
          "fieldName": "ClntNm",
          "fieldValue": "PT BNI ASSET MANAGEMENT",
          "confidence": "93.23",
          "kind": "triple"
        },
        {
          "fieldGroup": "TransactionDetails",
          "path": [],
          "fieldName": "TrnTyp",
          "fieldValue": "Redemption",
          "confidence": "94.69",
          "kind": "triple"
        },
        {
          "fieldGroup": "DebitDetails",
          "path": [{ "key": "details", "index": 0 }],
          "fieldName": "DrAccNo",
          "fieldValue": "30681655612",
          "confidence": "92.58",
          "kind": "triple"
        },
        {
          "fieldGroup": "CreditDetails",
          "path": [{ "key": "details", "index": 0 }],
          "fieldName": "CrAmt",
          "fieldValue": "5000000000",
          "confidence": "94.38",
          "kind": "triple"
        }
      ]
    }
  ]
}
```

| GET `fields[]` element | Rule |
|------------------------|------|
| `fieldGroup`, `path`, `fieldName` | **Identical** to POST. Copy into the save body; do not rebuild. |
| `fieldValue` | Current `Value` (triple) or scalar. This is what the grid edits. |
| `confidence` | Triple `Confidence` as stored (string). Scalars omit or `null`. **Read-only in UI.** |
| `kind` | `triple` \| `scalar` — grid can show confidence highlight only for `triple`. |

**How the portal builds the POST from this list (small):**

1. User edits cells in memory (`fieldValue` on the GET object).
2. On Save draft, for each `ingestDetails` row where any `fields[]` item differs from the last loaded/saved snapshot: include `{ detailId, fields: [ { fieldGroup, path, fieldName, fieldValue } ] }` for **changed items only**.
3. Do not send `confidence`, `kind`, `clientName`, or `extractedData`.
4. One HTTP call, `instructions[]` = dirty rows only. Eighteen instructions with three edits each → three inner objects, not eighteen blobs.

**Two debit legs** appear as two GET rows that share `fieldGroup`/`fieldName` and differ by `path[].index`. The grid keys a cell by `(fieldGroup, path, fieldName)`, not by `fieldName` alone.

**Optimistic lock:** return `version` on each ingest row (JPA `@Version`). Save may send it if the current API already does; if not, `409 CONCURRENT_EDIT` still requires re-GET and re-apply. After a successful save, replace that row’s `fields[]` from the save response (or re-GET) so the dirty snapshot resets.

**Same walker as save:** `CatalogExtractedDataFieldAccessor.list(stored, catalog)` for GET; `.setValue(...)` for POST. One catalog, two operations. Do not write a second flatten in the `getDetail` assembler.

### 3.1 `POST /api/fss/payments/gateway/v1/extraction-uploads/fields?uploadId={id}`

Unchanged: batched `instructions[]`, atomic, `detailId` ownership check, `READY_FOR_REVIEW` only, `@Version` → `409 CONCURRENT_EDIT`.

**Replace** each field item.

Was:

```json
{ "fieldGroup": "DebitDetails", "occurrenceIndex": 0, "fieldName": "DrAccNo", "fieldValue": "30681655612" }
```

Becomes:

```json
{
  "instructions": [
    {
      "detailId": "0190f2a1-7c3d-7e11-9b2a-000000000001",
      "fields": [
        {
          "fieldGroup": "TransactionDetails",
          "path": [],
          "fieldName": "ClntNm",
          "fieldValue": "PT BNI ASSET MANAGEMENT - CORRECTED"
        },
        {
          "fieldGroup": "DebitDetails",
          "path": [{ "key": "details", "index": 0 }],
          "fieldName": "DrAccNo",
          "fieldValue": "30681655612"
        },
        {
          "fieldGroup": "DebitDetails",
          "path": [{ "key": "fx" }],
          "fieldName": "rate",
          "fieldValue": "1.25"
        }
      ]
    }
  ]
}
```

| Element | Rule |
|---------|------|
| `fieldGroup` | First-level key on `initiationDetail`. Entity catalog. |
| `path` | Ordered segments. Empty `[]` = field lives on the group (triple-list or scalar). |
| `path[].key` | Next JSON property name. |
| `path[].index` | 0-based array index. Required when catalog marks `repeating: true`; **forbidden** on objects. |
| `fieldName` | Triple `Name`, or scalar property at the walked node. |
| `fieldValue` | New value only. Copied from the edited GET cell. |

**POST response** — keep `changedFieldCount` and summary fields. **Also return `fields[]`** for each saved instruction (same shape as GET) so the modal can replace that row without walking JSON and without a full `getDetail` of all *N* rows.

Indonesia addresses (fixture `after_ocr-llm-output.json`) — identical on GET `fields[]` and POST:

| Edit | `fieldGroup` | `path` | `fieldName` |
|------|----------------|--------|-------------|
| Client name | `TransactionDetails` | `[]` | `ClntNm` |
| Trn type | `TransactionDetails` | `[]` | `TrnTyp` |
| 1st debit account | `DebitDetails` | `[{ "key": "details", "index": 0 }]` | `DrAccNo` |
| 2nd debit amount (if two legs) | `DebitDetails` | `[{ "key": "details", "index": 1 }]` | `DrAmt` |
| Nested FX (when present) | `DebitDetails` | `[{ "key": "fx" }]` | `rate` |
| Array-in-array (future) | `DebitDetails` | `[{ "key": "details", "index": 0 }, { "key": "intermediaries", "index": 1 }]` | `IntAccNo` |

### 3.2 Errors (additions)

| Code | Condition |
|------|-----------|
| `400` | `path` segment `index` on a non-array key; missing `index` on a repeating key; `index` out of range; path not in catalog for that `fieldName`; `fieldGroup=header` (unresolvable) |
| Unchanged | empty `instructions[]`, unknown `fieldName`, `403` ownership, `404` upload, `409` status / concurrent edit |

---

## 4. Catalog (per entity — the extension point)

Config prefix `extraction.field-catalogs.{entity}`. ID example:

```yaml
extraction:
  field-catalogs:
    ID:
      groups:
        TransactionDetails:
          kind: triple-list
          fields:
            ClntNm:
              kind: triple
              path: []
              column: client_name          # omit if column does not exist (LLD: derive on read)
            TrnTyp:
              kind: triple
              path: []
              column: trn_typ
            TrnRef:
              kind: triple
              path: []
              column: transaction_ref      # omit if column not in DDL
            BankChg: { kind: triple, path: [] }
            Multi: { kind: triple, path: [] }
            NonS2B: { kind: triple, path: [] }
            ScaRef: { kind: triple, path: [] }
            ValDt: { kind: triple, path: [] }
        DebitDetails:
          fields:
            DrAccNo:
              kind: triple
              path: [{ key: details, repeating: true }]
              triples-at: data
            DrAmt:
              kind: triple
              path: [{ key: details, repeating: true }]
              triples-at: data
              column: debit_amount
              occurrence: 0
            DrAccCcy: { kind: triple, path: [{ key: details, repeating: true }], triples-at: data }
            DrAccNm: { kind: triple, path: [{ key: details, repeating: true }], triples-at: data }
            DrNarrLn1: { kind: triple, path: [{ key: details, repeating: true }], triples-at: data }
            Transaction-Id: { kind: triple, path: [{ key: details, repeating: true }], triples-at: data }
            rate:
              kind: scalar
              path: [{ key: fx }]
        CreditDetails:
          fields:
            CrAccNo: { kind: triple, path: [{ key: details, repeating: true }], triples-at: data }
            CrAmt: { kind: triple, path: [{ key: details, repeating: true }], triples-at: data }
            CrAccCcy: { kind: triple, path: [{ key: details, repeating: true }], triples-at: data }
            CrAccNm: { kind: triple, path: [{ key: details, repeating: true }], triples-at: data }
            CrNarrLn1: { kind: triple, path: [{ key: details, repeating: true }], triples-at: data }
            Transaction-Id: { kind: triple, path: [{ key: details, repeating: true }], triples-at: data }
```

Walker:

1. `node = initiationDetail[fieldGroup]`
2. For each catalog/request `path` segment: object → `node[key]`; array → `node[key][index]`
3. `kind=triple` → find `{ Name == fieldName }` in `triples-at` (or in `node` if `kind: triple-list`); set **`Value` only**
4. `kind=scalar` → set `node[fieldName] = fieldValue`
5. Else `400`

A country without `details`, e.g. `Debit.lines[]` of triples on the item:

```yaml
MY:
  groups:
    Debit:
      fields:
        AccNo:
          kind: triple
          path: [{ key: lines, repeating: true }]
          triples-at: "."
```

Same Java. Request: `fieldGroup: Debit`, `path: [{ "key": "lines", "index": 0 }]`, `fieldName: AccNo`.

---

## 5. Ingest table projections

`extracted_data` is the source of truth. Columns are copies for list/audit **without** re-parsing JSON on every read.

The **request never names columns**. After a real value change, `IngestDetailProjector` runs for catalog entries that have `column:`.

| JSON field (ID) | Typical column | Notes |
|-----------------|----------------|-------|
| `TrnTyp` | `trn_typ` | LLD: only promoted column; keep even if you drop the others |
| `ClntNm` | `client_name` | Only if the entity has this column |
| `TrnRef` or `TxRef` | `transaction_ref` | Map **one** source; do not map `DrAccNo` here |
| `DrAmt` | `debit_amount` | Requires `occurrence: 0` |

**Same projector** on `ExtractionTriggerHandler` insert and on field save. Two writers cause drift.

**Do not** re-parse the whole JSON to refill every column on every save. Only columns whose **changed** fields declare `column:`.

**Do not** promote a repeating field without `occurrence`. Editing `details[1].DrAmt` must not overwrite `debit_amount` unless that is the defined rule.

Physical setter map is **fixed** to columns that exist on `PaymentDataIngestDetailsEntity`. Catalog may only reference those keys. A new DB column is a migration; mapping `CustNm` → existing `client_name` is catalog-only.

If you follow LLD strictly (no `client_name` / `transaction_ref` columns): omit those `column:` entries; `getDetail` still derives `clientName` / `txRef` from JSON.

---

## 6. Save transaction (mandatory order)

One `@Transactional` per batch:

1. Load details; verify `upload_id`, `READY_FOR_REVIEW`, `@Version`.
2. Catalog-validate every address; apply edits via walker (`Value` / scalar only).
3. **Gate A — shape:** `ExtractedDataCodec.fromJson` / POJO round-trip of the merged document. Fail → rollback.
4. Diff vs stored JSON; skip no-ops (no audit, no projection).
5. Projector for changed fields with `column:`.
6. **Gate B — downstream** (Submit always; Save draft only if product requires handoff-ready drafts): `EntityPaymentMapper.toDownstream(stored, meta)` then Bean Validation. **No** `PaymentsCoreClient` / value-date enrichment here — that stays in initialize/handoff. Fail → rollback. Map violations back to `{ fieldGroup, path, fieldName }` (never DTO property names).
7. Persist `extracted_data` + columns + `updated_by` / `updated_at`.
8. `fss_payment_field_audit` (one row per real change) + one `FIELD_EDIT` action if any change.
9. Return summaries + `changedFieldCount`.

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

New country: implement mapper + catalog. Edit service unchanged.

Suggested error body:

```json
{
  "errorCode": "DOWNSTREAM_MAPPING_FAILED",
  "violations": [
    {
      "fieldGroup": "DebitDetails",
      "path": [{ "key": "details", "index": 0 }],
      "fieldName": "DrAccNo",
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
| `field_group` | Unchanged — first-level JSON key |
| `occurrence_index` | **Remove** from new DDL (or stop writing). Do not keep in parallel with path. |
| `field_path` | **Add** `JSONB` or `TEXT` — the request `path[]` as stored (`[]`, `[{"key":"details","index":0}]`, `[{"key":"fx"}]`) |
| `field_name` | Unchanged |
| `old_value` / `new_value` | Unchanged; server-computed |

If `occurrence_index` already exists in a deployed V6: additive `field_path` + write both during dual-run is acceptable **only** if `occurrence_index` = first repeating `path[].index` or null. Prefer one cutover in the same release as the API change if the table is not yet in production.

---

## 9. Code layout (patch existing LLD §9.1)

| Type | Name | Action |
|------|------|--------|
| Service | `ExtractionFieldEditService` | Keep — batch, lock, diff, audit, persist. **No** `DebitDetails` / `details` literals |
| Component | `CatalogExtractedDataFieldAccessor` | **New (or replace)** `IdExtractedDataFieldAccessor` — `list()` for GET, `setValue()` for POST |
| DTO | `EditableFieldDto` | Shared by GET `ingestDetails[].fields[]` and POST body/response (POST omits `confidence` / `kind` on input) |
| Config | `FieldCatalogProperties` | Bind `extraction.field-catalogs` |
| Component | `IngestDetailProjector` | **New** — catalog `column:` → entity setters; used by trigger handler + edit service |
| Interface | `EntityPaymentMapper` | **Add** `toDownstream` / `validate`; ID impl maps to the real IAP type found in scan |
| DTO | `FieldEditRequest` | Replace `occurrenceIndex` with `List<PathSegment> path` |
| Exception | `DownstreamMappingException` | **New** — 400 `DOWNSTREAM_MAPPING_FAILED` |
| Flyway/SQL | field audit | `field_path`; drop or stop `occurrence_index` |

**Do not** add `IdFieldEditService` / `MyFieldEditService`.

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

**Do not keep `occurrenceIndex` on the API “for ID” and add `path` later.** ID uses `path: [{ key: details, index: n }]`.

**Do not walk `extractedData` on the client to build `path[]`.** That re-implements the catalog in the portal. Bind the grid to GET `fields[]` and echo coordinates.

**Do not POST the GET list’s `extractedData` blobs.** Save is field assignments, not a document replace.

**Do not key a grid cell by `fieldName` alone.** Two debit legs share `DrAccNo`; the key is `(fieldGroup, path, fieldName)`.

---

## 11. Test matrix (minimum)

| # | Case | Expect |
|---|------|--------|
| 1 | `ClntNm` save | JSON `Value` changed; `Confidence` unchanged; `client_name` updated iff catalog has `column`; audit 1 row |
| 2 | `DrAccNo` `path index 0` | Triple under `details[0].data`; no `trn_typ` write |
| 3 | `DrAmt` `index 1` with `occurrence: 0` on column | JSON updated; `debit_amount` unchanged |
| 4 | `path: [{ key: fx }]` `rate` | Scalar set; 400 if `fx` absent and not in catalog |
| 5 | Replay identical save | `changedFieldCount: 0`; no audit |
| 6 | Merge that fails Jackson/`StoredExtractedData` | 400; no persist |
| 7 | Submit with blank `DrAccNo` | `DOWNSTREAM_MAPPING_FAILED`; no persist |
| 8 | `detailId` from another upload | 403; no persist |
| 9 | Stale `@Version` | 409; no persist |
| 10 | MY catalog `lines` without new Java walker | AccNo edit works if YAML registered |
| 11 | `getDetail` vs fixture | `fields[]` contains ClntNm `path:[]` and DrAccNo `path:[{details,0}]`; no header fields |
| 12 | Save echoes GET addresses only | Round-trip: GET → change one `fieldValue` → POST → GET (or POST `fields[]`) matches |
| 13 | Two credit legs | Two `CrAccNo` entries differ only by `path[].index` |

---

## 12. LLD / API sections this document patches

When applying to the pack (after repo scan Deliverable E):

| Location | Change |
|----------|--------|
| API §2.4 | `ingestDetails[].fields[]` flattened from catalog; grid contract; `version` on the row |
| API §2.5 | `occurrenceIndex` → `path[]`; POST is echo of GET addresses; response includes `fields[]` |
| LLD §5.5 | Walker `list`/`setValue` + projector + gates A/B |
| LLD §5.1 field audit | `field_path`; drop `occurrence_index` |
| LLD §6.5 | Addressing relative to `initiationDetail` via `path` |
| LLD §9.1 / §9.3 | Catalog accessor, projector, mapper validate |
| LLD §4.1 new-entity checklist | Catalog YAML instead of a new `*ExtractedDataFieldAccessor` (unless value model differs) |
| UX field grids | Bind to GET `fields[]`; Save draft filters dirty cells; do not parse `extractedData` |

---

## 13. Implementation order

0. Repo scan ([IDP_Field_Edit_REPO_SCAN_PROMPT.md](./IDP_Field_Edit_REPO_SCAN_PROMPT.md)); resolve Blockers; apply Deliverable E into this file if the real DTO/column names differ.  
1. DDL / entity: `field_path`; confirm which ingest columns exist.  
2. Catalog properties + ID YAML from `after_ocr-llm-output.json`.  
3. `CatalogExtractedDataFieldAccessor.list` + `.setValue` + tests against the fixture.  
4. `getDetail` assembler: populate `ingestDetails[].fields[]` via `list()` (same as save).  
5. `FieldEditRequest` / `EditableFieldDto` + controller.  
6. Projector + wire into extract insert and save.  
7. Gate A in edit service.  
8. Gate B via existing mapper type found in scan (Submit first).  
9. Portal: grid = GET `fields[]`; Save draft echoes dirty `{ fieldGroup, path, fieldName, fieldValue }`.  
10. Remove `occurrenceIndex` from API if it already shipped in this workspace.
