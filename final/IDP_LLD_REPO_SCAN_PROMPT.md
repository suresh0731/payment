# Repo scan prompt — FSS Payments Document Ingestion

Run this in a workspace containing the real Java repos (`51786-payment-gateway-service`, `51786-workflow-management`, `51786-idp-extraction-service`, and the portal UI if available).

It produces three things: a **reuse report**, a **risk register**, and a **sequenced implementation backlog**. Discovery only — no code is written.

The prompt below is delimited by markers rather than a code fence so that the nested examples render correctly. Copy everything between them.

---

=== BEGIN PROMPT ===

You are preparing to implement **FSS Payments Document Ingestion**, specified in `final/IDP_LLD.md` (Final v13), with companions `final/IDP_API_Reference.md`, `final/IDP_Document_Ingestion_Design.md`, and `final/IDP_UX_Design.md`.

**Do not write or modify any production code in this pass.** Your job is to ground the specification in what actually exists, and to produce an execution plan that a developer can start on Monday without rediscovering anything.

## 0. How to work

- **Evidence over inference.** Every claim cites `path/to/File.java:120-134`. If you cannot find something, write `NOT FOUND` and say where you looked. Never describe a class, config key, or column you have not opened.
- **Mark uncertainty explicitly** as `UNKNOWN — needs <who/what>`. An honest gap is far more useful than a confident guess; a wrong "reuse this" costs more than a missing one.
- **Breadth first, then depth.** Locate every area in §1 before reading any one deeply, so the risk register is complete rather than front-loaded.
- **Missing repo:** if a repo in scope is absent from the workspace, list it under *Not scanned* and continue. Do not infer its contents from the LLD.
- The LLD is a proposal, not ground truth. Where the repo contradicts it, the repo wins and the contradiction goes in §7.

## 1. Scan targets

### 1.1 Patterns to reuse

| What to find | Search hints | Used for |
|--------------|--------------|----------|
| External task handlers | `@ExternalTaskSubscription`, `ExternalTaskHandler`, `ExternalTaskHelper` | `ExtractionTriggerHandler`, `ExtractionPaymentHandoffHandler`, `IAPExtractionInitializeHandler` |
| Workflow client | `WorkflowService`, `startWorkflowProcess`, `setTaskDetails`, `completeCurrentTask`, `startMessageCorrelation` | Upload start, `performAction` façade, handoff |
| Message / payment persistence | `MessageService`, `fss_services_message`, `DOC_EXTRACTION`, `IAPIDPaymentsMessageHandler` | Handoff before correlation; one `message_id` per ingest detail row |
| Country/entity factories | `CountrySpecificProcessorFactory`, `CountryExcelMsgProcessorFactory`, `*Factory`, `*Registry` | `AbstractKeyedRegistry` and the three registries (LLD §9.10) |
| IAP ID handlers | `IAPIDPaymentTransactionHandler`, `IAPPaymentBulkTransactionHandler`, `Initialize_IAP` | Package placement, handler base class |
| Bulk upload controller | `BulkUploadController`, `retryTransaction`, `@RequestParam` id style | `ExtractionUploadController` URL conventions |
| Gateway action façade | `@PreAuthorize`, `paymentMaker`, action services with handler maps | `ExtractionUploadActionService`, `UploadActionHandler` |
| Extraction service POJOs | `InitiationDetail`, `ExtractedField`, `IdPaymentOutput`, `POST /v1/extract` | Jackson models — **reuse via the shared contract library, do not duplicate.** See §1.7a |
| Shared contract library | Search every POM in the workspace and the parent BOM for an artifact publishing these response POJOs (names containing `idp`, `extraction`, `contract`, `api`, `common`, `model`) | LLD §6.3.1 depends on it existing. Report the coordinates — this is §1.7a |
| JPA entities/repos | `*Entity`, `*Repository` in payment-gateway | New `ingestion` package — three tables |
| ZIP / PDF libraries | `zip4j`, `PDFBox`, `PDDocument`, `ZipFile`, `ZipInputStream`, archive password handling | `ZipArchiveExtractor`, `PdfDecryptor` — or existing equivalents. Flag any existing code using zip4j `ZipFile`, which needs a real `File` on disk and so cannot satisfy the no-temp-file rule (LLD §9.8.2/§9.8.7) |
| File upload utilities | ZIP unpack, encrypted PDF, multipart handlers | Avoid duplicating existing helpers |
| Camunda BPMN deploy path | `src/main/resources/*.bpmn`, `IAP_ID_Payments` | Deploy `FSSPaymentsDocIngestion.bpmn`; additive `IAP_ID_Extraction_Trigger` |
| Portal UI (optional) | `51786-transaction-data-tiles`, landing tabs, modal patterns | `InstructionUploadTab`, `UploadDetailModal` |

### 1.2 Platform facts

Fill the **Platform facts** table in LLD §13.2. Each row changes a decision already written into the design, so report the value and flag any that invalidates the current text.

| Fact | Where to look | Why it matters |
|------|---------------|----------------|
| Java version | parent POM `java.version` / `maven.compiler.release` | **All Java samples in the LLD are written for 17.** If lower, the §9.5.2 downgrade table must be applied before coding starts, not one compile error at a time |
| Modern features already in use | `record`, `var`, `Map.of`, sealed types in gateway | Match existing style |
| Spring Boot version | parent POM / BOM | Below 2.6, `@ConfigurationProperties` records need `@ConstructorBinding` (LLD §4.2) |
| Hibernate major version | dependency tree | Hibernate 5 may need `@Type(type = "pg-uuid")` for `UUID` ids (LLD §5.1.3) |
| Lombok, Micrometer, MapStruct | POM / BOM | `@Slf4j` in §9.2, metrics in §9.5.7 — do not introduce either for this feature alone |
| UUIDv7 generator | `com.fasterxml.uuid:java-uuid-generator` | Needed by §5.1.3; report version if already managed |
| Existing `Clock` bean | `Clock`, `systemUTC`, `Instant.now()` call sites | §9.5.6 requires injection; count direct `Instant.now()` / `new Date()` uses in the gateway |
| Test stack | JUnit 4 vs 5, Mockito, AssertJ, Testcontainers, H2, existing `@SpringBootTest` slices | §12.3 assumes Testcontainers Postgres is achievable — confirm or flag |

### 1.3 Database

| Check | Why |
|-------|-----|
| Datasource URL, driver version, Hibernate dialect | LLD targets **PostgreSQL**; flag any Oracle-only SQL still in the gateway |
| PostgreSQL server version | §5.4 uses `GENERATED ALWAYS AS IDENTITY` (PG 10+) |
| Database collation / locale | Non-`C` collation means a btree index cannot serve `LIKE 'prefix%'` on `file_id` (§5.1) |
| Default transaction isolation | The File ID allocator's `INSERT … ON CONFLICT DO UPDATE` assumes **`READ COMMITTED`** (§5.1.1); above it, concurrent allocation can abort with `40001`. Report if anything raises it globally |
| Exact type of `fss_services_message.id` | `message_id` / `payment_workflow_key` must match it exactly (§5.1.3) |
| Every `@Lob` on `String` or `byte[]` in gateway entities | On Postgres `@Lob` maps to `oid` large objects, not `TEXT`/`BYTEA` — these leak (§5.1). Report each occurrence and its real column type |
| Primary key generation convention in existing entities | `@GeneratedValue`, `UUID.randomUUID()`, sequence — flag anything that conflicts with application-assigned UUIDv7 (§5.1.3) |
| Migration tooling | Confirm **no Liquibase/Flyway** in the gateway, or document the exception |
| Is Camunda on the same database/schema? | Affects DDL ownership and connection-pool contention |

### 1.4 Runtime and operational limits (highest-risk section — do not skip)

The design makes a **synchronous extraction call of up to 10 minutes** inside a Camunda external task. Everything below decides whether that is survivable. Report the configured value and the source file for each.

| Setting | Search hints | Failure if wrong |
|---------|--------------|------------------|
| External task **lock duration** | `lockDuration`, `ExternalTaskClient`, `SubscriptionBuilder`, Camunda client config | **If the lock is shorter than the extraction call, Camunda re-delivers the task to a second worker and the file is extracted twice** — duplicate detail rows and potentially duplicate payments |
| External task client `maxTasks`, `asyncResponseTimeout`, thread pool size | same | Controls how many 10-minute calls run at once |
| Extraction HTTP client connect/read timeout | `RestTemplate`, `WebClient`, Feign config, `HttpComponentsClientHttpRequestFactory` | A read timeout below the 10-minute budget fails every large document |
| Any proxy / ingress / API gateway / load balancer timeout in front of the gateway or extraction service | k8s ingress annotations, nginx conf, service mesh | A 60s edge timeout silently caps the whole design regardless of client config |
| HikariCP `maximumPoolSize`, `connectionTimeout`, `leakDetectionThreshold` | `spring.datasource.hikari.*` | If the extraction call runs inside a transaction, each concurrent upload pins a connection for 10 minutes and the pool exhausts |
| Whether existing handlers hold a `@Transactional` across remote calls | handler source | Same as above — report the existing convention |
| `spring.transaction.default-timeout`, any `@Transactional(timeout=)` | config + annotations | A 10-minute call may exceed it |
| Multipart limits | `spring.servlet.multipart.max-file-size`, `max-request-size`, Tomcat `maxSwallowSize`, container body limits | 50 MB PDFs and ZIPs (§9.8) |
| Camunda job executor config | `camunda.bpm.job-execution.*` | Throughput of the ingestion process |
| Scheduler locking | `@EnableScheduling`, `@Scheduled`, ShedLock, Quartz | §7.1 reconciliation would otherwise run **simultaneously on every instance** |
| Number of deployed instances / replicas | deployment manifests, Helm values | Confirms the concurrency assumptions in §5.1.1 and §7.1 |

### 1.5 Name and route collisions

Report any existing use of these. A collision here is a production incident, not a merge conflict.

- Tables: `fss_payment_upload_meta`, `fss_payment_upload_content`, `fss_payment_data_ingest_details`, `fss_payment_upload_audit`, `fss_payment_file_seq`
- Camunda process key: `FSS_Payments_Document_Ingestion`
- External task topics: `Trigger_Data_Extraction`, `Trigger_Payment_From_Extraction`, `Initialize_IAP_From_Extraction`
- Message name: `IAP_ID_Extraction_Trigger`
- REST base path: `/api/fss/payments/gateway/v1/extraction-uploads`
- Bean / class names in §9.1 (`ExtractionUploadService`, `EntityHandlerRegistry`, `IdGenerator`, …)
- Config prefix: `extraction.*` in any existing `application.yml`

### 1.6 Cross-cutting infrastructure that can silently break this feature

| Check | Why |
|-------|-----|
| Global `@RestControllerAdvice` / `@ExceptionHandler` | An existing global advice may take precedence over `ExtractionUploadExceptionHandler` and change the `errorCode` contract in §9.11 |
| Global `ObjectMapper` customization | `FAIL_ON_UNKNOWN_PROPERTIES`, naming strategy, `@JsonInclude(NON_NULL)`, custom modules — any of these can break `extracted_data` round-tripping verbatim (§6.3) |
| Spring Security setup | Exact authority string for `paymentMaker` / `paymentChecker`, method-security enablement, how the current user is resolved for `uploaded_by` |
| Existing audit conventions | Whether an audit table/service already exists that `ExtractionUploadAuditService` should follow or reuse |
| Logging / MDC conventions | §9.5.7 puts `uploadId` in MDC; match the existing correlation mechanism if one exists |
| Existing rate limiting or request size filters | May reject large multipart uploads before the controller |

### 1.7 The real `structuredOutput` shape — document scope vs instruction scope

**Treat this as the second-highest-risk section after §1.4.** LLD §6.2.3 builds the entire persistence model on the fan-out: one extraction call per document returns an `initiationDetail[]` array of length *N*, and the gateway writes *N* `fss_payment_data_ingest_details` rows — because `fss_services_message` holds exactly **one payment instruction per row**. If the real shape differs, the DDL, the review modal, the handoff, and the confidence rollups are all wrong together.

**Two things are already decided — do not re-open them, verify the surrounding facts instead:**

| Decided | Where | What that means for this scan |
|---|---|---|
| `initiationDetail` is **always a `List`**, including *N*=1 | LLD §6.1a | The service is being modified to emit this. Report what it emits **today** so the `ExtractionServiceClient` adapter is written against reality, but do not propose a gateway-side array-vs-object branch |
| **The gateway builds `header`**; the service does not return it | LLD §6.1b | Questions about `TotInst` accuracy and `InstructionSummary` semantics are **retired** — the gateway derives both. Do not spend time on them |

Verify against the **Java types and a real fixture**, not against the LLD. Locate `IdPaymentOutput` / `ExtractResponse` / `InitiationDetail` / `ExtractedField`, plus `after_ocr-llm-output.json` or equivalent — **ideally one with more than one instruction**, since a single-instruction fixture cannot reveal any of the multi-instruction behaviour below.

| # | Question | Why it changes the design |
|---|----------|---------------------------|
| 1 | What shape does the service emit **today** — singular object or list? | Determines what `ExtractionServiceClient` must absorb while the §6.1a change is in flight. Not a design question, an adapter question |
| 2 | Does the response type still carry a `header`? If so, is it being removed as part of the §6.1a work? | The gateway now owns the header (§6.1b). A header still on the wire is dead weight to be dropped, **not** a source to read from |
| 3 | **Where is the whole-file confidence score, exactly?** | **Highest-value question in this section.** A document-level score **does exist** (confirmed by the product team) — but the LLD does not know its field name or location. Report the exact JSON path: a sibling of `initiationDetail` inside `structuredOutput`, a field on `ExtractResponse`, or somewhere else. LLD §6.4 cannot be implemented without this |
| 3a | Is that score a **0–100 percentage or a 0–1 fraction**? | A silent unit mismatch makes every file breach `extraction.confidence.low-threshold`. Report the literal values seen in fixtures, not the declared type |
| 3b | Is it **always present** on `COMPLETED`? | Decides whether §6.4's `AVG` fallback is a rare path or the common one |
| 3c | Is it a string or a number? | Consistency with `Confidence1`, which is a string |
| 4 | Is `Confidence1` present on **every** instruction, and is it a string or a number? | §6.4 parses strings with `Double.parseDouble` and falls back to the minimum per-field `Confidence`. Report the declared Java type and whether it is ever null/absent/empty |
| 5 | Are per-field `Confidence` values ever numbers rather than strings? | Mixed types break a `String`-typed `ExtractedField.confidence`. Check the Java record and whether any custom deserializer normalizes them |
| 5a | Is per-field confidence exposed **anywhere other than** the `{Name, Value, Confidence}` triples? | The LLD assumes triples are the only source. A second one needs an explicit precedence rule |
| 6 | *(Retired — the gateway assigns `TotInst` from the list size, §6.1b.)* | — |
| 7 | *(Retired — the gateway computes `InstructionSummary[]` as a group rollup, §6.1b.)* | — |
| 8 | Does anything correlate an instruction back to a **page or document region**? (`DocRef`, `DocId`, `page`, bounding boxes) | Determines whether the review modal can show a PDF preview scoped to the instruction being edited, or only the whole file |
| 9 | Are `TxRef` values **unique within a document**? | Affects display and support lookups only. `instruction_index` is the stable identity regardless, and re-extract is dropped (LLD §7.3), so this is no longer an idempotency question |
| 10 | Can `initiationDetail[]` be **empty** on `status=COMPLETED`? | §6.2.3 treats empty-on-COMPLETED as a failed extraction. Confirm whether the service can actually emit this, or returns `FAILED`/422 instead |
| 11 | Can `DebitDetails.details[]` / `CreditDetails.details[]` hold **more than one** leg per instruction? | The LLD notes phase 1 uses a single leg each. If real documents carry multiple, `EntityPaymentMapper` leg-building and the modal both need to handle it |
| 12 | What is the realistic and maximum *N* for a production document? | Drives batch-insert sizing, the *N* `fss_services_message` writes and *N* correlations at handoff, and whether the review modal needs pagination. An *N* of 3 and an *N* of 300 are different features |
| 13 | Are there template variants per entity/country where the shape differs? | §6.3.1 versions POJOs per template (`model.extraction.id.v2`). Report every template id the service supports and whether their outputs are structurally identical |

Report the answers as a table, each row citing `path/to/File.java:line` or `fixture.json` — and **paste the actual `structuredOutput` block from a multi-instruction fixture verbatim**, so LLD §6.2 can be replaced with a real *N*>1 example rather than the current *N*=1 one.

### 1.7a The shared contract library

LLD §6.3.1 now depends on a **common library that already publishes the extraction response POJOs**, rather than extracting a new contract module or duplicating records in the gateway. Confirm it exists and is usable — each row below can invalidate that plan:

| # | Question | Why it changes the design |
|---|----------|---------------------------|
| 1 | What is the library's group id, artifact id, and current version? Where is it published? | Becomes a POM entry in LLD §9.8.6 — or none, if the gateway already depends on it |
| 2 | Which types does it actually expose? `InitiationDetail`, `ExtractedField`, `LegDetails`, `LegDetail`, `ExtractRequest`, `ExtractResponse`, `JobStatus`? | Anything missing has to come from somewhere; report the gaps rather than assuming completeness |
| 3 | What `maven.compiler.release` is it built at? | **Blocking if it is 17 and the gateway is on 8 or 11** (§1.2) — the class files cannot be read at all |
| 4 | What does it drag in transitively? Spring AI, WebFlux, a JDBC driver, Lombok? | A contract library should be records only. Report the full `dependency:tree` for it so exclusions can be specified |
| 5 | Does it already model `initiationDetail` as a **`List`**, and has `header` been removed from its response type? | If it predates the §6.1a shape, updating **the library** is part of that work item — not something the gateway works around |
| 6 | Does it carry the whole-file confidence field from §1.7 question 3? | If the field exists on the wire but not in the library type, it cannot be read type-safely and the library needs updating too |
| 7 | Who owns it, and what is the release cadence? | Determines whether the §6.1a change can land on the gateway's timeline or needs a temporary local shape |

## 2. Validation checks

- Confirm **`entity` is the only routing key** on new DDL — no separate `country` column. Note if the extraction HTTP client must map `entity` → `country` in metadata (adapter only).
- Work §1.7 in full — the extraction contract is where a wrong assumption is most expensive.
- Confirm **one detail row per instruction** end to end: *N* instructions → *N* `fss_payment_data_ingest_details` → *N* `fss_services_message` → *N* `IAP_ID_Payments` instances. Scan the gateway **and** the UI for legacy single-row-per-upload or single-instruction-per-file assumptions, and check whether any existing handler assumes one message per uploaded file.
- Confirm nothing existing writes **multiple instructions into one `fss_services_message` row**, and that `MessageService.save` has no per-file uniqueness constraint that *N* rows from one document would violate.
- Confirm the **gateway façade** rule: the portal must not call `workflow-management` directly for Submit/Cancel. Find existing `WorkflowService` usage from gateway services.
- Identify the **base class or helper** used by existing IAP external task handlers (retries, complete/fail, variable read).
- Check parent POM / BOM for **zip4j** and **pdfbox**; reuse managed versions if present.
- **Password security:** confirm no existing code persists upload passwords in DB, audit, logs, or Camunda variables.
- Cross-check `final/IDP_API_Reference.md`: `getUploads` serves `instructionCount` and `confidence` from meta (not computed on poll); `getDetail` returns flat `ingestDetails[]`; query param standard is `uploadId`.

## 3. Deliverable A — Reuse & alignment report

For each row in LLD **§9.3** (interface & class inventory), plus `ZipArchiveExtractor`, `PdfDecryptor`, `UploadEntryFilter`, `SubmitUploadActionHandler`, `CancelUploadActionHandler`, `AbstractKeyedRegistry`, `IdGenerator`, `FileIdGenerator`, `FileIdParser`, `PostgresFileIdSequenceAllocator`, and the exception classes:

### &lt;ClassName&gt;
- **Verdict:** Reuse `path/to/Existing.java` | Extend | **New**
- **Pattern to copy:** `path/File.java:120-134` — quote 5–15 lines
- **Package:** recommended, based on neighbours
- **Gaps:** anything the LLD assumes that does not exist

Then fill both tables in LLD §13.2 verbatim so they can be pasted back.

## 4. Deliverable B — Risk register

Every finding that threatens the design, most severe first:

| # | Finding | Evidence (`path:line`) | Severity | Impact | Recommended action |
|---|---------|------------------------|----------|--------|--------------------|

Severity definitions:

- **Blocker** — the design cannot work as written (for example, the external task lock is shorter than the extraction call, or an edge timeout caps the request well under 10 minutes). Must be resolved before step 1 of §14.
- **Major** — works, but needs a design change or an owner decision (for example, the connection pool cannot support the target concurrency).
- **Minor** — local adjustment during implementation.

## 5. Deliverable C — Implementation backlog

Convert LLD §14 into a concrete, ticket-ready backlog **grounded in the real repo**. Preserve the §14 order and dependencies, but split, merge, or add tasks wherever the scan showed the LLD was wrong about what exists.

One row per task:

| ID | Task | Files to create / modify (real paths) | Depends on | Size | Acceptance gate | Open questions |
|----|------|----------------------------------------|------------|------|-----------------|----------------|

Rules:

- **Size** is S (&lt; ½ day), M (½–2 days), or L (&gt; 2 days). Split any L that can be split.
- **Acceptance gate** must be an observable check — a passing named test, a successful startup assertion, a verified Cockpit state — never "code complete".
- Every task blocked by an `UNKNOWN` or a **Blocker** carries that marker in *Open questions* and does not appear in the ready-to-start set.
- Call out which tasks can run **in parallel** and where the critical path runs.
- Put the resolution of every **Blocker** as an explicit task ahead of the code it blocks.
- Finish with a **ready-to-start set**: tasks with no unresolved dependency or unknown, which is where work begins on day one.

## 6. Deliverable D — Package tree

The §9.1 tree rewritten against the real repo structure and its actual base package.

## 7. Deliverable E — LLD corrections

Everywhere the repo contradicts `final/IDP_LLD.md`, as `section → what it says → what is actually true → suggested replacement text`. This is what gets applied back into the document, so be specific enough to edit from.

## 8. Output rules

- Markdown; the five deliverables in order, under clear headings.
- Lead with a short summary: how many classes are reusable, the count of Blocker / Major / Minor risks, and the single biggest thing that would derail the build.
- **Do not implement anything.** Discovery only, unless I explicitly ask.

=== END PROMPT ===

---

## After the scan

1. Paste the reuse report and both §13.2 tables back into `IDP_LLD.md`, or attach as `IDP_LLD_REPO_FINDINGS.md`.
2. Apply **Deliverable E** corrections to the LLD before any code is written — that is the "enhanced LLD".
3. Resolve every **Blocker** in the risk register. Section 14 step 0 is not complete while one is open.
4. Work the **Deliverable C** backlog in order, honouring the §12.3 quality gates.
