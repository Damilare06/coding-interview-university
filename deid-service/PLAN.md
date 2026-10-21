# Enterprise Data De-identification Service: Build Plan

Version 3, October 2026. Incorporates the architecture review of 2026-10-21.
Change log at the end.

## 0. What the service is

A privacy-first processing service that securely ingests enterprise data through
connectors or file imports, discovers and classifies sensitive information,
de-identifies it under versioned policy, validates the released dataset, and
exports approved output together with independently verifiable evidence.

Two outputs per release: a usable de-identified dataset and a signed processing
assurance report.

The differentiator is defensible assurance: completeness, authorisation, measured
privacy risk, and non-retention within an audited boundary. Not PII detection.

Excluded: model training, HIPAA specialisation, marketplace, analytics or BI,
cataloguing, MDM, general ETL. Raw customer data is never collected for debugging
or detector improvement without separate written authorisation.

## 1. Four separate assurances, stated precisely

We make four distinct promises. They are never collapsed into one, in contracts,
marketing, or the manifest.

| Assurance | What it establishes | What it does not establish |
|---|---|---|
| Schema coverage | Every discovered field, including nested fields, received a policy decision | That every value inside a field was detected |
| Processing completeness | Every record in the defined source snapshot has a documented disposition, with duplicate and loss detection | Anything about detection quality |
| Detection effectiveness | Measured precision and recall per identifier type on labelled and adversarial sets, with quantified uncertainty | Perfect detection in arbitrary text |
| Non-retention | No unauthorised raw-data persistence was observed within the audited processing boundary, under documented infrastructure controls and verification tests | That nothing was ever stored anywhere, unqualified |

And one gate that is legal rather than technical:

| Authorisation | Source provenance, claimed rights, permitted purpose, and permitted recipients were recorded and checked before release |

De-identified is not the same as anonymous, legally distributable, or free of
confidential business information. Each is assessed separately.

## 2. Customer workflow

1. Connect or import, with scoped read-only access.
2. Pass 1, discovery and planning: classify, profile, compute dataset-level risk
   statistics, generate the transformation policy, obtain approvals.
3. Pass 2, partitioned transformation against the same source snapshot.
4. Global validation: completeness, independent privacy assessment, utility,
   integrity, authorisation, non-retention evidence.
5. Atomic release: the dataset becomes visible only when every gate passes;
   otherwise it is quarantined in customer-controlled storage.

## 3. Architecture

Control plane and data plane are separated. The control plane is hosted by us and
stores no record values. The data plane is a portable, headless service with a CLI
that runs in our isolated environment or, as the enterprise default, inside the
customer's own cloud account. Core transformations are identical in both.

```
                 ┌──────────────── Pass 1: discovery and planning ────────────────┐
 Source snapshot │ classify (multi-dim) → profile → global risk stats → policy →   │
 (consistent)    │ approvals                                                      │
       │         └──────────────────────────────┬───────────────────────────────┘
       │                                        ▼
       │         ┌──────────────── Pass 2: partitioned transformation ────────────┐
       ├────────▶│ Arrow batches (adaptive size) → transform → per-partition       │
       │         │ disposition ledger + commitments → quarantine storage (customer)│
       │         └──────────────────────────────┬───────────────────────────────┘
       │                                        ▼
       │         ┌──────────────── Global validation and release ─────────────────┐
       └ changed?│ snapshot unchanged? → completeness → independent privacy engine │
         abort   │ → utility → authorisation → non-retention evidence → atomic     │
                 │ publish or quarantine                                           │
                 └────────────────────────────────────────────────────────────────┘
 Control plane (ours, no record data): tenants, policies, jobs, approvals,
 manifests, delivery registry, key references, licensing rules.
```

### Design rules
- Raw data lives only in worker memory and customer-controlled quarantine storage.
  Quarantined intermediate output is pseudonymised personal data and is protected
  as such, never assumed safe because a transform ran.
- The source snapshot must be consistent: database snapshot isolation, immutable
  object versions, or a connector-defined extraction boundary. Both passes read the
  same snapshot. If it changes between passes, the job aborts.
- Policies are versioned YAML; the policy hash is in every manifest.
- Partitions are idempotent units with their own disposition ledger. Retries
  overwrite; duplicates are detected, not just counted.
- Batch size adapts to a memory budget, not a fixed row count.
- The data plane is cloud-agnostic. Cloud-specific orchestration stays in our
  hosted environment behind thin abstractions.
- Hosted file imports land in a short-lived ingest bucket with enforced lifecycle
  deletion, and that bucket is inside the audited boundary and disclosed as such.

### Deployment models

| Model | Advantage | Trade-off |
|---|---|---|
| Vendor-hosted isolated workers | Easy onboarding for file imports | Non-retention boundary is ours to prove |
| Customer cloud or VPC (enterprise default) | Sovereignty; evidence from customer logs | Deployment and support effort |
| Confidential computing (later) | Runtime attestation | Does not cover the rest of the pipeline |

### Stack

| Layer | Choice | Note |
|---|---|---|
| Data plane | Python, PyArrow, Polars; headless service plus CLI | Typed interfaces; engine separate from orchestration |
| Control plane | Python, FastAPI, PostgreSQL | No record values |
| Connectors | Python connector SDK | Contract: snapshot, partition, read batches, checkpoint |
| Transformation detection | Presidio plus custom recognisers | Throughput budget required |
| Assurance detection | Independent rule set and, where warranted, independently trained models | Must not share false negatives with the above |
| Hosted execution | ECS/Fargate, SQS, Step Functions | Behind abstractions |
| Customer execution | Same container via Terraform module or customer scheduler | |
| Secrets and keys | Secrets Manager or Vault plus KMS | Per tenant, per purpose |
| Evidence | Signed manifests, hash-chained audit log, customer cloud activity logs | |
| Web UI | Next.js, later | After evidence format stabilises |

Not initially: Kafka, Kubernetes, Spark. Revisit only if load tests show distributed
execution beats isolated parallel workers.

## 4. Classification

Multidimensional, not five mutually exclusive buckets. Each field, nested path,
array element type, and file metadata attribute carries:

| Dimension | Values |
|---|---|
| Structure | scalar, structured object, array, free text, binary, embedded document |
| Personal information | none, direct identifier, quasi-identifier, sensitive personal information |
| Business sensitivity | public, internal, confidential, proprietary, trade secret |
| Permission | authorised for transformation, permitted for onward distribution, prohibited |
| Required action | remove, tokenize, generalize, retain, quarantine for review |

Three detection layers feed this: schema heuristics, content sampling with
validators, and NER plus confidential-information patterns (credentials, keys,
pricing, roadmap language) for free text. Schema changes between snapshot and
processing abort the run. Low-confidence fields block until a named operator
approves; approval is recorded.

A fully anonymous record that contains unpublished acquisition plans still fails
the business-sensitivity and permission dimensions.

## 5. Transformation engine

| Action | Use | Rules |
|---|---|---|
| suppress | national IDs, exact address, raw names | Drop or null |
| pseudonymize | join keys, emails | HMAC-SHA256, per-tenant per-purpose key, one-way. Output is labelled pseudonymised personal data, never anonymous |
| tokenize (format-preserving) | only where a real format requirement exists | No default algorithm. FF3-1 is not adopted: NIST's 2025 draft of SP 800-38G rev 1 removed the FF3 family. Use a reviewed construction, case by case |
| generalize | ZIP to ZIP3, age bands, region | Hierarchies in policy |
| date_shift | timestamps | Per-entity shift; flagged as quasi-identifier because distinctive sequences survive; assessed in the privacy engine |
| bucket / round | income, counts | Configurable |
| redact_text | notes, tickets | Typed placeholders |
| keep | non-identifying, authorised | Explicit |

Removed from v1: generic `perturb` with Laplace noise. Differential privacy needs a
privacy unit, contribution bounds, a release mechanism, and cumulative accounting.
It returns only as a separately specified mechanism with a validated use case.
Removed: salted hashes as a protection in themselves; all keyed transforms are
purpose-limited keyed constructions.

### Key management
- One-way by default. No re-identification vault without a demonstrated requirement.
- Keys per tenant and per purpose, held in KMS separately from data, rotated, with
  key version in the manifest.
- Cross-dataset or cross-delivery token consistency is a per-contract decision.
  Keys are scoped to a contract so tokens from different recipients cannot be joined.
- True anonymisation is a separate, rigorously assessed output mode, not a label
  applied to pseudonymised output.

### Free-text throughput budget
NER is the bottleneck. Phase 1 sets a per-record latency budget, batched inference,
model-size options, and cost per million text records before any scale target is
claimed. Test tabular-only, text-heavy, and mixed workloads separately.

## 6. Assurance subsystem

Separate from the transformation engine, with its own rules and models, benchmarked
on labelled records containing real sensitive-data patterns. Any unresolved gate
fails the release closed.

**Gate A, source snapshot and input completeness.** Consistent snapshot descriptor,
file inventory with object versions, partition counts, schema manifest, connector
checkpoints, rolling input hash. Snapshot drift aborts.

**Gate B, record-level integrity.** Not count equality alone. Each partition keeps
a disposition ledger: every source record has exactly one outcome (exported,
suppressed, excluded by suppression policy, errored), with duplicate and loss
detection via keyed row commitments. Only commitment roots and counts leave the
worker, so the ledger cannot itself leak identifiers. Partition manifests bind to
the published objects by checksum. Referential integrity across related tables is
checked where declared. Publication is atomic.

**Gate C, release-specific privacy assessment.** Chosen per output type and
recipient, considering external information and prior deliveries:

| Output | Validation |
|---|---|
| Structured, direct identifiers removed | Independent identifier scan, quasi-identifier risk, linkage analysis |
| Pseudonymised relational | Token isolation, key scope, joinability |
| Free text | Independent residual PII and confidential-information detection |
| Aggregates | Small-cell suppression, disclosure risk |
| Repeated exports | Cross-release linkage, cumulative disclosure |

k-anonymity and l-diversity are applied where their assumptions hold, with
thresholds per release, never as a universal gate. Detector effectiveness is
reported with uncertainty. The engine also flags transforms that preserved more
information than policy intended.

**Gate D, non-retention within the audited boundary.** Enforced at infrastructure
level and evidenced independently of the worker:
- workload identity; write permission only to quarantine, export, and evidence prefixes
- egress restricted to source and sink endpoints
- read-only root filesystem, no persistent volumes, encrypted ephemeral buffers,
  swap and core dumps disabled
- application logging and telemetry scrubbed by schema: structured logs with
  allow-listed fields only; connector debug and retry caches disabled
- failed multipart uploads cleaned by lifecycle rule; quarantine lifecycle enforced
- periodic control tests that attempt each leakage path and record the result
- evidence cross-referenced to the customer's own cloud activity logs

The certificate states: no unauthorised raw-data persistence was observed within
the audited processing boundary, under the documented controls and verification
tests. Nothing stronger.

**Gate E, authorisation.** Provider identity, claimed rights to process, intended
purpose, permitted recipients, redistribution restrictions, and the CPRA
de-identification obligations (reasonable safeguards, public commitment against
re-identification, contractual restrictions on recipients) are recorded and
checked before release.

**Utility report.** Column retention, generalisation levels, null-rate change, and
distribution drift versus source for declared analytic columns.

**Evidence schema.** The manifest and audit chain contain counts, commitment roots,
policy and key versions, gate results, identities, and timestamps. They never
contain raw values or unkeyed hashes of identifiers, which would themselves be
re-identifiable. The delivery registry maps keyed tokens to recipients and
contracts under a separate key and access policy, so deletions can be propagated
without the registry becoming a linkage oracle.

## 7. Regulatory and contractual framework

| Framework | Platform behaviour | Narrowed claim |
|---|---|---|
| CCPA/CPRA | De-identification obligations as Gate E items | Compliance is the customer's; we supply evidence |
| California Delete Act and DROP | Generic suppression-policy interface first. The official DROP workflow, with its recurring 45-day cycle, standardisation and hashing procedures, and outcome reporting, is a separately scoped capability built only once the covered entity is confirmed | Row exclusion alone does not satisfy DROP |
| Other US state laws | Policy packs; Texas, Oregon, Vermont registries | Applicability confirmed with counsel |
| GDPR / UK GDPR | Pseudonymisation with separate keys; anonymisation as a distinct assessed mode; DPIA template; Art. 30 export from manifests | Pseudonymised data remains personal data; manifests support but do not constitute Art. 30 records; the DPIA template is an aid, not a completed assessment |
| EDPB Guidelines 02/2026 on anonymisation | Inform the privacy engine | In consultation; not final |
| NIST SP 800-188 | Disclosure-risk method | |
| ISO/IEC 27559 | Control checklist | Not a certification |
| NIST Privacy Framework, SOC 2 | Organisational controls | |
| Sector and contract | Financial, employment, education, consumer, and proprietary data carry their own restrictions, captured in Gate E | Excluding HIPAA removes one regime, not all |

## 8. Roadmap

| Phase | Scope | Exit criterion |
|---|---|---|
| 0. Threat model and contracts | The four assurances and their limits, processing boundary, retention model incl. logs, telemetry, swap, egress, third-party SDKs; source licensing; data-plane interfaces | Reviewed architecture and threat model |
| 1. Secure reference pipeline | Headless data-plane service plus CLI; file imports; multi-dim classification; policies; transforms; two-pass model on a snapshot abstraction; disposition ledger; signed manifests; NER throughput budget | Complete processing and testable evidence on synthetic data |
| 2. Independent assurance | Independent privacy engine; Gates A to E; non-retention controls and control tests; global release gate; negative-path tests | Demonstrably fails closed on missing records, detector blind spots, unauthorised exports, infrastructure failures |
| 3. Scale and connectors | Adaptive batching; resumable jobs; PostgreSQL and S3; parallel workers | 1M then 10M records with bounded memory, recovery, full accounting, on tabular, text-heavy and mixed sets |
| 4. Enterprise deployment | Customer-VPC module; RBAC; audit operations; hardening; retention controls | Enterprise security review |
| 5. Extended | More connectors; DROP workflow if applicable; advanced privacy methods; Spark if justified by benchmarks | Demand-driven |
| 6. Pilot | Real data in customer cloud; independent review | Pilot acceptance |

## 9. Acceptance criteria for the first enterprise release

| Area | Criterion |
|---|---|
| Scale | 10M records, three workload types, bounded memory |
| Completeness | Every snapshot record has one disposition; duplicate and loss tests pass |
| Retention | No unauthorised persistence observed within the audited boundary; control tests pass; customer-log cross-reference verified |
| Detection | Per-identifier precision and recall with uncertainty, reported per release |
| Privacy | Release-specific assessment performed and recorded for every export |
| Authorisation | Gate E evidence present for every export |
| Utility | Utility report per export |
| Reliability | Checkpoint recovery, idempotent retries, no silent partition failure, snapshot drift aborts |
| Negative paths | Missing records, detector blind spots, unauthorised exports, and infrastructure failures cannot pass silently |

Targets to be established on representative data, not claims.

## 10. Decisions

| Decision | Resolution |
|---|---|
| Runtime | Python plus Arrow; typed interfaces; engine separate from orchestration; benchmark before Spark |
| Reversibility | One-way, purpose-scoped keyed pseudonymisation; no vault; anonymisation a separate assessed mode |
| Deletion lists | Generic suppression-policy interface first; DROP workflow later if the business is a covered broker |
| Repository | New, isolated repository with its own security controls and dependency governance |
| Execution environment | Vendor-managed control plane; customer-controlled data plane from the outset; identical core engine in both |

## 11. Next deliverables

1. Phase 0 specification and threat model.
2. De-identification specification: detectors, transforms, key ownership and rotation.
3. Assurance specification: each gate's evidence, the evidence schema, the
   customer-run verifier, the non-retention control tests.
4. Connector SDK contract including snapshot semantics.
5. Backlog and validation suite: labelled and adversarial test sets, negative-path tests, load tests.

## Change log

v3 (review of 2026-10-21):
- Replaced the single "nothing missed" claim with four separate assurances and
  narrowed the non-retention certificate wording.
- Added the two-pass model with a consistent source snapshot and drift abort.
- Replaced count-only reconciliation with a record-level disposition ledger using
  keyed commitments; added atomic publication and referential checks.
- Made the assurance detector independent of the transformation detector.
- Replaced the five-bucket taxonomy with multidimensional classification covering
  nested data, metadata, business sensitivity, and permissions.
- Dropped FF3-1 as default, dropped generic Laplace perturbation, dropped salt-as-protection,
  flagged date-shift sequences as quasi-identifiers.
- Made k-anonymity and l-diversity release-specific rather than universal.
- Expanded the non-retention threat model to logs, telemetry, swap, core dumps,
  egress, connector caches, failed uploads, and hosted import buckets.
- Added an evidence schema rule against unkeyed identifier hashes and scoped the
  delivery registry under its own key.
- Scoped DROP as a separate recurring workflow; narrowed GDPR and ISO claims.
- Reordered phases so assurance arrives before scale; reference pipeline is a
  headless service with a CLI.
- Batch size is adaptive to memory budget.

v2: merged the internal v1 plan with the external roadmap; added portable data
plane, NER budget, enforceable Gate D, key management, token-consistency policy,
utility report, delivery registry, licensing gate.
