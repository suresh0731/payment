# Repo scan prompt — FSS Payments Document Ingestion

Use this prompt in a workspace that contains the real Java repos (`51786-payment-gateway-service`, `51786-workflow-management`, `51786-idp-extraction-service`, etc.).

---

## Prompt (copy below this line)

```
You are implementing FSS Payments Document Ingestion per the specification in `final/IDP_LLD.md` (v8 implementation guide).

**Before writing any new code**, scan the repositories and produce a **Reuse & alignment report** with concrete file paths, class names, and code citations.

### 1. Scan targets (search each repo)

| What to find | Search hints | Used for |
|--------------|--------------|----------|
| External task handlers | `@ExternalTaskSubscription`, `ExternalTaskHandler`, `ExternalTaskHelper` | `ExtractionTriggerHandler`, `ExtractionPaymentHandoffHandler`, `IAPExtractionInitializeHandler` |
| Workflow client | `WorkflowService`, `startWorkflowProcess`, `setTaskDetails`, `completeCurrentTask`, `startMessageCorrelation` | Upload start, `performAction` façade, handoff |
| Message / payment persistence | `MessageService`, `fss_services_message`, `DOC_EXTRACTION`, `IAPIDPaymentsMessageHandler` | Handoff before correlation; one `message_id` per ingest detail row |
| Country/entity factories | `CountrySpecificProcessorFactory`, `CountryExcelMsgProcessorFactory`, `*Factory`, `*Registry` | `ExtractionPaymentRouteRegistry`, `EntityHandlerRegistry` pattern |
| IAP ID handlers | `IAPIDPaymentTransactionHandler`, `IAPPaymentBulkTransactionHandler`, `Initialize_IAP` | Package placement, handler base class |
| Bulk upload controller | `BulkUploadController`, `retryTransaction`, `@RequestParam` id style | `ExtractionUploadController` URL conventions (`getUploads`, `getDetail`, `performAction`) |
| Gateway action façade | `@PreAuthorize`, `paymentMaker`, action services with handler maps | `ExtractionUploadActionService`, `UploadActionHandler`, `SubmitUploadActionHandler` |
| Extraction service POJOs | `InitiationDetail`, `ExtractedField`, `IdPaymentOutput`, `POST /v1/extract` | Jackson models — **reuse, do not duplicate** |
| Jackson config | `ObjectMapper`, `@JsonProperty`, existing DTO packages | `ExtractedDataCodec` / deserialization of `extracted_data` CLOB |
| JPA entities/repos | `*Entity`, `*Repository` in payment-gateway | New `ingestion` package — **three tables**: meta, content, ingest details |
| Manual DDL / migrations | `Liquibase`, `Flyway`, `manual-ddl`, `resources/sql` | LLD uses **manual SQL only** — confirm no migration tool in gateway |
| application.yml routing | `@ConfigurationProperties`, list-of-objects config | `extraction.payment-routes`, `extraction.entity-templates`, `extraction.upload.zip`, `extraction.upload.password` |
| ZIP / PDF libraries | `zip4j`, `PDFBox`, `PDDocument`, `ZipFile`, archive password handling | `ZipArchiveExtractor`, `PdfDecryptor` — or existing equivalents to reuse |
| File upload utilities | ZIP unpack, encrypted PDF, multipart upload handlers | Avoid duplicating if gateway already has similar helpers |
| Camunda BPMN deploy path | `src/main/resources/*.bpmn`, `IAP_ID_Payments` | Deploy `FSSPaymentsDocIngestion.bpmn`; additive `IAP_ID_Extraction_Trigger` |
| Portal UI (optional) | `51786-transaction-data-tiles`, landing tabs, modal patterns | `InstructionUploadTab`, `UploadDetailModal` — see `final/IDP_UX_Design.md` |

### 2. Report format

For each row in IDP_LLD.md **§9.3** (interface & class inventory), output:

```markdown
### <ClassName>
- **Reuse:** [existing class path] OR **New**
- **Pattern match:** [similar class] — cite 5–15 lines showing the pattern to copy
- **Package:** recommended package based on neighbors
- **Gaps:** anything the LLD assumes that does not exist in repo
```

Include explicit rows for v8 additions even if not in §9.3 table: `ZipArchiveExtractor`, `PdfDecryptor`, `UploadEntryFilter`, `SubmitUploadActionHandler`, `CancelUploadActionHandler`, `EncryptedFileException`, `InvalidPasswordException`.

### 3. Validation checks

- Confirm **`entity` is the only routing key** on new DDL (`fss_payment_upload_meta.entity`) — no separate `country` column. Note if extraction HTTP client must map `entity` → `country` in metadata (adapter only).
- Confirm extraction service returns **`initiationDetail[]`** (array) and which Jackson types deserialize it; locate fixture `after_ocr-llm-output.json` or service test fixtures.
- Confirm **one `fss_payment_data_ingest_details` row per instruction** — scan for any legacy single-row-per-upload assumptions in gateway or UI.
- Confirm **gateway façade** for user review: portal must **not** call `workflow-management` directly for Submit/Cancel — find existing `WorkflowService` client usage from gateway services.
- List Java version in parent POM (8 / 11 / 17) and which modern features are already used (`record`, `Optional`, `var`, `Map.of`, sealed interfaces, etc.).
- Identify the **base class or helper** used by existing IAP external task handlers (retries, complete/fail, variable read).
- Confirm **no Liquibase/Flyway** in gateway (or document exception if found).
- Check parent POM / BOM for **zip4j** and **pdfbox** — reuse managed versions if present.
- Password security: confirm no existing code persists upload passwords in DB, audit, logs, or Camunda variables.

### 4. API / UX alignment (read-only cross-check)

Cross-check against `final/IDP_API_Reference.md` and `final/IDP_UX_Design.md` — no UI implementation, but note gaps for backend:

- `getUploads` should support file-level rows with `instructionCount` and overall document `confidence` from meta (not computed on poll).
- `getDetail` returns flat `ingestDetails[]` — UI groups by `subActivityId` / `trnTyp` client-side; no grouped API required unless repo already has a grouping pattern to reuse.
- Query param standard: `uploadId` (alias `extractionUploadId` optional).

### 5. Output deliverables

1. Reuse & alignment report (markdown)
2. Updated package tree proposal aligned to real repo structure
3. List of LLD sections that need correction after scan (if any)
4. Do **not** implement yet unless I ask — discovery only

Reference spec: `final/IDP_LLD.md` sections 2.4, 4, 5, 6, 7, 9 (incl. §9.8 ZIP/password), 13.
Companion docs: `final/IDP_API_Reference.md`, `final/IDP_UX_Design.md`, `final/FSSPaymentsDocIngestion.bpmn`.
```

---

## After the scan

1. Paste the **Reuse & alignment report** back into `IDP_LLD.md` §13.2 (or attach as `IDP_LLD_REPO_FINDINGS.md`).
2. Update **§9.3** class inventory with real package paths from the scan.
3. Implement following §14 order: config → entities/repos → POJOs → ZIP/PDF extractors → registry → handlers → controller → E2E tests (§12).
