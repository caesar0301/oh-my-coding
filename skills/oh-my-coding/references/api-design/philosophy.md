# API Design Philosophy — Seven Decidable Principles

This file is the **normative layer** of the API design reference. It states the
end-state invariants and their acceptance criteria for an API, and
nothing else. It is **shape-neutral by contract**: no vendor resource model,
protocol type, field name, or path syntax appears here. All external API
specifics live in the case-study subcategories under this directory (see
`README.md`).

A project's own RFCs and interface contracts take precedence over this file on
conflict; record the conflict rather than silently overriding the project. When
this file and a case study disagree, this file wins.

## Design constraints

1. **Shape neutrality** — A project's contracts, specs, and implementations
   MUST NOT contain any external API's proprietary resource model, protocol
   type, field name, or path syntax. The exclusion boundary is listed below.
2. **Law and implementation are separate** — This file gives only invariants
   and acceptance criteria. Field-level contracts belong to the project's own
   specs and interface designs; this file MUST NOT depend on any draft.
3. **Decidability** — Every principle MUST map to verifiable statements, never
   to slogans. A rule without an acceptance test does not belong here.
4. **Evolution first** — New capabilities SHOULD be added through dimensions,
   parameters, or configuration; they MUST NOT be added through new endpoints,
   new required fields, or renamed fields.

## Vocabulary

| Term | Meaning |
| ---- | ------- |
| **Operation Envelope** | The constant job envelope (field set and state machine) shared by all long-running actions, independent of action type. |
| **Typed Configuration** | The typed configuration inside the envelope that carries per-action differences. When opaque to the service, it is passed through unchanged, additive only. |
| **Opaque Cursor** | An opaque pagination cursor. Clients MUST NOT parse or construct it; it is bound to filter and order. |
| **Open Dimension** | An open-ended vocabulary or tag dimension that may grow freely; the server does not fix its values in the contract. |
| **Compatibility Contract** | Field behavior and evolution rules: additions are optional; existing fields MUST NOT be renamed, retyped, default-changed, constructor-changed, or made required. |
| **Projection** | A read-only observation view of a resource, selected by parameter rather than endpoint, eventually consistent and carrying `asOf`. |

## Exclusion boundary (shape neutrality)

The following shapes and names MUST NOT appear in a project contract that
adopts this philosophy. This is the single external-reference boundary; the
principles below describe only end-state design.

| Area | Do not introduce |
| ---- | ---------------- |
| Job abstraction | A vendor's long-running-operation protocol type; vendor job-reference naming or SQL job types; a shared `/v1/jobs` service or central orchestrator when the project's own service boundaries forbid it. |
| Resources and verbs | `:verb` path syntax; vendor-specific `resource.method` naming; protobuf standard-method signatures. |
| Pagination | `maxResults` / `totalRows` semantics; any specific token encoding. |
| Idempotency | Vendor-specific request-id naming or mandatory UUID format; vendor-specific retry status-code tables. |
| Envelope and evolution | `labels` field-name constraints; `schema` / schema-update options; protobuf field-behavior annotations; a mandatory vendor etag as concurrency control. |
| Errors | A vendor's `google.rpc.Status` type; vendor `errorResult` naming; vendor-specific reason namespaces. |
| Projection | Vendor `fields` syntax details or `projection` naming; SQL-style column-selection semantics. |

---

## P1 — One Operation Abstraction

**Statement**: All long-running actions MUST share one `Operation Envelope`;
action-type differences MUST appear only in `Typed Configuration`.

**Rules**:

1. Every service MUST use the same envelope field set and state machine; MUST
   NOT add a dedicated job resource or dedicated status enum for one action
   type.
2. Differences MUST be captured in `Typed Configuration`. When the
   configuration is opaque to the service, it MUST be passed through
   unchanged, additive only.
3. Every long-running action MUST offer `202 + Location`, a `get` snapshot,
   `events`, and `cancel`. `Location` MUST point to a `get`-able snapshot, and
   the `202` body and the `get` body MUST be isomorphic.
4. Resource lifecycle state and job state MUST be expressed separately;
   `queued` / `waiting` MUST NOT be written into a resource's `status`.
5. Execution-time failure MUST be expressed in the envelope's terminal error
   field, MUST NOT be expressed as an HTTP error on the creation request.

**Acceptance**: For any long-running action, one can write its envelope and
`Typed Configuration` branch without adding an endpoint or status value; `202`
and `get` bodies have identical fields; no resource `status` contains a job-state
value.

## P2 — Resource-Oriented, Bounded Verbs

**Statement**: The API surface MUST be organized around resources (nouns);
actions MUST converge to a finite verb set, with custom actions reserved for
what standard verbs cannot express.

**Rules**:

1. Every resource MUST have one canonical name and one canonical path; the same
   resource MUST NOT be exposed under two parent paths; a `Ref` MUST resolve
   mechanically to that path.
2. The verb set is fixed by the project's own protocol (for example a 7 + 2
   set). A new named action MUST first prove it cannot be expressed by
   `create` / `patch` / `run` / `cancel` plus a state field, and MUST be recorded
   in a verb-mapping table.
3. A custom action MUST NOT be ambiguous with a resource-id path segment; the
   action name MUST NOT carry a preposition or a state name.
4. A read-only dry run MUST be expressed by a `validateOnly`-style parameter on
   the same endpoint; MUST NOT create a separate `/check` endpoint per resource.
5. Resource references MUST form a DAG; cross-service references MUST NOT embed
   resource objects.

**Acceptance**: The verb table is enumerable and consistent with the project's
protocol; each resource has one canonical path; the routing table has no id /
action conflicts; adding a dry run does not increase the endpoint count.

## P3 — Cursor-First Pagination

**Statement**: Every collection read MUST offer `Opaque Cursor` pagination from
the first public version; offset paging MUST NOT be the only mechanism.

**Rules**:

1. `list` MUST provide `pageSize` and an opaque `pageToken`; the response MUST
   contain `nextPageToken` and a uniform collection envelope.
2. `pageToken` MUST be opaque and bound to filter and order; clients MUST NOT
   parse or construct it.
3. The first version offers cursor pagination only; offset paging is out of
   scope and MUST NOT be provided as a parameter.
4. `filter` and `orderBy` MUST be reserved as common extension points; adding a
   filter dimension MUST NOT add an endpoint or a required parameter.
5. Collection response field names MUST be uniform; module-specific variants
   (`items` / `entries` / `series` / `nodes`) MUST NOT appear.

**Acceptance**: For any `list`, a filter dimension can be added and pages can
be traversed stably without changing the endpoint; concurrent inserts cause no
duplicates or omissions; all collection responses use the same field names.

## P4 — Request-Scoped Idempotency

**Statement**: Idempotency MUST be a request-level contract, not an accidental
property of a resource type.

**Rules**:

1. All create actions and all named actions MUST accept a request identifier
   (caller-chosen `id` or `Idempotency-Key`) and MUST declare the mechanism used.
   **Exception**: short-lived credential-issuance endpoints are non-idempotent,
   MUST NOT accept an idempotency key, and MUST declare this explicitly.
2. A replay MUST return the first successful response (same status code and
   `Location`) and MUST NOT create a second resource or second side effect.
3. The request identifier MUST define its scope (account + service + endpoint),
   format, and retention period; a same-id different-payload conflict MUST have
   an explicit semantic.
4. One-time plaintext (credentials, secrets) MUST define replay behavior; it
   MUST NOT claim both "returned once" and "unconditionally idempotent" without
   explanation.
5. Idempotency MUST be consistent with the project's compensation model:
   clients may retry safely; the server MUST NOT automatically roll back
   another party's resources.

**Acceptance**: Replaying the same identifier twice yields byte-equivalent
responses and a side-effect count of one; the same id on a different endpoint
does not collide; the same action declares idempotency consistently across
services.

## P5 — Uniform Envelope, Open Dimensions, Additive Evolution

**Statement**: Every resource MUST have a constant envelope; classification
dimensions MUST be `Open Dimension`s; evolution MUST be additive only.

**Rules**:

1. Every resource MUST have the project's minimal envelope; `kind` MUST be
   globally unique and MUST NOT be reused for business classification, which
   MUST be named separately.
2. Growable classification dimensions MUST be expressed as open vocabularies or
   free tags; the server MUST NOT fix values in the contract; unknown values
   MUST NOT break clients.
3. `Compatibility Contract`: new fields MUST be optional; existing fields MUST
   NOT be renamed, retyped, default-changed, constructor-changed, or made
   required. Response enums MAY gain values; clients MUST tolerate unknown
   values.
4. Opaque objects such as `Typed Configuration` MUST be additive only and
   unknown keys MUST be passed through.
5. Every field MUST declare its behavior (required / read-only / immutable) so
   that adding a read-only field does not break clients.
6. An optional mechanism (for example `Ref@etag`, `view`, `validateOnly`), once
   declared, MUST define its producer, consumer, and failure semantics; it MUST
   NOT be declared without being used.

**Acceptance**: Every field can be listed with a behavior tag; adding a read-only
field or a new enum value requires no client upgrade; the `kind` registry has no
name collisions; every declared optional mechanism has consumer semantics.

## P6 — Two-Layer, Machine-Readable Errors

**Statement**: Errors MUST be split into "request rejected" and "job terminal
failure" layers; both layers MUST share one machine-readable shape.

**Rules**:

1. An HTTP error body MUST contain a stable `code`, `message`, request
   identifier, and `details[]`; `details` MUST carry at least reason / domain /
   metadata; clients MUST NOT branch on `message` text.
2. One `code` MUST map to exactly one semantic and one HTTP status; the same
   `code` MUST NOT map to both `400` and `422`.
3. Non-HTTP error signals (asynchronous resolution failure, deferred-state
   wait, gate failure) MUST belong to an operation terminal state or a data
   field; they MUST NOT be mixed into the HTTP error-code table.
4. The error model MUST cover both edge-level and product-level error bodies and
   state which to use when.
5. Retryability MUST be decidable from the response body, not only from a
   documentation table.

**Acceptance**: For any failure scenario, one can determine its layer, that its
`code` is unique, and that its `details` are machine-readable; an asynchronous
resolution failure does not appear in the creation
request's HTTP error list.

## P7 — Projection as Parameter

**Statement**: Different observation views of the same resource MUST be
expressed through a `Projection` parameter; MUST NOT add an endpoint per view.

**Rules**:

1. Observation dimensions MUST be selected through common `view` / `fields`
   parameters; a view enum MAY gain fields, MUST NOT remove published fields.
2. A `Projection` MUST carry `asOf` and MUST NOT be used for gating or release
   decisions.
3. Adding an observation dimension MUST NOT increase the endpoint count.
4. Aggregate views MUST declare eventual consistency; authoritative decisions
   MUST use the resource `get`.

**Acceptance**: Adding an observation dimension leaves the endpoint count
unchanged; all read-only projections carry `asOf`; no projection is used for
gating decisions.

---

## Worked examples

### Adding a new long-running action

A new post-processing action MUST NOT create a dedicated resource or status
enum. Add a new `Typed Configuration` branch to the existing envelope and reuse
`202 + Location` / `get` / `events` / `cancel` (P1). The verbs remain `create` +
`cancel`; no new verb (P2). New configuration keys are additive and unknown keys
pass through (P5).

### Filtering and paging by arbitrary tags

Filter by a custom tag, request a summary view, and page stably. Tags are an
`Open Dimension` — no fixed enum, no new column (P5). Use `filter` + `orderBy` +
an opaque `pageToken`, with response field names identical to other `list`
responses (P3). The summary is a `view` parameter, not a new endpoint (P7).
Endpoint count, required parameters, and existing field names are unchanged —
the decidable form of design constraint 4.

### Asynchronous resolution failure after creation

A creation request references a resource that does not exist. The request
validates only reference syntax and returns `202 + Location` normally (P1).
Resolution happens after creation, at a later lifecycle boundary; failure is
expressed in the operation's terminal error field with reason and metadata, not
as an HTTP error on creation (P6). The failure reason is machine-readable;
clients do not parse `message`.

## Relationship to project-level specs

This file is an upstream design authority, not a replacement for a project's
own specs. Field-level contracts (verb mapping table, `kind` registry, error
`details` registry, field-behavior annotations) belong to the project. This file
MUST NOT depend on those drafts, and those drafts MUST NOT redefine these seven
principles.
