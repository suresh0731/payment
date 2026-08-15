# FSS Payments Document Ingestion — Final Documentation Pack

> **Status:** Final — consolidated for implementation and review  
> **Created:** 2026-08-02  
> **Updated:** 2026-08-10 — gateway owns `header`; re-extract dropped (`FAILED` terminal); confidence at three payload scopes; file status as a fold over instruction statuses. Applied across all four documents  
> **Earlier:** 2026-08-04 (v4 — FSS rename, ZIP bulk upload, multi-instruction fan-out; v6 — gateway façade for maker/checker actions)  
> **Supersedes:** `draft/` working copies and `IDP_LLD_latest.md` (obsolete — safe to delete). Use **`final/`** as source of truth.

---

## Canonical documents — safe to delete

**For implementation, use only the files in `final/` listed below.** Everything else in this repo for this feature is historical.

### Use these (source of truth)

| Purpose | **Canonical file** | Notes |
|---------|-------------------|--------|
| **Low Level Design (backend/DB)** | **[IDP_LLD.md](./IDP_LLD.md)** | Schema, handlers, gateway façade, DDL, tests |
| Architecture & BPMN | [IDP_Document_Ingestion_Design.md](./IDP_Document_Ingestion_Design.md) | Two-BPMN strategy, ZIP, multi-instruction handoff |
| REST APIs | [IDP_API_Reference.md](./IDP_API_Reference.md) | Upload + user review action endpoints; **`POST /fields` addressing superseded by Field Edit Design** |
| Field edit patch | [IDP_Field_Edit_Design.md](./IDP_Field_Edit_Design.md) | Field spec YAML, `{ fieldGroup, fieldName, occurrenceIndex }`, projections, Gate A/B — patch this onto existing `fields` code |
| Portal UX | [IDP_UX_Design.md](./IDP_UX_Design.md) | Tabs, table, modal, button → API map |
| BPMN deploy artifact | [FSSPaymentsDocIngestion.bpmn](./FSSPaymentsDocIngestion.bpmn) | Process id `FSS_Payments_Document_Ingestion` |
| Start here | [README.md](./README.md) | Index and reading order |

### Safe to delete (obsolete)

| Path | Superseded by |
|------|----------------|
| **`final/IDP_LLD_latest.md`** | **`final/IDP_LLD.md`** |
| **`draft/`** (entire folder) | **`final/`** |

---

## Document set

| # | Document | Audience | Purpose |
|---|----------|----------|---------|
| 1 | [IDP_Document_Ingestion_Design.md](./IDP_Document_Ingestion_Design.md) | Architects, tech leads, BPMN reviewers | Production-safe two-BPMN strategy, ZIP upload, multi-instruction handoff, deployment & regression plan |
| 2 | [IDP_LLD.md](./IDP_LLD.md) | Backend, DB, integration engineers | **Implementation guide** — patterns, interfaces, Camunda handlers, DDL |
| 3 | [IDP_UX_Design.md](./IDP_UX_Design.md) | Frontend, UX, product | Landing-page tabs, upload table, detail modal, role-based actions |
| 4 | [IDP_API_Reference.md](./IDP_API_Reference.md) | Frontend, backend, integration | **All REST APIs** — request/response shapes, UI action map, extraction service contract |
| 5 | [IDP_Field_Edit_Design.md](./IDP_Field_Edit_Design.md) | Backend, frontend | **Patch spec** — field-spec walker (`extraction.field-specs`), wire `{ fieldGroup, fieldName, occurrenceIndex }` (client never sends `path`), ingest-column projections, downstream DTO validation before persist. Patches API §2.5 / LLD §5.5 |

### BPMN artifact (in this folder)

| File | Description |
|------|-------------|
| [FSSPaymentsDocIngestion.bpmn](./FSSPaymentsDocIngestion.bpmn) | Common country-agnostic document ingestion process — deploy to `51786-workflow-management` |

### Related repo artifacts (outside `final/`)

| Path | Description |
|------|-------------|
| [IDP_LLD_REPO_SCAN_PROMPT.md](./IDP_LLD_REPO_SCAN_PROMPT.md) | **Run in active code workspace** before ingestion implementation — discovers reusable classes |
| [IDP_Field_Edit_REPO_SCAN_PROMPT.md](./IDP_Field_Edit_REPO_SCAN_PROMPT.md) | **Run before the field-edit patch** — grounds field spec / projector / Gate B in real DTOs and columns |
| [../ID_payments.bpmn](../ID_payments.bpmn) | Indonesia payment process — **additive** change: `IAP_ID_Extraction_Trigger` + `Initialize_IAP_From_Extraction` |
| [../ID_payments.md](../ID_payments.md) | Existing IAP_ID_Payments workflow reference |
| [../after_ocr-llm-output.json](../after_ocr-llm-output.json) | Reference extraction JSON (single instruction; production uses `initiationDetail[]`) |
| [../51786-idp-extraction-service/](../51786-idp-extraction-service/) | Built extraction service (`POST /v1/extract`, mock mode) |

---

## Recommended reading order

0. **Repo scan** — [IDP_LLD_REPO_SCAN_PROMPT.md](./IDP_LLD_REPO_SCAN_PROMPT.md) in active code workspace; fill LLD §13.2
1. **Design** §1–§2 — decisions and architecture
2. **UX** §2–§5 — what users see and do (PDF + ZIP upload, multi-instruction modal)
3. **LLD** §2–§6 — services, BPMN task map, DDL, structured output contract
4. **[API Reference](./IDP_API_Reference.md)** — endpoints, payloads, UI → API map
5. **LLD** §4 — `ExtractionPaymentRouteRegistry` (entity handoff)
6. **Design** §14–§16 — regression matrix and implementation checklist
7. **Field-edit patch** — [IDP_Field_Edit_Design.md](./IDP_Field_Edit_Design.md) after [IDP_Field_Edit_REPO_SCAN_PROMPT.md](./IDP_Field_Edit_REPO_SCAN_PROMPT.md) (if Save draft / `POST /fields` is in scope)

---

## Architecture at a glance

```mermaid
flowchart LR
    UI["Portal — Instruction Upload tab"]
    GW["payment-gateway-service"]
    EXT["idp-extraction-service"]
    WFM["Camunda"]
    PAY["Entity payment BPMN e.g. IAP_ID_Payments"]

    UI -->|POST extraction-uploads PDF/ZIP| GW
    GW -->|startWorkflow per PDF| WFM
    WFM -->|Trigger_Data_Extraction| GW
    GW -->|POST /v1/extract sync| EXT
    EXT -->|initiationDetail[] only| GW
    GW -->|merge header + complete task| WFM
    UI -->|performAction SUBMIT/CANCEL| GW
    GW -->|setTaskDetails + completeCurrentTask| WFM
    WFM -->|Trigger_Payment_From_Extraction| GW
    GW -->|N × startMessageCorrelation per instruction| WFM
    WFM --> PAY
```

### Key design rules

| Rule | Detail |
|------|--------|
| One ingestion BPMN for all countries | `FSS_Payments_Document_Ingestion` — upload, OCR/LLM, single user review |
| Entity routing **not** in ingestion BPMN | `Trigger_Payment_From_Extraction` fans out via `ExtractionPaymentRouteRegistry` keyed by `entity` |
| Entry points in entity payment BPMNs | e.g. `IAP_ID_Extraction_Trigger` on `IAP_ID_Payments.bpmn` when `entity=ID` |
| ZIP bulk upload | One upload row + workflow **per PDF** inside the archive; non-PDF entries ignored |
| File ID | `YYENTYXXXXX` per identified file — year + Level-1 access code (`IDJY` for Indonesia) + sequence scoped to that year and code; files sorted by upload timestamp ascending before allocation. Sequence is padded to 5 digits and grows beyond it rather than running out |
| Multi-instruction PDF | Extraction returns `initiationDetail[]` and nothing else; the **gateway builds `header`** and merges it onto each row; **one payment trigger per instruction** |
| Retry after a failed extraction | **Re-upload, not re-extract.** `FAILED` is terminal; there is no re-extract endpoint, button, or status edge. A protected document needs its password re-entered, since none is stored |
| Confidence | Three scopes, all from the extraction payload — file, instruction, field. The file score is **not** an average of the instruction scores |
| Status | `fss_payment_upload_meta.status` is a **fold** over its instruction rows (`FAILED` ranked first), written only by `ExtractionUploadAggregateService`. Instruction rows never read `UPLOADED` or `PROCESSING` |
| Kill-switch | One key: **`extraction.upload.enabled`**, read at startup. Not runtime-toggleable |
| Extraction service is domain-agnostic | No Camunda, no payment tables — gateway owns workflow |
| User review actions via gateway façade | Portal calls `POST .../performAction` (`SUBMIT` / `CANCEL`); optional `/submit`/`/cancel` aliases — gateway updates DB + `fss_payment_upload_audit`, then WFM server-side |
| Phase 1 = sync extract | Gateway blocks on `POST /v1/extract` (~10 min budget, 15 min HTTP envelope) |
| Phase 1 entity | Indonesia (`entity=ID`) — registry row only |
| Database | **PostgreSQL** — manual `.sql` DDL, no Liquibase/Flyway; application code stays JPA/JPQL-only so nothing is dialect-bound |
| Identifiers | Surrogate keys are native `UUID` columns holding **UUIDv7** (time-ordered, non-enumerable). Keys owned by other systems — `message_id`, `payment_workflow_key`, `extraction_id` — keep their existing types |

---

## Naming change (v3/v4)

| Legacy (v1) | Current (v4) |
|-------------|----------------|
| `IDP_Document_Ingestion` | `FSS_Payments_Document_Ingestion` |
| `IDP_Document_Ingestion.bpmn` | `FSSPaymentsDocIngestion.bpmn` |
| `fss_idp_*` / `fss_doc_extract_*` | `fss_payment_upload_*` + `fss_payment_data_ingest_details` |
| `IAP_ID_IDP_Trigger` | `IAP_ID_Extraction_Trigger` |
| `Initialize_IAP_From_IDP` | `Initialize_IAP_From_Extraction` |
| `channel=IDP_UPLOAD` | `channel=DOC_EXTRACTION` |
| `country` (upload routing) | **`entity`** — same value, single field (phase 1: `ID`) |
| BPMN message wait for extraction | `ExtractionResultGateway` on task completion |

---

## Phase 1 scope summary

| In scope | Out of scope (v1) |
|----------|-------------------|
| ID landing tab UX (upload + table + modal) | New sidebar nav route |
| Single PDF + ZIP bulk upload (PDFs only) | Non-PDF file types as direct upload |
| Multi-instruction PDF → N payment instances | China ETF handoff (registry stub only) |
| `fss_payment_upload_*` + `fss_payment_data_ingest_details` + `fss_payment_upload_audit` (`file_id` on meta; `retry`, `error_desc` on details) | `document-service` integration |
| `51786-idp-extraction-service` | SSTM feedback for `DOC_EXTRACTION` channel |
| Additive `IAP_ID_Payments.bpmn` change | |

---

## Implementation status (high level)

| Component | Status |
|-----------|--------|
| `51786-idp-extraction-service` | Built (sync `/v1/extract`, mock mode, `id-payment` template) — **one change pending:** `initiationDetail` must become an array. Blocks the gateway work that depends on it |
| `FSSPaymentsDocIngestion.bpmn` | In `final/` — ready for workflow-management |
| `IAP_ID_Payments.bpmn` additive diff | In repo root `ID_payments.bpmn` |
| `51786-payment-gateway-service` | Planned — upload APIs + `ExtractionUploadActionService` (user review façade) documented in LLD |
| Portal UI | Specified in UX doc — not built |

---

## Document history

| Date | Change |
|------|--------|
| 2026-07-30 | Initial draft pack in `draft/` |
| 2026-08-02 | Routing responsibilities, registry model, sync extract, timeouts |
| 2026-08-02 | **Final pack** — consolidated into `final/` with README index |
| 2026-08-02 | Added [IDP_API_Reference.md](./IDP_API_Reference.md) |
| 2026-08-05 | **v7** — Entity-only routing synced across API Reference + Design + UX |
| 2026-08-04 | **v5** — Three-table schema; `retry` + `error_desc` on ingest details |
| 2026-08-09 | Added timestamp-ordered File ID generation across design, LLD, API, and UX |
| 2026-08-09 | File ID finalized as `YYENTYXXXXX` — year-scoped, overflow-safe sequence; counter persisted in `fss_payment_file_seq` |
| 2026-08-09 | **Target database is PostgreSQL** — all DDL converted from Oracle types (`BYTEA`, `TEXT`, `TIMESTAMPTZ`, `SMALLINT`/`BIGINT`, `NUMERIC`, identity PK); added Postgres-specific guidance on `@Lob`, isolation level, TOAST, and prefix-index collation |
| 2026-08-09 | Surrogate keys moved from `VARCHAR(36)` to native `UUID` holding **UUIDv7** — time-ordered index locality without enumerable IDs in the API; dropped `extraction_workflow_key` (always equalled `upload_id`) |
| 2026-08-09 | LLD implementation guide hardened — Java-17 sample baseline plus an explicit 8/11 downgrade table, constructor-bound validated config, one generic `AbstractKeyedRegistry` replacing three copy-pasted registries, injected `Clock`, statelessness and JPA entity conventions, and correlation/observability rules |
| 2026-08-09 | Added §7.0 runtime budget for the 10-minute synchronous extraction call (lock duration, timeouts, connection pool), scheduler locking for multi-instance deployments, and File ID steps to the §14 build order; repo scan prompt rewritten to also produce a risk register and a ticket-ready implementation backlog |
| 2026-08-09 | File ID allocator simplified to a single atomic `INSERT … ON CONFLICT … RETURNING` now that PostgreSQL is confirmed — five classes and the retry loop removed, portability kept at the `FileIdSequenceAllocator` interface; `block-size` and `max-allocation-attempts` config dropped |
| 2026-08-09 | `ZipArchiveExtractor` reworked — switched to zip4j `ZipInputStream` (the documented `ZipFile(InputStream)` does not exist and would have forced temp files), now returns extracted **and** skipped entries so the API's `skippedEntries[]` can actually be populated, enforces the ZIP-bomb and file-count limits while streaming, and separates a wrong password from a corrupt archive |
| 2026-08-10 | **`POST /fields` reshaped from per-instruction to batched.** The endpoint took a single `detailId` while **Save draft** is a file-level button, so a maker editing several instructions and pressing one button silently persisted only the last selected row. `detailId` moved into the body as `instructions[]`, the call is atomic, the response returns refreshed per-instruction summaries, and every `detailId` is verified to belong to the `uploadId`. Synced across API Reference, LLD, UX, and Design |
| 2026-08-10 | **Field-level audit added — `fss_payment_field_audit` (`V6`).** Maker edits to machine-extracted values were previously unaudited: no field name, no before-value, only `updated_by`/`updated_at` on the row. New append-only table records one row per *changed* field with `old_value` / `new_value`, keyed on `detail_id` alone, alongside one `FIELD_EDIT` action row per save. Diff is computed server-side, no-ops write nothing, `Confidence` is never overwritten (LLD §5.5). `extraction_id` was added to keep the trail readable across re-extracts; with re-extract dropped there is one extraction per upload, so the column is now invariant per upload and **redundant** — keep it only as forward-compatibility, and document it as invariant if you do (LLD §5.1). No `upload_id` (the detail-row join is needed anyway to render the instruction) and no `old_confidence` (already retained in `extracted_data`) |
| 2026-08-10 | **Review-edit addressing made entity-agnostic.** `section` / `legIndex` encoded `id-payment`'s structure — `TRANSACTION`/`DEBIT`/`CREDIT` and the word *leg* are Indonesia's payload shape, not a universal one, so a second country would have forced a change to the API contract **and** the audit schema. Replaced by `fieldGroup` (the verbatim JSON container key) + `occurrenceIndex` (position within a repeating group), with legal values supplied per entity by a new `FieldGroupCatalog` and `ExtractedDataFieldAccessor` resolved through `EntityFieldAccessorRegistry`. A new country now ships a catalog and an accessor; the `fields` contract and `fss_payment_field_audit` stay fixed. Header paths remain unreachable because resolution is rooted at the `initiationDetail` node |
| 2026-08-14 | **Field-edit patch pack.** Nested objects (`DebitDetails.fx.rate`) live in YAML `path`, not on the HTTP wire. [IDP_Field_Edit_Design.md](./IDP_Field_Edit_Design.md) locks per-entity **field spec** (`extraction.field-specs`, `EntityFieldSpec`, `FieldSpecAccessor`) + `{ fieldGroup, fieldName, occurrenceIndex }` + ingest-column projector + shape/downstream validation. Companion [IDP_Field_Edit_REPO_SCAN_PROMPT.md](./IDP_Field_Edit_REPO_SCAN_PROMPT.md) is a discovery-only scan before the patch. |
| 2026-08-10 | **Four contract decisions applied across the whole pack**, each reversing an earlier position. **(1) The gateway owns `header`;** the extraction service returns `initiationDetail` as an array and nothing else, so `TotInst` is computed from `size()` and the compare-and-warn reconciliation rule is retired. **(2) `initiationDetail` becomes an array in the built extraction service** rather than the LLD de-scoping to one instruction per file — the single-object shape capped every file at one payment and made the normal multi-instruction case silently lossy. **(3) Re-extract is removed** — no endpoint, no button, no status edge, no `RE_EXTRACT` audit action; `FAILED` is terminal and retry is a re-upload, which also means a protected document needs its password re-entered. **(4) Confidence is a real payload value at three scopes** (file, instruction, field), so `AVG(confidence_score)` is a fallback rather than the rule. Also settled: `fss_payment_upload_meta.status` is a fold over instruction statuses with one writer, instruction rows never hold `UPLOADED`/`PROCESSING`, the kill-switch has one spelling (`extraction.upload.enabled`), `getDetail` takes `uploadId` like every other endpoint, and the error-code catalog is exhaustive with a per-endpoint authorization matrix and entity-scoped `404`s. Synced across LLD, API Reference, UX, Design, and the repo scan prompt |
