# Enterprise Data De-identification Service
## Architecture Specification v1.2 (consolidated, single source of truth)

Date: 21 October 2026
Status: approved product boundary; proposed technical architecture, subject to Phase 0 validation
Customer: the enterprise data owner (or authorised controller). Never the AI data buyer.
Outcome: help an owner securely connect or import its data, transform it to a specified
de-identification standard, and return privacy-assessed exports plus verifiable
processing evidence to that owner.

This document replaces all earlier plans, specs, reviews, and implication notes.
Items marked **[Phase 0]** are agreed positions that Phase 0 must confirm or revise.
Items marked **[verify]** are external facts to confirm before quoting to owners.

---

## 1. Product purpose and boundaries

### 1.1 Product statement

A managed enterprise data de-identification service. Owners bring data and export
requirements, which may be informed by a specification they received from a buyer.
We help them connect sources to our hosted platform, map requirements to an approved
privacy policy, process the data, verify quality and completeness, and hand back the
approved dataset and evidence. The owner alone decides whether, where, to whom, and on
what terms to sell or deliver it. A buyer specification is a processing input, not a
buyer account or workflow.

In owner-facing material we describe ourselves as a managed de-identification service
provider. Regulatory role follows activity, not labels, but the label should not
confuse an owner's counsel.

### 1.2 Build

Owner onboarding, source connection and import, schema discovery, requirements capture,
policy configuration, de-identification, quality and utility evaluation, completeness
reconciliation, non-retention assurance, secure export to the owner, owner approvals,
and an export assurance bundle.

### 1.3 Do not build

Buyer accounts, logins, portals, directories, APIs, discovery, deal matching, pricing,
contracting, listings, marketplace, buyer dashboards, buyer transaction tracking,
delivery to buyer systems, downstream buyer deletion orchestration, buyer or recipient
registry. No hidden buyer roadmap. Also excluded in v1: HIPAA-specific workflows,
universal identity graphs, general ETL, BI, catalogue or MDM, arbitrary media,
generic differential privacy claims, re-identification vault, model training.

### 1.4 Operating assumptions

1. The owner is the contracting customer and controls source and destination.
2. We operate a secure hosted platform with assisted setup. Owner-cloud deployment is
   an optional enterprise requirement, not the default.
3. An owner may submit a buyer's format, schema or privacy requirements as a document
   or structured spec. This does not authorise any interaction with the buyer.
4. A technically de-identified export does not establish legal permission to sell it.
5. We cannot promise perfect PII detection or absolute proof that raw bytes never
   existed in any storage system.

### 1.5 The four assurances, kept separate

| Assurance | Establishes | Does not establish |
|---|---|---|
| Schema coverage | Every discovered field, incl. nested, received a policy decision | That every value inside a field was detected |
| Processing completeness | Every record in the verified snapshot has one disposition, with loss and duplicate detection | Anything about detection quality |
| Detection effectiveness | Measured precision and recall per identifier type on named benchmark sets, with uncertainty | Perfect detection in arbitrary text |
| Non-retention | No unauthorised raw-data persistence observed within the audited boundary under documented controls and tests | That nothing was ever stored anywhere |

Plus one legal gate: owner authorisation (provenance, lawful basis, purpose, permission
to export) was recorded and checked. De-identified is not the same as anonymous,
legally distributable, or free of confidential business content.

---

## 2. Owner journey

1. **Intake and permissions.** Authorised personnel, source systems, access scope,
   commercial-use rights claims, jurisdictions, supported data types.
2. **Output specification.** Owner provides export shape, required fields, utility
   tolerances, privacy posture, optional buyer-provided spec, intended downstream use.
   Operator converts to an owner-approved machine-readable policy.
3. **Connect or import.** Minimum-scoped read-only access, or upload into the secure
   ingestion path. Hosted raw staging is disclosed when it occurs.
4. **Discover and plan.** Snapshot inventory, nested-field and text classification,
   aggregate profiling, restricted or unsupported content, metric selection,
   approval-ready policy shown to the owner.
5. **Process.** Partitioned Arrow transforms, job-scoped pseudonyms where justified,
   no external AI model calls by default, protected internal staging.
6. **Validate independently.** Snapshot scope, dispositions, policy execution,
   residual privacy and confidentiality risk, utility, authorisation, non-retention.
7. **Owner approval and export.** Approved output exposed only to the owner or its
   destination, with manifest, assurance report, warnings. Authenticated downloads
   carry an expiry; owner-controlled destinations are the owner's to manage.
8. **Close.** Verify staging cleanup, emit closure evidence, retain only job-level
   metadata and evidence, offer owner-controlled re-run.

The service never needs the buyer's identity, price, deal status, contract or
destination. A buyer named in an owner's spec is a sanitised owner-provided document
reference, not a buyer record.

---

## 3. Architecture and deployment

```
            ENTERPRISE DATA OWNER (the only customer)
   sources / imports | output spec | approvals | receives export
                          │ scoped connection or import
                          ▼
   ┌──────────── OUR HOSTED SERVICE ─────────────────────────────┐
   │ CONTROL PLANE: intake, connector refs, policy, jobs,         │
   │ approvals, sanitised manifests, evidence, audit records.     │
   │ Never raw values. Never a person-token ledger.               │
   │                     │ signed execution spec                  │
   │                     ▼                                        │
   │ DATA PLANE (portable OCI container, tenant-isolated):        │
   │ snapshot → classify/plan → Arrow partitions → transform      │
   │ → protected staging → independent assurance                  │
   │ → fail-closed release controller                             │
   │        owner export ◄──┤ signed assurance bundle             │
   │                        └── verified staging cleanup          │
   └──────────────────────────────────────────────────────────────┘
                          ▼
            OWNER RECEIVES EXPORT. Any sale is outside this platform.
```

### 3.1 Deployment profiles and the claim each supports

| Profile | Behaviour | Truthful claim |
|---|---|---|
| HOSTED_STREAM (preferred for connectors) | Source batches pulled directly into isolated processing memory; transformed staging may persist temporarily; source not durably staged | No persistent raw input in approved application storage, within audited SDK, system and network boundaries; limitations documented |
| HOSTED_IMPORT (large uploads) | Files persist in a segregated encrypted intake bucket with short enforced lifecycle and verified cleanup | Temporary raw vendor-side storage occurred; scope, retention, deletion, logging and backup or versioning limits disclosed. Never "raw data was never stored" |
| OWNER_CLOUD (enterprise option) | Same signed container in the owner's cloud with owner storage, keys, telemetry | No raw persistence in vendor-managed infrastructure within verified boundaries; owner staging may exist |

Intermediate state is unavoidable for global privacy analysis, resumability, snapshot
consistency, and atomic multi-file export. Intermediate records stay protected and are
never considered de-identified because a first transformation ran.

### 3.2 Multi-tenancy **[Phase 0]**

Per-tenant keys; no worker shared across tenants within a run; tenant-scoped
quarantine, queues and evidence stores; tenant-isolation tests in section 17.

### 3.3 Operator access to raw data **[Phase 0]**

Default: none. Uncertain-field review is done by an owner-side reviewer seeing samples,
or via synthetic and redacted previews. Break-glass access is owner-consented,
time-boxed, and logged into the job evidence. Minimal review tooling is in Phase 1.

### 3.4 External assurance of the hosted platform

In hosted profiles every signing identity belongs to us, so independent assurance is
independent code, not an independent party. The owner's trust anchor is external:
SOC 2 Type II, penetration testing, and auditor access to infra-verifier evidence.
Audit readiness is a Phase 4 exit criterion.

---

## 4. Module boundaries

| Module | Responsibility | Durable state allowed |
|---|---|---|
| owner-api / operator CLI | authenticated jobs, imports, setup, approvals, exports | owner and job metadata |
| spec-intake | owner requirements to typed rules; conflict flags | policy spec, sanitised reference |
| connector-sdk | source auth, snapshot planning, inventory, bounded extraction | scoped refs and checkpoints |
| discovery-classifier | nested-field enumeration; personal, secret, confidential detection | controlled profiles |
| policy-compiler | type-check and freeze YAML to signed job policy | digest, approvals |
| transform-engine | Arrow streaming, redaction, generalisation, controlled tokens | memory and approved temp outputs |
| reconciler | partition dispositions, loss and duplicate detection | protected counts, commitments, source proof refs |
| privacy-validator | independent detectors and dataset-wide disclosure risk | controlled risk statistics |
| utility-validator | schema conformance, field survival, distribution and join fidelity | aggregates reviewed for leakage |
| rights-preflight | owner's asserted authority to access and process | declarations, approvals |
| release-controller | owner approval, destination allowlist, gates, atomic publish | release metadata, signed manifest |
| infra-verifier | independently verify permissions, logging, storage, evidence visibility | config, control results, audit pointers |
| evidence-signer | sign source, processing, assurance and release evidence separately | signatures, non-sensitive pointers |

No module holds buyer profiles, recipient APIs, buyer notifications or buyer deletion
interfaces. Queue payloads and job specs carry references only, never row data.

---

## 5. Connectors, snapshots and scaling contract

Initial sources: CSV, JSONL, Parquet; S3-style object storage; PostgreSQL. Later, by
owner demand: MySQL, Snowflake, BigQuery, GCS, SFTP, selected SaaS.

Connector interface: `plan_snapshot`, `enumerate_partitions`, `stream_partition
(partition, max_bytes)`, `verify_snapshot`, `close`. Snapshot semantics are explicit:
immutable object versions with checksums; database snapshot or export ID with keyset
pagination (never OFFSET over changing tables); warehouse time-travel or export
version; SaaS deterministic pagination with replay boundary.

A SaaS API without replay or a bounded snapshot cannot receive a complete-snapshot
assurance. Use an owner-authorised immutable extract or mark the completeness boundary
explicitly. Fail closed when a strong completeness guarantee was contracted and cannot
be met. Database snapshot lifetime **[Phase 0]**: default to owner-authorised immutable
extract when processing time exceeds what the database can hold a snapshot for;
otherwise bounded snapshot with abort on expiry.

Workers process Arrow partitions under adaptive byte and memory limits, not fixed row
counts. Each partition has a deterministic logical ID and idempotent destination key;
retries never duplicate output; attempt IDs are distinct from partition IDs.
ERROR-disposition retries re-read the source partition rather than caching failed rows.
Container workers plus a simple queue first; measure before Spark. NLP throughput is
benchmarked separately: ten million short rows and ten million long documents are
different workloads.

---

## 6. Owner-supplied output specifications

A specification is owner-approved, job-scoped: output schema, format, required
non-sensitive fields, excluded fields, acceptable utility loss, allowed linkage within
the owner's approved export set, privacy posture. It may reflect a buyer's preferences;
the buyer is not a system actor.

1. Ingest under owner authorisation. Original spec documents may contain buyer names
   and terms: store under tenant encryption, owner-deletable, excluded from telemetry,
   with a retention period.
2. Normalise to a typed versioned `OutputSpec`.
3. Resolve conflicts against mandatory privacy, confidentiality and rights rules.
   External requirements never bypass them.
4. Compile to `DeidentificationPolicy`; preview to the owner with synthetic or redacted
   samples; collect explicit approval.
5. Record `output_spec_digest`, `compiled_policy_digest` and approvals in evidence.

No hardcoded AI-training formats, buyer connectors, or assumptions about a purchaser's
workload.

---

## 7. Classification and transformation

### 7.1 Classification

Multidimensional: structure (scalar, array, nested object, free text, unsupported
binary); personal information (direct, quasi, sensitive, none detected); enterprise
confidentiality (public, internal, confidential, secret or credential); source-use
permission (allowed, restricted, unknown); action and confidence. Schema heuristics
plus validators and content profiling plus local NER and custom recognisers. Sampling
discovers risk; the transform inspects every value the policy covers. New fields and
unsupported content fail closed. Personal values are separated from business
confidential content such as credentials, deal terms, and internal plans.

### 7.2 Actions

suppress, redact, bucket, generalise, owner- or job-scoped one-way pseudonymisation,
justified date shift (flagged as quasi-identifier because sequences survive), explicit
keep. Not in v1: generic noise or DP, reversible vault, automatic FF3 or FF3-1
(removed from NIST's 2025 draft of SP 800-38G rev 1 **[verify]**), universally stable
identities. Each action is validated YAML compiled to a strict typed policy; digests
are immutable per run. Priority: prohibit export > suppress or reject > redact or
generalise > purpose-scoped tokenise > explicit keep.

### 7.3 Pseudonymisation and keys

HMAC-SHA-256 or a vetted keyed PRF over a canonicalised input with context (field
namespace, tenant, purpose, key version, normalised value); no concatenation ambiguity;
truncation only after collision analysis. Tokens stable only within one owner-approved
job, dataset family or project when joins are required; projects unlinkable by default.
Tokens are pseudonymisation, never asserted anonymisation.

Key custody **[Phase 0]**: default per-tenant keys in our KMS; owner-held keys (BYOK)
as the enterprise option. Pseudonymisation, record-commitment and signing keys are
distinct with separate rotation. Rotation states whether historical linkage breaks or
is preserved via an owner-controlled bridge. Key references and versions are persisted,
never keys.

---

## 8. Two-pass execution and gates

**Pass 1, discover and plan.** Stable inventory; classification; output spec;
protected aggregate risk profiles; operator and owner review; approved policy; frozen
snapshot and policy digests. If repeatable reads are unavailable, require an
owner-authorised extract or staging profile.

**Pass 2, execute.** Re-read the same snapshot; streamed Arrow batches; policy
transforms; dispositions and protected commitments; stage in non-public quarantine;
safe retry and reconcile.

**Global validation (a third pass over staged output).** Source coverage; record
identity and duplicate accounting; residual personal and confidential data;
dataset-wide linkage risk including cumulative risk across all prior releases of the
same source by this owner (default-on, since the platform holds that metadata);
privacy posture; utility versus spec; infrastructure controls; owner approval.

**Publication.** Stage everything, verify independently, sign the gate decision,
publish through an owner-controlled pointer or commit manifest. A partial file never
appears as an approved delivery. Destinations without atomic publication need a
documented adapter-specific protocol.

### 8.1 Gates

| Gate | Evidence | Fail condition | Kind |
|---|---|---|---|
| A. Source scope | immutable snapshot manifest, partition plan, connector receipts | missing or changed scope | mandatory |
| B. Processing accounting | unique record dispositions, counts, partition commits | missing, unexplained duplicate, inconsistent count | mandatory |
| C. Privacy and confidentiality | independent scanner, global risk, approved exceptions | unacceptable residual identifiers, secrets or risk | mandatory |
| D. Output utility | requested schema, field survival, distributions, joins, file validity | misses owner spec or tolerances | advisory **[Phase 0]**: produces a warning the owner acknowledges in release approval; acknowledgment never relaxes C |
| E. Non-retention controls | independently collected storage, network, logging, cleanup evidence | unapproved persistence, or visibility gap where a strict claim was contracted | mandatory |
| F. Owner authorisation | access-to-process evidence pre-ingest; export approval at release | missing rights or approval, or illegal policy request | mandatory |

All mandatory gates must pass. An operator cannot override a compliance or security
failure. Exceptions require a separate scope and authorised review.

---

## 9. Reconciliation and evidence

Counts alone are insufficient. Each logical record is identified by a source-stable
position or key plus a job-scoped protected commitment, including identical content at
distinct legitimate locations. Each in-scope input has exactly one terminal
disposition: `EXPORTED`, `SUPPRESSED`, `EXCLUDED_BY_OWNER_POLICY` (covers owner
deletion or opt-out lists), `ERROR`. Unresolved `ERROR` blocks export unless the
contract defines a narrower partial scope.

Commitment construction **[Phase 0]**: per-partition keyed row commitments aggregated
into a Merkle root or a keyed multiset hash, multiplicity preserved, commitment key
distinct from the pseudonymisation key. Only roots and counts leave the worker.

Partition evidence: source receipt, attempt and retry IDs, counts, commitment root,
transform version, output content commitment, execution identity. Source evidence is
generated or verified by the connector or owner environment where feasible, not taken
solely from the worker.

Four separately signed evidence sets: source scope, transformation execution,
independent assurance, final owner release. An owner-run verifier checks signatures,
partition coverage, ledger totals, output hashes and versions, policy versions, release
state, and evidence availability, and distinguishes unsupported evidence from failed
checks. Signatures attest provenance of evidence, not truth of every claim.

No unsalted hash of names, emails, IDs, low-entropy values or sample text enters job
metadata or telemetry.

Truthful claim: "All records in the specified, verified extraction scope were
reconciled." Not: "every record in the organisation was discovered."

### 9.1 Evidence artifacts

`SourceManifest`, `PolicyManifest`, `PartitionManifest`, `PrivacyUtilityReport`,
`InfrastructureEvidence`, `ReleaseManifest`, each with a stable schema version and
canonical signing. Gate results use explicit statuses such as `PASS`,
`PASS_WITH_DOCUMENTED_SCOPE`, `VERIFIED_WITH_LIMITATIONS`, `NON_RETENTION_UNVERIFIED`,
`FAIL`. Zero residual findings are not proof that all PII was found.

---

## 10. Privacy and utility assurance

The privacy engine does not re-run the transformation detector. It uses independent
rules and model components, adversarial cases, protected owner dictionaries, and
human-labelled samples where appropriate. Precision and recall are reported on named
benchmark populations, never as if per-export recall were known without ground truth.

Release findings: residual patterns; sampled-review results; uniqueness and
quasi-identifiers; external linkage assumptions; cumulative risk from prior owner
exports; uncertainty and limitations. k-anonymity and l-diversity are optional
context-specific tests, never universal certification, and there is no universal k.
`PSEUDONYMIZED` and `ASSESSED_DEIDENTIFIED` are distinct output classifications; a
legal anonymisation claim requires contextual assessment and qualified review.

Utility is measured against the owner's spec: retained columns, categorical and numeric
drift, suppressed rows, interval fidelity, join validity, useful text remaining. The
gate decision object has no utility input; commercial value never offsets privacy risk.

---

## 11. Non-retention and secure hosting

**Controls.** Tenant and job segregation; narrowly scoped workload identities;
encrypted transport and storage; secret manager and KMS; allow-listed source and export
endpoints; deny arbitrary egress; no external LLM or DLP API by default; read-only root
filesystems; restricted mounts, swap, core dumps and diagnostic retention; allow-listed
structured telemetry; no raw values in traces or troubleshooting; short-lived
credentials; staged-data lifecycle rules; image signing, SBOM, vulnerability management.

**Evidence.** Infrastructure-side storage inventories and access logs; tested egress
denies; image identities; temporary object and version inventory with cleanup checks;
intake bucket lifecycle audit; job-level worker permissions; independently observed
policy snapshots; logging coverage and limitations recorded. A worker's self-reported
written paths are never sufficient.

**Hosted-import truth.** Uploaded raw objects exist temporarily in the intake bucket.
The report states retention period, encryption, access policy, deletion status, and
residual version or backup limitations. Owners requiring zero durable vendor staging use
HOSTED_STREAM or OWNER_CLOUD.

---

## 12. Owner export and lifecycle

Destinations: authenticated owner download, owner-controlled S3, GCS or Azure storage,
owner-authorised database or warehouse. No buyer delivery connectors.

Two deliverables per job: the clean export package (data files, machine-verifiable
manifest, output classification and limits) and the owner assurance package
(reconciliation, policy digests, privacy and confidentiality findings, utility versus
spec, authorisation records, non-retention observations, integrity, cleanup evidence).
A role-scoped subset the owner may forward onward is a formatting option, not a feature.

The platform retains only owner, job and release metadata under a defined schedule. No
recipient registry, no buyer profiles, no cross-owner token graph, no tracking of sales
or buyer possession.

---

## 13. Legal and commercial responsibility

Before source access, the owner identifies its authority to permit processing and
applicable restrictions. We still evaluate our own obligations; an owner assertion is
evidence, not a waiver. Before release, the owner approves the exact output and
destination; we independently check policy satisfaction and prohibited content.

**Our role.** Processor under GDPR Art. 28 and service provider under CCPA. Required
artifacts **[Phase 0 or 4]**: data processing agreement template, sub-processor list,
DPIA input package for owners, procedure for owner deletion requests against retained
evidence. As a processor returning output to the owner and selling nothing, we are
unlikely to be a data broker under California, Texas, Oregon or Vermont law; the owner
may be. Counsel confirms. DROP is out of scope; the owner may supply its own deletion
list, applied at ingest as `EXCLUDED_BY_OWNER_POLICY` with evidence.

**Billing model [Phase 0].** If we charge for processing only, the incentive conflict
is small. If we take a share of the owner's sale, release approval is separated from
commercial roles, thresholds are set per owner contract, and the gate decision has no
utility input in code (already required by section 10).

We never represent pseudonymous data as anonymous or imply an export is automatically
legal to resell. This document is architecture, not legal advice.

---

## 14. Interfaces and job state

```python
class OwnerOutputSpec: ...          # schema, format, utility constraints, purpose, release mode

class SourceConnector:
    def plan_snapshot(self, selection): ...
    def enumerate_partitions(self, snapshot): ...
    def stream_partition(self, partition, max_bytes): ...
    def verify_snapshot(self, snapshot): ...
    def close(self): ...

class DeidEngine:
    def transform(self, batches, compiled_policy, key_provider): ...

class AssuranceEngine:
    def assess(self, staged_release, source_manifest, compiled_policy): ...

class OwnerExportSink:
    def stage(self, partition, batches): ...
    def publish(self, signed_gate_decision, owner_destination): ...

class ManifestVerifier:
    def verify(self, bundle, output_objects): ...
```

Typed models for `JobSpec`, `SourceManifest`, `CompiledPolicy`, `PartitionRef`,
`PartitionManifest`, `GateDecision`, `ReleaseManifest`. No handwritten success flag
bypasses gates; the publisher consumes a signed validated gate decision. CLI and API
drive the same state machine.

```
DRAFT -> AUTHORIZED_TO_ACCESS -> SNAPSHOTTED -> DISCOVERED -> POLICY_APPROVED
  -> PROCESSING -> STAGED -> ASSURANCE_REVIEW -> OWNER_APPROVED
  -> PUBLISHED_TO_OWNER -> CLOSED | CLOSED_WITH_EXCEPTION

Failure states from any pre-publication state: BLOCKED, QUARANTINED, FAILED, CANCELED.
A published release is immutable; correction is an explicit successor event.
```

Authorisation to access and process is separate from authorisation to release the
exact output.

---

## 15. Scale and budgets

Primary scaling variable is bytes and text token volume, not row count. Benchmark
structured, text-heavy and mixed relational datasets. Report rows/s, MB/s, tokens/s,
peak memory, elapsed time, cost per job, NLP share of compute, evidence overhead,
recovery behaviour, at 1M then 10M rows. Python, PyArrow, Polars; deterministic local
transforms; Presidio plus custom recognisers; bounded-memory containers; simple async
queue. Partition and idempotency contract from day one; independent validation from the
outset. Defer Spark, Kafka, Ray, Kubernetes and connector catalogues until load tests
show need. No SLA on 10M text-heavy records before representative benchmarking.

---

## 16. Engineering phases

| Phase | Build | Exit criterion |
|---|---|---|
| 0. Product and threat model | Hosted retention modes; owner workflow; the [Phase 0] decisions in this document as ADRs; connector semantics; policy and evidence schemas; processor artifacts; billing model | Approved threat model, ADRs, negative test list |
| 1. Reference owner job | New repository; signed container; typed package; CLI and headless API; CSV, JSONL, Parquet streaming; Arrow partition contract; rules-first classification; policy compiler; suppress, redact, generalise, HMAC; private staging; disposition manifests; minimal owner-side review flow; synthetic data generator | Reconciled synthetic end-to-end export with evidence; fail-closed policy tests; no unsupported passthrough; no buyer surface |
| 2. Enterprise assurance | Independent scanner; global risk and utility; infra-verifier; canonical signed manifests; offline verifier; owner approvals; atomic publication | Every negative test blocks publication or records an approved exception |
| 3. Hosted production and scale | Owner onboarding; S3 and PostgreSQL connectors; hosted import isolation; queue and parallel workers; per-tenant KMS; lifecycle; 1M then 10M benchmarks | Documented throughput, complete job evidence, tenant-isolation tests pass |
| 4. Enterprise readiness | SOC 2 Type II readiness; penetration test; RBAC; operational tooling; OWNER_CLOUD module; further connectors; pilot reports | Independent security review and one owner pilot accepted |
| 5. Demand-driven | Connectors, lawful anonymisation methods, policy packs, optional owner dashboard | Prioritised by owner demand |

A managed, human-assisted initial workflow is intentional. A buyer-facing surface is
permanently out of scope.

### 16.1 Repository layout

```
enterprise-deid/
  docs/architecture.md  docs/threat-model.md  docs/adr/
  schemas/{job,policy,manifest}.schema.json
  src/deid/{api,cli,connectors,classification,policy,transform,validate,
            reconcile,evidence,release,crypto,adapters}
  tests/{unit,property,fixtures,negative,integration,load,security}
  deploy/{local,aws,azure,gcp}
  examples/{policies,synthetic-data,manifests}
```

---

## 17. Negative-path acceptance tests

- Source changes between discovery and transformation: abort or re-snapshot.
- Connector omits a partition: no complete-processing certification.
- 100 duplicates offset 100 dropped records with matching totals: accounting fails.
- New field or nested property appears: block until classified and approved.
- Planted PII or secret missed by primary scanner: independent test exposes it; no
  perfect-recall claim.
- Owner's requested format retains prohibited content: block or require revised spec.
- Worker writes outside allowed paths or contacts unapproved endpoint: denied, logged.
- Hosted upload remains after retention deadline: cleanup evidence fails, escalates.
- Export crashes mid-way: no PUBLISHED_TO_OWNER, no partial approved package.
- Operator publishes without exact owner approval: rejected.
- Buyer-specific identity, login, destination, transaction or billing feature appears:
  rejected as out of scope.
- Verifier sees signature mismatch, missing snapshot proof, incomplete ledger or changed
  object: verification fails.
- Per-partition anonymity passes but global anonymity fails: global gate blocks.
- Cross-project stable-token request: denied absent approved scope.
- Tenant A's job can observe tenant B's queue, staging, keys or evidence: fails.
- Utility gate D fails and owner acknowledges: release proceeds only if C still passes.

Quality thresholds for recall, risk, utility, throughput, cost and evidence are set on
representative pilot data; no universal numbers are invented.

---

## 18. Decisions

**Locked.** Owner is the only customer; managed hosted service first; owner-cloud
optional; owner supplies requirements; all output to owner; no buyer integrations or
marketplace; Python and Arrow; headless engine with CLI and API; partition and
reconciliation design from day one; independent assurance; signed owner evidence;
key-scoped one-way pseudonymisation; no re-identification vault; utility gate advisory,
privacy gate mandatory.

**Phase 0, with real owners.** Supported dataset types; connector auth patterns;
acceptability of hosted raw upload; retention windows; first two connectors; text and
structured mix; project-wide linkability needs; output classification and
disclosure-risk methodology; operator versus owner approvals; billing model; and every
item marked [Phase 0] above.

---

## 19. References **[verify before quoting]**

- NIST SP 800-188 (2023), De-Identifying Government Datasets. Technical guidance, not certification.
- ISO/IEC 27559:2022, privacy-enhancing de-identification framework. Checklist, not certification.
- GDPR Art. 4(5), 28, 30, Recital 26.
- EDPB Guidelines 02/2026 on Anonymisation: draft under consultation as of 21 Oct 2026.
- California Civil Code §1798.140, definition of deidentified information.
- NIST SP 800-38G rev 1, second public draft (2025): FF3 family removed.

This is architecture, not a guarantee of anonymity, rights clearance, or compliance.
