# Enterprise Data De-identification Service: Merged Build Plan

Version 2, October 2026. Merges the internal plan (v1) with the external roadmap
reviewed on 2026-10-21, plus the gaps that neither covered.

## 0. What the service is

A privacy-first processing service that securely ingests enterprise data through
connectors or file imports, discovers and classifies sensitive information,
de-identifies it under versioned policy, verifies the whole dataset, and exports
approved output together with independently verifiable evidence.

Two outputs per run: a usable de-identified dataset and a signed processing
assurance report.

The differentiator is provable integrity and privacy assurance, not PII detection.

Deliberately excluded: model training, HIPAA specialisation, data marketplace,
general analytics or BI, data cataloguing, MDM, general ETL. Raw customer data is
never collected for debugging or for improving detectors unless a customer
separately authorises it in writing.

## 1. What we promise, precisely

These qualifications go into contracts, marketing, and acceptance criteria.

**No data stored** means no persistent raw customer data in infrastructure we
operate. Source systems and export destinations necessarily retain data. A
customer-controlled processing environment may use encrypted temporary storage,
and that is the customer's storage.

**No data missed** means complete record accounting for a defined source snapshot:
every input row has a recorded disposition (exported, suppressed, excluded by
opt-out, errored). It does not mean every identifying detail in arbitrary free
text is guaranteed detected. Detection quality is measured and reported per
identifier type, never claimed as absolute.

## 2. Customer workflow

1. Connect or import: databases, warehouses, SaaS, object storage, files.
2. Discover and classify: schema, PII, confidential fields, processing permissions.
3. De-identify: policy-driven suppression, masking, generalisation, tokenisation.
4. Verify and govern: completeness, privacy risk, utility, integrity, non-retention.
5. Release: licensing and purpose check, then export to customer storage plus the
   assurance report. The release fails closed if any gate is unresolved.

## 3. Five non-negotiable capabilities

| Capability | Delivery |
|---|---|
| Secure ingestion | Connectors and imports with scoped, read-only access |
| High-quality de-identification | Structured and free-text detection, configurable transforms, risk assessment, utility report |
| Data completeness | Every record in the snapshot accounted for, including errors and exclusions |
| Zero vendor-side raw retention | Enforced by infrastructure, evidenced from the customer's own logs |
| Privacy and governance | Policies, authorisation, lineage, release controls, auditability, licensing validation |

## 4. Architecture

Control plane and data plane are separated. The control plane is hosted by us and
stores no record values: tenants, connector references, policies, jobs, approvals,
manifests, audit metadata, key references. The data plane is a portable container
that runs either in our isolated environment or, as the high-assurance default,
inside the customer's own cloud account.

```
 Sources                     Data plane worker (ephemeral, portable container)          Sinks
 Postgres/MySQL  ─┐          ┌────────┐  ┌──────────┐  ┌───────────┐  ┌───────────┐     ┌─ customer object storage
 Snowflake/BQ    ─┤ read-only│ Ingest │─▶│ Classify │─▶│ Transform │─▶│ Gates     │─────┤  customer warehouse
 S3/GCS/Azure    ─┼─────────▶│ Arrow  │  │ schema + │  │ policy    │  │ A B C D   │     └─ authenticated download
 CSV/Parquet/JSON─┘ batches  │ stream │  │ content  │  │ engine    │  │ fail-closed│
                             └────────┘  └──────────┘  └───────────┘  └─────┬─────┘
                                                                           ▼
                                           signed manifest + utility report + audit chain
                                                                           ▼
 Control plane (ours, no record data): tenants, policies, jobs, approvals, manifests,
 delivery registry, key references (KMS), licensing rules.
```

### Design rules
- Raw data lives only in memory inside a worker and is gone when the worker ends.
- Every column must be classified; unclassified is a hard failure.
- Policies are versioned YAML; the policy hash is in every manifest.
- Processing is chunked and partitioned from day one. Each partition is an
  idempotent unit of work with its own reconciliation row. Retries overwrite,
  never double-count.
- The data plane is cloud-agnostic: a container image with thin abstractions for
  queue, object storage, secrets, and KMS. Cloud-specific orchestration
  (Step Functions, Fargate, SQS) is used only in our hosted environment.

### Deployment models

| Model | Advantage | Trade-off |
|---|---|---|
| Vendor-hosted isolated workers | Easy onboarding, file-import MVP | Harder to prove zero retention |
| Customer cloud or VPC processing (default for enterprise) | Data sovereignty, retention evidence from customer logs | More deployment and support work |
| Confidential computing (later) | Hardware runtime attestation | Operational complexity; does not by itself prove the rest of the pipeline retained nothing |

### Stack

| Layer | Choice | Note |
|---|---|---|
| CLI and API | Python, FastAPI | Web UI deferred until the evidence format is stable |
| Metadata and governance | PostgreSQL | No record values, ever |
| Connectors | Pluggable Python connector SDK | Contract: snapshot, partition, read batches, checkpoint |
| Processing | PyArrow + Polars | Columnar, bounded memory |
| Detection | Presidio + custom recognisers; cloud DLP as optional backend | Throughput budget required, see 6.3 |
| Execution (hosted) | ECS/Fargate + SQS + Step Functions | Behind the portable abstraction |
| Execution (customer cloud) | Same container, customer's scheduler or our Terraform module | |
| Secrets and keys | Secrets Manager / Vault + KMS | Per-tenant, per-purpose keys |
| Evidence | Signed manifests, hash-chained audit log, customer cloud activity logs | |
| Web app (later) | Next.js + TypeScript | Phase 5 or later |

Not in the initial build: Kafka, Kubernetes, Spark. Revisit only on benchmarks.

## 5. Classification

Three layers, all recorded in the manifest:
1. Schema catalogue: name and type heuristics (email, phone, national ID, name,
   DOB, address, IP, device ID, account number, geo).
2. Content sampling: regex plus validators (Luhn, email syntax, E.164, IP) over a
   sample of each column. Disagreement with layer 1 is flagged.
3. Free text: NER producing spans with entity types.

Every column lands in exactly one of
`direct_identifier | quasi_identifier | sensitive | free_text | non_identifying`
with a confidence. Low confidence blocks the run until a named operator approves
the mapping; the approval is in the manifest.

## 6. De-identification engine

### 6.1 Transform actions

| Action | Use for | Notes |
|---|---|---|
| suppress | national IDs, exact address, raw names | Drop or null |
| pseudonymize | join keys, emails | HMAC-SHA256, per-tenant per-purpose key, one-way |
| tokenize (FPE) | account numbers that must keep format | FF3-1 |
| generalize | ZIP to ZIP3, age to bands, city to region | Hierarchies in policy |
| date_shift | timestamps | Consistent per-entity shift, intervals preserved |
| bucket / round | income, counts | Configurable bins |
| perturb | numeric measures | Laplace noise, optional DP budget |
| redact_text | notes, tickets | Spans replaced with typed placeholders |
| keep | non-identifying | Must be explicit |

### 6.2 Key management and reversibility
- One-way pseudonymisation is the default. No re-identification vault in v1: it is
  the single hardest thing to prove we do not retain.
- Keys are per tenant and per purpose, held in KMS separately from data (GDPR
  Art. 4(5) pseudonymisation requires separate holding). Rotation supported with
  key version recorded in the manifest.
- Cross-dataset consistency (same person, same token across sources or deliveries)
  is a per-contract, per-buyer policy decision, never a default. Salts are scoped
  to a contract so tokens from different buyers cannot be joined.

### 6.3 Free-text throughput budget
NER is the bottleneck, not columnar transforms. Phase 1 establishes a per-record
latency budget, batched inference, model-size options, and a cost per million
text records before any 10M target is claimed.

## 7. Enterprise assurance subsystem

A separate subsystem with four independent gates. Any unresolved gate fails the
export closed.

**Gate A, input completeness.** Source snapshot descriptor, file inventory,
partition counts, schema manifest, connector checkpoints, rolling content hash of
the input computed while streaming.

**Gate B, transformation integrity.** Per partition:
`rows_in = rows_exported + rows_suppressed + rows_optout + rows_errored`.
Schema preservation, controlled suppression, idempotency, output content hash.

**Gate C, privacy assurance.** Independent residual-identifier scan on the output
using the same detectors as classification (zero hits required), k-anonymity on
quasi-identifiers (default k >= 5, per contract), l-diversity on sensitive
columns, linkage-risk score, measured detector precision and recall per identifier
type on validated test sets.

**Gate D, non-retention evidence.** Enforced, not asserted:
- worker IAM role can write only to the export prefix and the evidence prefix
- read-only root filesystem, no persistent volumes, encrypted ephemeral temp
- egress restricted to source and sink endpoints
- worker lifecycle logged; temp cleanup verified at exit
- evidence is the customer's own cloud activity log (CloudTrail or equivalent),
  cross-referenced in the manifest, so the customer verifies it without trusting us.

**Utility report** (new): per export, column retention, generalisation levels,
null-rate change, and distribution drift versus source for declared analytic columns.

**Run manifest** (JSON, signed with the tenant key): source descriptor and hash,
policy ID, version and hash, classification table, per-column action, per-partition
reconciliation, gate results, utility report, output descriptor and hash, worker
identity, paths written, approvals with identity and time, key versions used.

**Audit log**: hash-chained entries containing counts, hashes, and actions only.

**Delivery registry** (new): which pseudonym hashes went to which recipient under
which contract. Holds no raw values. Required so that a deletion request after
export can be propagated to recipients.

## 8. Regulatory and contractual framework

| Framework | What the platform accounts for |
|---|---|
| CCPA/CPRA | De-identified-information requirements (public commitment, contractual restrictions on re-identification), consumer rights, disclosures |
| California Delete Act | Honour DROP deletion lists at ingest (matching rows dropped and counted, never exported) and propagate to recipients via the delivery registry. Registration applicability depends on actual activity, confirm with counsel |
| Other US state laws | Texas, Oregon, Vermont broker registries; varying sensitive-data and consent rules via policy packs |
| GDPR / UK GDPR | Pseudonymisation with separate keys, anonymisation assessment, DPIA template, Art. 30 records generated from manifests, transfer analysis. Track the EDPB anonymisation guidance; verify its status before citing it to customers |
| NIST SP 800-188, ISO/IEC 27559 | Disclosure-risk-based assessment as the method; 27559 as the control checklist |
| NIST Privacy Framework, SOC 2 | Organisational controls, auditable processing integrity |
| Customer contracts | Source ownership, permitted uses, sublicensing, provenance, redistribution limits |

Licensing validation is a release gate: de-identifying data does not create the
right to license it. Source permission and intended downstream use are checked
before export.

Two external facts to verify with counsel before repeating to customers: the DROP
processing start date and the EDPB draft anonymisation guidelines' status.

## 9. Roadmap

Each phase ends with an independently testable deliverable.

| Phase | Focus | Exit criterion |
|---|---|---|
| 0. Specification and threat model | Data types, workflows, deployment boundary, regulatory applicability, prohibited retention, exact promises from section 1 | Approved spec and threat model |
| 1. Core engine | CSV, JSONL, Parquet; classification layers 1 to 3; transform library; policy loader; chunked idempotent partition contract; NER throughput budget | Correct output on labelled datasets; measured throughput |
| 2. Assurance | Gates A to D, signed manifests, audit chain, utility report, fail-closed release, licensing check | Reproducible evidence; a verifier tool a customer can run |
| 3. Scale | Worker queue, autoscaling, checkpoints, retries, bounded memory | 1M then 10M records with full accounting |
| 4. Connectors and exports | PostgreSQL, S3, then prioritised systems; opt-out list ingest; delivery registry | End-to-end with source-scope reconciliation |
| 5. Hardening | Tenant isolation, customer-cloud Terraform module, RBAC, security testing, monitoring, web UI | Security-reviewed, operationally ready |
| 6. Pilot | Real customer data in customer cloud, independent privacy and performance review | Pilot acceptance against contractual criteria |

Phases 0 to 3 come before a broad connector catalogue.

## 10. Acceptance criteria for the first enterprise release

| Area | Criterion |
|---|---|
| Scale | 10M records across structured and text-heavy datasets, bounded memory |
| Completeness | 100% of snapshot records reconciled per Gate B |
| Retention | No vendor-side persistent raw data in the tested deployment, evidenced from customer logs |
| Detection | Per-identifier precision and recall thresholds set on validated test data, reported per run |
| Utility | Utility report delivered with every export |
| Reliability | Checkpoint recovery, idempotent retries, no silent partition failure |
| Governance | Every export carries verifiable policy, lineage, gate, licensing and execution evidence |
| Security | Isolation, least privilege, credential protection, deletion tests, release authorisation |

These are targets to be established on representative data, not claims.

## 11. Next deliverables

1. Phase 0 system specification and threat model, with section 1 promises as
   acceptance criteria.
2. De-identification specification: detectors, transforms, risk scoring, utility.
3. Assurance specification: exactly how each gate produces evidence without
   retaining raw data, and the customer-run verifier.
4. Connector SDK contract.
5. Build backlog and validation suite: labelled test datasets, load tests, release gates.
