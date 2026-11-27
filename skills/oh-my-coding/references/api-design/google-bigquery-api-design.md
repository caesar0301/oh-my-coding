# Case Study: Google BigQuery API Design

> **Attribution**: Google BigQuery REST v2 — `jobs`, `Job`, `Dataset`
> (`https://cloud.google.com/bigquery/docs/reference/rest/v2/jobs`), and the
> Google API Improvement Proposals AIP-121/122/132/136/151/155/157/158/163/180/193
> (`https://google.aip.dev/`). Retrieved 2026-11.
> This is an **informative case study**, not a normative source. `philosophy.md`
> is the authority; when they appear to disagree, the philosophy wins.

## Why this case study exists

BigQuery is a long-evolved, large-scale data API whose design choices
crystallized into the AIP principles that also inform our seven-principle
philosophy. It is useful as a **source of philosophy**, not of shape: its
resource model, field names, and path syntax are tied to a SQL data warehouse
and MUST NOT be copied into a project's own API contract. The rule is **borrow
the philosophy, never the shape**.

## Principle-by-principle mapping

| Principle | How this API instantiates it | Transferable lesson |
| --------- | ---------------------------- | ------------------- |
| P1 One Operation Abstraction | All asynchronous work is a single `jobs` resource; action differences live in a `configuration` union rather than separate resources. | One envelope plus typed configuration beats one resource per action. |
| P2 Resource-Oriented, Bounded Verbs | Resource-oriented surface with standard methods; custom methods are exceptions and must avoid path collisions (AIP-136/180). | Organize around nouns; keep custom actions rare and unambiguous. |
| P3 Cursor-First Pagination | `list` uses `pageToken` / `nextPageToken`; pagination was designed in from the start rather than added later (AIP-132/158). | Pagination is a first-version contract, not a retrofit; tokens are opaque and bound to filter/order. |
| P4 Request-Scoped Idempotency | `jobReference.jobId` is caller-chosen; request ids are optional but, once supplied, must guarantee idempotency and replay the first response (AIP-155). | Let the caller name the request, then guarantee replay safety. |
| P5 Uniform Envelope, Open Dimensions, Additive Evolution | Constant `kind` / `etag` / `id` / `selfLink` envelope; free-form `labels` map; backward-compatible evolution forbids new required fields or default changes (AIP-122/180); views add fields only (AIP-157). | A constant shell plus open dimensions and additive-only change is what makes evolution safe. |
| P6 Two-Layer, Machine-Readable Errors | A unified error model with machine-readable `ErrorInfo`; dynamic information does not go into `message` (AIP-193). | Stable `code` + structured `details` + separate job-terminal failure layer. |
| P7 Projection as Parameter | `projection` (FULL/MINIMAL) and `dryRun` are parameters on the same endpoint, not separate endpoints. | Observation modes and dry runs are switches, not new endpoints. |

## Startup failure vs execution failure

This API separates "the request was rejected" from "the job ran and failed":
a job that starts successfully but fails later reports the failure in its
terminal state, not as an HTTP error on `jobs.insert`. This is the concrete
motivation for P6's two-layer error model: creation-time HTTP errors and
job-terminal errors are different layers and MUST NOT be conflated.

## Borrow philosophy, not shape — do not copy list

The following shapes belong to this API and MUST NOT appear in a project
contract that follows `philosophy.md`. They are the exclusion boundary made
concrete; see `philosophy.md` §"Exclusion boundary" for the general form.

| Area | Do not copy |
| ---- | ----------- |
| Job abstraction | `google.longrunning.Operation`; `jobReference` / `jobId` naming and SQL job types; a shared `/v1/jobs` service or central orchestrator. |
| Resources and verbs | `:verb` path syntax; `jobs.query` / `tabledata.list` naming; protobuf standard-method signatures. |
| Pagination | `maxResults` / `totalRows` semantics; any specific token encoding. |
| Idempotency | `jobId` naming and mandatory UUID format; vendor-specific retry status-code tables. |
| Envelope and evolution | `labels` field-name constraints; `schema` / schema-update options; protobuf field-behavior annotations; `etag` as mandatory concurrency control. |
| Errors | `google.rpc.Status` type; `errorResult` naming; vendor-specific reason namespaces. |
| Projection | `fields` syntax details and `projection` naming; SQL-style column-selection semantics. |

**Why**: a project's own API has its own resource model, ownership rules, and
verb set that differ from a SQL warehouse's — for example, different object
sizes, reference semantics, submission mechanisms, and lifecycle states.
Importing SQL job types, a vendor operation protocol, or `:verb` syntax would
collide with that local model rather than serve it.

## Targeted reads

When only one principle is in question, search this file by tag:

- P1 → `Principle-by-principle mapping`, row `P1`
- P3 → `Principle-by-principle mapping`, row `P3`
- P4 → `Principle-by-principle mapping`, row `P4`
- P5 → `Principle-by-principle mapping`, row `P5`
- P6 → `Startup failure vs execution failure` and row `P6`
- P7 → `Principle-by-principle mapping`, row `P7`
- Any forbidden shape → `Borrow philosophy, not shape — do not copy list`
