# Repo scan prompt — Field edit patch (field spec, projections, downstream validation)

Run this in a workspace that contains the **implemented** ingestion code (`51786-payment-gateway-service` and related repos), not only `final/` markdown.

It produces a **reuse report**, a **gap vs this patch**, a **risk register**, and a **ticket-ready backlog** so implementation is a surgical patch — not a second design.

Discovery only — **no production code** in this pass.

Copy everything between the markers.

---

=== BEGIN PROMPT ===

You are preparing to **patch** the existing Document Ingestion field-edit path so it matches `final/IDP_Field_Edit_Design.md` (GET flattened `fields[]`, `{ fieldGroup, fieldName, occurrenceIndex }` addressing, field-spec YAML walker, ingest-column projections, downstream DTO validation before persist).

Canonical companions (do not re-litigate ingestion as a whole):

- `final/IDP_Field_Edit_Design.md` — **this patch is source of truth** for GET `fields[]`, field spec (`extraction.field-specs`), projections, gates A–B. Client never sends `path`.
- `final/IDP_LLD.md` — existing services, DDL, `ExtractionFieldEditService`, audit
- `final/IDP_API_Reference.md` §2.4 and §2.5 — current `getDetail` / `fields` contract
- `final/IDP_UX_Design.md` — Save draft UX; whether the modal already walks `extractedData`
- Fixture: `after_ocr-llm-output.json` (and any multi-instruction fixture you find)

**Do not write or modify production code in this pass.** Ground the patch in what exists. Where the repo contradicts the field-edit design, the **repo wins for names/types/columns**; record the contradiction so the design file can be edited before coding. Where the repo still uses `occurrenceIndex`, that is expected — **keep it** for repeating YAML paths. Reject a client-supplied `path`.

## 0. How to work

- **Evidence over inference.** Every claim cites `path/to/File.java:120-134` or `*.sql`. If missing: `NOT FOUND` and where you looked.
- **Mark uncertainty** as `UNKNOWN — needs <who/what>`.
- **Breadth first:** finish §1 inventory before deep-reading one class.
- **Missing repo:** list under *Not scanned*; do not invent it from the LLD.
- Do **not** expand scope to re-extract, header ownership, or BPMN. Field edit + ingest projections + mapper validation only.

## 1. Scan targets

### 1.1 Field-edit stack (must find)

| What | Search hints | Why |
|------|----------------|-----|
| Fields endpoint | `extraction-uploads/fields`, `FieldEdit`, `ExtractionFieldEdit` | Confirm batched `instructions[]` vs old per-`detailId` query param |
| getDetail assembler | `getDetail`, `IngestDetailSummaryDto`, `extractedData` | Must add `fields[]` via accessor `list()`; confirm UI today parses triples from `extractedData` |
| Request DTO | `occurrenceIndex`, `fieldGroup`, `fieldName`, `fieldValue`, `section`, `legIndex` | Exact JSON the portal already sends |
| Accessor | `ExtractedDataFieldAccessor`, `IdExtractedDataFieldAccessor` | Whether `details[].data[]` is hardcoded |
| Field spec | `FieldGroupCatalog`, `IdFieldGroupCatalog`, `extraction.field-specs` | Java vs YAML; replace with `EntityFieldSpec` / `EntityFieldSpecRegistry` |
| Registry | `EntityFieldAccessorRegistry` | How entity is resolved on save |
| Codec | `ExtractedDataCodec`, `StoredExtractedData` | Gate A target type |
| Audit write | `ExtractionFieldAuditService`, `fss_payment_field_audit`, `FIELD_EDIT` | Columns including `occurrence_index` |
| Optimistic lock | `@Version` on ingest details; `CONCURRENT_EDIT` | Must keep |
| Status guard | `READY_FOR_REVIEW` on edit | Must keep |

### 1.2 Persistence — ingest details

| Check | Why |
|-------|-----|
| `PaymentDataIngestDetailsEntity` (or equivalent) — **every field** | Which columns exist: `trn_typ`, `transaction_ref`, `client_name`, `extracted_data`, `version` |
| DDL / Flyway / Liquibase / manual SQL for `fss_payment_data_ingest_details` and `fss_payment_field_audit` | Whether `occurrence_index` is already in prod-shaped scripts; type of `extracted_data` (`TEXT` vs `JSONB`) |
| `@Lob` on `extracted_data` | Postgres `oid` leak if present |
| Who writes `trn_typ` / `transaction_ref` today | Insert-only vs also on save — dual writers |
| How `getDetail` builds `clientName` / `txRef` | Derived vs column |

### 1.3 Downstream mapping (Gate B)

| What | Search hints | Why |
|------|----------------|-----|
| Exact DTO name | `IAPSSTMRequestDTO`, `SSTMIAPRequestDTO`, `IAPID*Request`, `Payment`, `PaymentData` | Design uses a placeholder; **report the real type** |
| `EntityPaymentMapper` / `IdEntityPaymentMapper` / IAP initialize handler mapping from extraction JSON | `mapToPayment`, `InitiationDetail`, `ExtractedField` | Gate B must call this, not a second mapper |
| `ValidationService` / `javax.validation` / `jakarta.validation` on that DTO | Reuse vs new `Validator` |
| Remote calls inside mapping | `PaymentsCoreClient.getValueDate`, HTTP | Must **not** run on Save draft — split enrich vs map |
| Empty `DrAccNm` / `CrAccNm` (`Confidence: "0"`) — does mapping or Bean Validation fail? | Decides Gate B on save vs submit only |

### 1.4 Portal

| What | Why |
|------|-----|
| Save draft request builder | Must echo GET `fields[]` addresses; find `occurrenceIndex` / `legIndex` / client-side JSON walk |
| Field grid model | Whether cells are keyed by `Name` only (breaks two legs) or by path |
| Whether portal parses `{ Name, Value, Confidence }` | If yes, that code is deleted in this patch — flag as a portal task |
| Error handling for `400` field errors | Can it bind `DOWNSTREAM_MAPPING_FAILED.violations[]`? |

### 1.5 Config and entity extension

| What | Why |
|------|-----|
| `extraction.*` in `application.yml` | Where to add `field-specs` |
| Existing `@ConfigurationProperties` records | Match constructor binding |
| Second entity besides `ID` | Whether a second accessor already exists |

### 1.6 Tests to extend (do not rewrite blindly)

Find tests for field edit, accessor, audit, ingest entity. List class names. Note if they assert `occurrenceIndex`.

## 2. Validation checks (patch-specific)

- Confirm **one** edit service (no `IdPaymentFieldUpdateService` that bypasses audit).
- Confirm save is **one transaction** with JSON + audit (or report if audit is after-commit).
- Confirm `header` is not writable today (denylist vs root-at-`initiationDetail`).
- Confirm whether `data[]` is addressed by `Name` or by array index in current code.
- List every `if ("DebitDetails")` / `.getDetails().get(i).getData()` in the edit path — these become field-spec YAML.
- Confirm Java / Hibernate / validation API versions only if they block records or `jakarta.validation`.

## 3. Deliverable A — Current vs target

Table:

| Area | Current (path:line) | Target (`IDP_Field_Edit_Design.md`) | Patch action |
|------|---------------------|--------------------------------------|--------------|
| getDetail | | `ingestDetails[].fields[]` via accessor `list()` | |
| Request DTO | | echo of GET `{ fieldGroup, fieldName, occurrenceIndex, fieldValue }` — no `path` | |
| Accessor | | `FieldSpecAccessor.list()` + `setValue()` | |
| Field spec | | YAML `extraction.field-specs.ID` | |
| Audit columns | | keep `occurrence_index`; optional server-computed `field_path` | |
| Ingest columns | | projector map | |
| Mapper validate | | Gate A / Gate B | |
| Portal grid | | bind GET `fields[]`; do not walk `extractedData` | |
| Portal save | | dirty filter of GET addresses | |

Then for each **new** type in design §9: Reuse | Extend | New, with package based on neighbours.

Fill a **column inventory**:

| DB column | On entity? | Written where today? | Field-spec `column:` candidate? | Notes |
|-----------|------------|----------------------|------------------------------|-------|

Fill a **DTO inventory**:

| Type | Package | Used at handoff? | Safe for Gate B (no I/O)? | Bean Validation annotations? |

## 4. Deliverable B — Risk register

| # | Finding | Evidence | Severity | Impact | Recommended action |
|---|---------|----------|----------|--------|--------------------|

- **Blocker** — cannot patch as written (DTO does not exist; edit not transactional; `extracted_data` is not the fixture shape).
- **Major** — design tweak (Bean Validation rejects empty account names; `occurrence_index` already in production).
- **Minor** — local rename.

Call out explicitly:

- Dual-write `occurrence_index` + `field_path` vs single cutover.
- Gate B on Save draft vs Submit given empty `CrAccNm`.
- Real class name replacing `IAPSSTMRequestDTO` in the design doc.

## 5. Deliverable C — Implementation backlog

Ticket-ready, **patch order** (respect design §13). Real file paths.

| ID | Task | Files to create / modify | Depends on | Size | Acceptance gate | Open questions |
|----|------|--------------------------|------------|------|-----------------|----------------|

Size: S (< ½ day), M (½–2 days), L (> 2 days). Split L.

Acceptance: named test, or “Save draft of `ClntNm` against fixture updates JSON Value and not Confidence”.

Include:

- Apply Deliverable E into `IDP_Field_Edit_Design.md` (DTO/column names) **before** Java.
- Keep `occurrenceIndex` on the wire for repeating YAML paths; reject client `path`.
- `GET .../details` `fields[]` and Save draft in the **same** release (portal must not walk `extractedData`).
- Projector wired on **insert and save**.
- Tests from design §11 (including GET flatten and GET→POST echo).

**Ready-to-start set** at the end.

## 6. Deliverable D — Package tree (delta only)

Show only packages/classes this patch adds or renames, against the real base package (e.g. `com.sc.fss.paymentgateway....`).

## 7. Deliverable E — Design/LLD/API corrections

`document § → what it says → what is actually true → replacement text`.

Must include:

- Exact downstream type name for Gate B.
- Exact ingest column names (or “do not project client_name”).
- Whether `FieldGroupCatalog` already exists and should be replaced by `EntityFieldSpec` / `FieldSpecAccessor`.
- Fixture vs stored `extracted_data` (header present? `initiationDetail` object vs array on the row).

## 8. Output rules

- Markdown; deliverables A–E in order.
- Lead with: what already matches the patch, Blocker/Major/Minor counts, the one thing that would make a naive patch fail (e.g. portal still on `occurrenceIndex`).
- **Do not implement** unless I explicitly ask after this scan.

=== END PROMPT ===

---

## After the scan

1. Apply **Deliverable E** to `IDP_Field_Edit_Design.md` (and API §2.5 / LLD §5.5 if those files are still the published contract).
2. Resolve every **Blocker**.
3. Implement **Deliverable C** in order. Keep `ExtractionFieldEditService` as the only persist path.
4. Do not put `path` on GET/POST. Portal stays on `{ fieldGroup, fieldName, occurrenceIndex }`; YAML holds the walk.
