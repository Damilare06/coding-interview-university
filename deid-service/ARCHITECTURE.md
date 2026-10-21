# Enterprise Data De-identification Service
## Architecture Specification v1.2

21 October 2026. Product boundary approved; technical architecture proposed, subject to
Phase 0 validation. This is the single source of truth and replaces all earlier documents.
Items marked **[open]** are decided in Phase 0.

---

## 1. Product

A managed de-identification service for enterprise data owners. The owner connects or
imports data to our hosted platform, we transform it to the owner's specified standard,
verify completeness and privacy, and return the export plus signed evidence to the owner.
What the owner does with the export afterwards is outside the platform.

**Customer:** the data owner only. A buyer's requirements may arrive as part of the
owner's specification; the buyer is never a system actor. No buyer accounts, delivery,
registry, portal, or marketplace, ever.

**Also excluded in v1:** HIPAA-specific workflows, model training, re-identification
vault, universal identity graph, general ETL, BI, catalogue or MDM, media files,
generic differential privacy.

**What we can truthfully promise.** Four separate assurances, never collapsed:

| Assurance | Establishes | Does not establish |
|---|---|---|
| Schema coverage | Every discovered field, including nested, received a policy decision | That every value inside a field was detected |
| Processing completeness | Every record in the verified snapshot has exactly one disposition, with loss and duplicate detection | Anything about detection quality |
| Detection effectiveness | Measured precision and recall per identifier type on named benchmark sets | Perfect detection in free text |
| Non-retention | No unauthorised raw-data persistence observed within the audited boundary under documented controls and tests | That nothing was ever stored anywhere |

Plus owner authorisation: provenance, lawful basis, purpose and permission to export are
recorded and checked. De-identified does not mean anonymous, legally resellable, or free
of confidential business content. We are a processor; the owner is the controller.

---

## 2. Owner journey

1. **Intake.** Authorised personnel, sources, access scope, rights claims, jurisdictions,
   supported data types.
2. **Output specification.** Schema, format, required fields, utility tolerances, privacy
   posture, intended use. Normalised to a typed `OutputSpec`, compiled to a policy,
   previewed with synthetic or redacted samples, approved by the owner.
3. **Connect or import.** Read-only scoped connector, or upload to the intake path.
4. **Discover and plan.** Snapshot inventory, classification, aggregate profiling,
   approval-ready policy.
5. **Process.** Partitioned Arrow transforms into protected staging.
6. **Validate.** Independent assurance over the whole staged release.
7. **Approve and export.** Owner approves the exact output and destination; atomic publish
   with manifest and assurance package.
8. **Close.** Verify staging cleanup; retain only job metadata and evidence.

Authorisation to access and process (step 1) is separate from authorisation to release
the exact output (step 7).

---

## 3. Architecture

```
 OWNER: sources / imports | output spec | approvals | receives export
                         │
   ┌─────────────────── OUR HOSTED SERVICE ───────────────────────┐
   │ CONTROL PLANE: intake, connector refs, policies, jobs,        │
   │ approvals, manifests, evidence, audit. Never raw values.      │
   │                     │ signed execution spec                   │
   │ DATA PLANE (portable OCI container, tenant-isolated):         │
   │ snapshot → classify/plan → Arrow partitions → transform       │
   │ → protected staging → independent assurance → fail-closed     │
   │ release → owner export + signed evidence → verified cleanup   │
   └───────────────────────────────────────────────────────────────┘
```

### 3.1 Deployment profiles

| Profile | Behaviour | Claim it supports |
|---|---|---|
| HOSTED_STREAM (default for connectors) | Source batches pulled straight into processing memory; transformed staging may persist temporarily | No persistent raw input in application storage, within audited boundaries |
| HOSTED_IMPORT (file uploads) | Raw files persist in a segregated encrypted intake bucket with enforced lifecycle and verified cleanup | Temporary raw storage occurred; scope, retention, deletion and backup limits disclosed |
| OWNER_CLOUD (enterprise option) | Same signed container in the owner's cloud with owner storage and keys | No raw persistence in vendor infrastructure |

Intermediate state is unavoidable for global analysis, resumability and atomic export.
It is protected as personal data until released.

### 3.2 Tenancy and access
- Per-tenant keys, queues, staging and evidence stores; no worker serves two tenants in
  one run. Isolation is tested (section 11).
- Operators have no access to raw values. Uncertain-field review is done by an owner-side
  reviewer, or on synthetic and redacted previews. Break-glass access is owner-consented,
  time-boxed and logged into job evidence.
- Queue payloads and job specs carry references, never rows.

### 3.3 External assurance
In hosted profiles every signing key is ours, so independent assurance is independent
code, not an independent party. The owner's trust anchor is external: SOC 2 Type II,
penetration testing, and auditor access to infrastructure evidence (Phase 4).

---

## 4. Modules

| Module | Responsibility | Durable state |
|---|---|---|
| owner-api / CLI | jobs, imports, approvals, exports | owner and job metadata |
| spec-intake | owner requirements to typed `OutputSpec` | spec and digest |
| connector-sdk | source auth, snapshot, inventory, bounded extraction | scoped refs and checkpoints |
| classifier | nested-field enumeration; personal, secret, confidential detection | profiles |
| policy-compiler | validate and freeze YAML to a signed typed policy | digest, approvals |
| transform-engine | Arrow streaming transforms | memory and approved staging |
| reconciler | dispositions, loss and duplicate detection | counts, commitment roots |
| privacy-validator | independent detectors, dataset-wide risk | risk statistics |
| utility-validator | conformance to spec | aggregates |
| rights-preflight | owner's asserted authority | declarations, approvals |
| release-controller | gates, owner approval, atomic publish | release manifest |
| infra-verifier | independently verify permissions, logging, storage, cleanup | control results |
| evidence-signer | sign the four evidence sets separately | signatures |

---

## 5. Connectors and snapshots

Initial sources: CSV, JSONL, Parquet, S3-style object storage, PostgreSQL. Others by
owner demand.

Interface: `plan_snapshot`, `enumerate_partitions`, `stream_partition(partition,
max_bytes)`, `verify_snapshot`, `close`. Snapshot semantics are explicit per source:
immutable object versions with checksums; database snapshot or export ID with keyset
pagination; warehouse time-travel version; SaaS deterministic pagination with replay
boundary. A source that cannot provide a repeatable boundary gets no completeness
assurance; if one was contracted, the job fails closed.

**[open]** Database snapshot lifetime: owner-authorised immutable extract when
processing exceeds what the database can hold a snapshot for; otherwise bounded snapshot
with abort on expiry.

Partitions have deterministic logical IDs and idempotent destination keys; attempt IDs
are distinct; retries re-read the source rather than caching rows. Batch size adapts to
a byte and memory budget.

---

## 6. Classification and transformation

**Classification** is multidimensional per field: structure (scalar, array, nested,
free text, unsupported binary); personal information (direct, quasi, sensitive, none);
business confidentiality (public, internal, confidential, secret or credential);
source-use permission (allowed, restricted, unknown); action and confidence. Detection
uses schema heuristics, validators and profiling, and local NER with custom recognisers.
Sampling discovers; the transform inspects every value the policy covers. New fields and
unsupported content fail closed.

**Actions:** suppress, redact, bucket, generalise, scoped one-way pseudonymise, justified
date shift (treated as a quasi-identifier), explicit keep. Priority: prohibit > suppress >
redact or generalise > tokenise > keep. Not in v1: noise or DP, FF3-class
format-preserving encryption (removed from NIST's 2025 draft of SP 800-38G rev 1),
reversible tokens. Policies are YAML compiled to a strict typed schema with an immutable
digest per run. External requirements never bypass mandatory privacy rules.

**Pseudonymisation:** HMAC-SHA-256 or a vetted keyed PRF over a canonicalised input with
context (tenant, purpose, field namespace, key version, normalised value). Tokens are
stable only within one owner-approved project; projects are unlinkable by default. Tokens
are pseudonymisation, never anonymisation.

**Keys.** Pseudonymisation, record-commitment and signing keys are distinct, with
separate rotation; references and versions are persisted, never keys. **[open]** Custody:
per-tenant keys in our KMS by default, owner-held keys as the enterprise option.

---

## 7. Execution and gates

**Pass 1, plan:** inventory, classification, output spec, aggregate risk profile,
approved policy, frozen snapshot and policy digests.

**Pass 2, execute:** re-read the same snapshot, stream Arrow batches, transform, record
dispositions and commitments, stage in quarantine.

**Global validation**, a third pass over staged output: coverage, record accounting,
residual personal and confidential content, dataset-wide linkage risk including
cumulative risk across all prior releases of the same source by this owner, utility,
infrastructure controls, owner approval.

**Publication:** sign the gate decision, publish atomically through an owner-controlled
pointer or commit manifest. A partial file is never an approved delivery.

| Gate | Evidence | Fails on | Kind |
|---|---|---|---|
| A. Source scope | snapshot manifest, partition plan, connector receipts | missing or changed scope | mandatory |
| B. Accounting | per-record dispositions, counts, commitments | missing or duplicate record, inconsistent count | mandatory |
| C. Privacy and confidentiality | independent scan, global risk | unacceptable residual identifiers, secrets or risk | mandatory |
| D. Utility | conformance to owner spec | misses spec or tolerances | advisory: owner acknowledges in release approval; never relaxes C |
| E. Non-retention | independently collected storage, network, logging, cleanup evidence | unapproved persistence, or evidence gap where a strict claim was contracted | mandatory |
| F. Owner authorisation | access evidence pre-ingest; export approval at release | missing | mandatory |

No operator override of a mandatory gate. The gate decision object has no utility input.

State machine:
```
DRAFT -> AUTHORIZED_TO_ACCESS -> SNAPSHOTTED -> DISCOVERED -> POLICY_APPROVED
  -> PROCESSING -> STAGED -> ASSURANCE_REVIEW -> OWNER_APPROVED
  -> PUBLISHED_TO_OWNER -> CLOSED | CLOSED_WITH_EXCEPTION
Any pre-publication state may enter BLOCKED, QUARANTINED, FAILED or CANCELED.
A published release is immutable; correction is an explicit successor event.
```

---

## 8. Reconciliation and evidence

Each logical record is identified by a source-stable position or key plus a job-scoped
keyed commitment, so identical content at distinct locations is preserved. Dispositions:
`EXPORTED`, `SUPPRESSED`, `EXCLUDED_BY_OWNER_POLICY` (includes owner-supplied deletion
lists), `ERROR`. Unresolved `ERROR` blocks release unless the contract defines a partial
scope.

**[open]** Commitment construction: per-partition keyed row commitments aggregated into a
Merkle root or keyed multiset hash, multiplicity preserved. Only roots and counts leave
the worker.

Source evidence is produced or verified by the connector or owner environment where
feasible, not solely by the worker. Four evidence sets are signed separately: source
scope, transformation execution, independent assurance, owner release. Artifacts:
`SourceManifest`, `PolicyManifest`, `PartitionManifest`, `PrivacyUtilityReport`,
`InfrastructureEvidence`, `ReleaseManifest`, with stable schema versions and explicit
statuses (`PASS`, `PASS_WITH_DOCUMENTED_SCOPE`, `VERIFIED_WITH_LIMITATIONS`,
`NON_RETENTION_UNVERIFIED`, `FAIL`). An owner-run verifier checks signatures, coverage,
ledger totals, output hashes, policy versions and release state, and distinguishes
unsupported evidence from failed checks. Signatures attest provenance, not truth.

No unsalted hash of identifiers, low-entropy values or sample text enters metadata or
telemetry. Owner spec documents are tenant-encrypted, owner-deletable and excluded from
telemetry.

---

## 9. Privacy and utility assurance

The privacy validator does not re-run the transformation detector. It uses independent
rules and models, adversarial cases, owner dictionaries and labelled samples. Precision
and recall are reported on named benchmark populations only. Findings cover residual
patterns, uniqueness and quasi-identifiers, linkage assumptions, cumulative risk across
the owner's prior exports, and limitations. k-anonymity and l-diversity are optional
context-specific tests with no universal threshold. `PSEUDONYMIZED` and
`ASSESSED_DEIDENTIFIED` are distinct output classifications; the latter requires
contextual assessment and qualified review.

Utility is measured against the owner's spec: retained columns, drift, suppressed rows,
interval fidelity, join validity, text remaining after redaction.

---

## 10. Non-retention controls

Enforced: tenant and job segregation; scoped workload identities with write access only
to staging, export and evidence paths; encryption in transit and at rest; KMS and secret
manager; allow-listed source and export endpoints with all other egress denied; no
external LLM or DLP API by default; read-only root filesystem; swap, core dumps and
diagnostic retention disabled; allow-listed structured telemetry; no raw values in traces;
short-lived credentials; lifecycle rules on every staging location; image signing, SBOM
and vulnerability management.

Evidenced independently of the worker: storage inventories and access logs, tested egress
denies, image identities, temporary object and version inventories with cleanup checks,
intake bucket lifecycle audit, worker permissions, observed policy snapshots, logging
coverage and its gaps.

---

## 11. Negative-path tests

- Source changes between passes: abort or re-snapshot.
- Connector omits a partition: no completeness certification.
- 100 duplicates offset 100 dropped records with matching totals: accounting fails.
- New field appears: blocked until classified and approved.
- Planted PII or secret missed by the primary detector: exposed by the independent one or
  by benchmark; no perfect-recall claim.
- Owner spec would retain prohibited content: blocked.
- Worker writes outside allowed paths or contacts an unapproved endpoint: denied, logged.
- Upload remains past its lifecycle: cleanup evidence fails and escalates.
- Export crashes mid-way: no `PUBLISHED_TO_OWNER`, no partial package.
- Publish without owner approval: rejected.
- Utility gate fails and owner acknowledges: release proceeds only if C passes.
- Per-partition anonymity passes, global fails: blocked.
- Cross-project stable-token request: denied absent approved scope.
- Tenant A can observe tenant B's queue, staging, keys or evidence: fails.
- Evidence tampered or object changed: verifier fails.
- Any buyer-facing feature introduced: rejected as out of scope.

Thresholds for recall, risk, utility, throughput and cost are set on representative
pilot data.

---

## 12. Scale

Primary scaling variable is bytes and text tokens, not rows. Benchmark structured,
text-heavy and mixed datasets at 1M then 10M rows: rows/s, MB/s, tokens/s, peak memory,
cost per job, NLP share of compute, evidence overhead, recovery. Python, PyArrow, Polars,
Presidio plus custom recognisers, bounded-memory containers, a simple queue. Distributed
engines only if load tests show need. No SLA on text-heavy 10M before benchmarking.

---

## 13. Phases

| Phase | Build | Exit |
|---|---|---|
| 0. Threat model and ADRs | Every **[open]** item; connector semantics; policy and evidence schemas; processor artifacts (DPA template, sub-processor list, DPIA input package, deletion procedure for retained evidence); billing model | Approved threat model and ADRs |
| 1. Reference owner job | New repository; signed container; typed package; CLI and headless API; CSV, JSONL, Parquet; Arrow partition contract; rules-first classification; policy compiler; suppress, redact, generalise, HMAC; staging; disposition manifests; owner-side review flow; synthetic data generator | Reconciled synthetic end-to-end export with evidence; fail-closed policy tests |
| 2. Assurance | Independent validator; global risk and utility; infra-verifier; signed manifests; offline verifier; owner approvals; atomic publication | Every negative test blocks or records an approved exception |
| 3. Hosted production | Owner onboarding; S3 and PostgreSQL connectors; intake isolation; queue and parallel workers; per-tenant KMS; lifecycle; benchmarks | Documented throughput; tenant-isolation tests pass |
| 4. Enterprise readiness | SOC 2 Type II readiness; penetration test; RBAC; OWNER_CLOUD module; pilot | Independent security review; one owner pilot accepted |
| 5. Demand-driven | Connectors, anonymisation methods, policy packs, owner dashboard | By owner demand |

Repository layout:
```
enterprise-deid/
  docs/{architecture.md,threat-model.md,adr/}
  schemas/{job,policy,manifest}.schema.json
  src/deid/{api,cli,connectors,classification,policy,transform,validate,
            reconcile,evidence,release,crypto,adapters}
  tests/{unit,property,fixtures,negative,integration,load,security}
  deploy/{local,aws,azure,gcp}
  examples/{policies,synthetic-data,manifests}
```

---

## 14. Decisions

**Locked:** owner is the only customer; hosted service first, owner-cloud optional;
Python and Arrow; headless engine with CLI and API; partition and reconciliation contract
from day one; independent assurance; four separately signed evidence sets; one-way
scoped pseudonymisation, no vault; utility gate advisory, privacy gate mandatory.

**Open for Phase 0:** database snapshot lifetime; key custody; commitment construction;
billing model (processing fee, or revenue share which requires separating release
approval from commercial roles); supported dataset types; first two connectors;
acceptability of hosted raw upload; retention windows; approval roles.

**References, verify before quoting:** NIST SP 800-188; ISO/IEC 27559; GDPR Art. 4(5),
28, 30, Recital 26; EDPB Guidelines 02/2026 (draft, in consultation); Cal. Civ. Code
§1798.140; NIST SP 800-38G rev 1 second draft (FF3 removed).
