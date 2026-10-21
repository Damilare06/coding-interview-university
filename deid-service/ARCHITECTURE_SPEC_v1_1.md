# Enterprise Data De-identification Service
## Consolidated Architecture Specification v1.1 — Data-Owner-First Managed Service

**Date:** 21 October 2026  
**Status:** Approved product boundary; proposed technical architecture, subject to Phase 0 validation  
**Business:** AI data brokerage firm  
**Customer:** The enterprise data owner or authorized data controller, **not the AI data buyer**  
**Primary outcome:** Help an enterprise securely connect or import its data, transform it to a specified de-identification standard, and return privacy-assessed exports plus verifiable processing evidence **to that enterprise**.

---

## 1. Product purpose and boundaries

### Product statement

We operate a **managed enterprise data de-identification service**. Enterprise data owners bring data and export requirements (which may be informed by specifications they received from an AI data buyer). Our team helps them connect their sources to our secure platform, map their requirements to an approved privacy policy, process the data, verify quality and completeness, and hand back the approved datasets and evidence. The **owner independently decides whether, where, to whom, and on what terms to sell or deliver** its data. A buyer specification is a *data-processing input*, not a buyer account or workflow in our system.

### Strict scope

**Build:** owner onboarding, source connection/import, schema discovery, requirements capture, privacy policy configuration, de-identification, quality/utility evaluation, completeness reconciliation, non-retention assurance, secure export **to the owner**, owner approvals, and an export assurance bundle.

**Do not build:** buyer accounts, buyer login/portal, buyer identity directory, buyer-specific APIs, buyer discovery, deal matching, pricing negotiation, contracting, listings, marketplace, training/inference infrastructure, data-buyer dashboards, buyer transaction tracking, delivery directly to buyer systems, downstream buyer deletion orchestration, or a buyer/recipient registry. **No hidden buyer product roadmap.** Commercial brokerage activity, if any, happens outside this processing platform.

**Other exclusions in v1:** HIPAA-specific workflows, universal identity graphs, general-purpose ETL, BI, full data catalog/MDM, arbitrary media (audio/video/images/PDFs unless separately scoped), generic differential privacy claims, and a re-identification vault.

### Explicit operating assumptions

1. The data owner is the contracting service customer and controls the source and final approved export destination.
2. Our firm primarily operates a **secure hosted processing platform**, including assisted setup. A customer-cloud/private deployment is an **optional enterprise requirement**, not a mandatory default.
3. The owner may submit a buyer's format, schema, or privacy requirements as a document or structured job specification. This does not authorize direct interaction with the buyer.
4. An exported dataset being technically de-identified does **not** by itself establish legal permission to sell, sublicense, or redistribute it. Required legal/contractual authorization is independently reviewed and recorded.
5. We cannot truthfully promise perfect PII detection or an absolute proof that raw bytes never existed in any storage system.

---

## 2. Customer journey: managed service

1. **Owner intake and permissions.** Identify authorized company personnel, source systems, access scope, commercial-use rights claims, applicable jurisdictions, and whether the proposed data types are supported.
2. **Receive output specification.** Owner provides desired export shape, required fields, analytical utility, privacy thresholds, optional buyer-provided specification, and intended downstream use. The operator converts these into an owner-approved machine-readable processing policy; the system does not connect to a buyer.
3. **Connect/import.** Owner grants minimum-scoped read-only connector access, authorizes an extraction, or uploads data into the secure ingestion path. Hosted raw staging is explicitly disclosed when needed.
4. **Discover and plan.** Inventory source snapshot, classify nested fields and text, profile aggregate characteristics, identify restricted or unsupported content, select privacy and utility metrics, and show owner an approval-ready policy.
5. **Run processing.** Partition and transform using bounded Arrow batches, consistent owner/project-scoped pseudonyms where justified, no external AI model calls by default, and a protected internal output staging area.
6. **Independently validate.** Confirm snapshot scope and record dispositions, policy execution, residual privacy/confidentiality risk, output utility, rights/release approval, and non-retention control evidence.
7. **Owner approval and export.** Expose approved output only to the owner or owner-controlled destination, with integrity manifest, assurance report, warnings, and a defined expiry/deletion schedule.
8. **Complete job.** Verify cleanup of vendor-hosted staging, emit closure evidence, retain only authorized job-level metadata/evidence, and provide owner-controlled re-run capabilities.

At no point does this service need to know the eventual buyer's identity, price, commercial deal status, contract, or destination system. If an owner's spec mentions a buyer by name, the platform can store a sanitized, owner-provided document reference; this is **not** a buyer record.

---

## 3. Architecture and deployment

```text
                ENTERPRISE DATA OWNER — THE ONLY PRODUCT CUSTOMER
      ┌─────────────────────────────────────────────────────────────┐
      │ Source systems / file imports                               │
      │ Owner-supplied output specification (may originate elsewhere)│
      │ Authorized staff approve policy and receive final exports   │
      └──────────────────────────┬──────────────────────────────────┘
                                 │ secure scoped connection/import
                                 ▼
      ┌──────────────── OUR SECURE HOSTED SERVICE ──────────────────┐
      │ OWNER/OPERATOR CONTROL PLANE                                │
      │ Intake | connector refs | policy | jobs | approvals          │
      │ sanitized manifests | evidence | job-level audit records    │
      │ NEVER raw record values / universal person-token ledger      │
      │                         │ signed execution spec              │
      │                         ▼                                    │
      │ ISOLATED DATA PLANE (portable OCI container)                 │
      │ source snapshot → classify/plan → Arrow partitions           │
      │ → transform → protected staging → independent assurance      │
      │ → fail-closed release controller                             │
      │                         │                                    │
      │ owner export ◄──────────┤ signed assurance bundle            │
      │                         └─── verified staging cleanup        │
      └─────────────────────────────────────────────────────────────┘
                                  │
                          OWNER RECEIVES EXPORT
                                  │
          Any subsequent data sale/delivery is outside this platform
```

### Deployment profiles and truthful non-retention claims

| Profile | Source/staging behavior | What may be claimed |
|---|---|---|
| **HOSTED_STREAM** (preferred for connectors) | Pull approved source batches directly into isolated processing memory. Protected transformed staging may persist temporarily. Source data is not intentionally durably staged. | No persistent **raw** input in approved application storage, subject to audited SDK/system/network boundaries; independent evidence and limitations documented. |
| **HOSTED_IMPORT** (preferred for large file uploads) | Files initially persist in a segregated, encrypted vendor-controlled ingestion bucket with short, enforced lifecycle and verified cleanup. | **Temporary raw vendor-side storage occurred**; disclose scope, retention, deletion, logging, and residual backup/versioning limitations. Never claim “raw data was never stored.” |
| **OWNER_CLOUD** (enterprise option) | Same signed data-plane container runs in the owner's cloud, using owner-controlled storage, keys, and telemetry. | No raw input persistence in **vendor-managed infrastructure**, within verified boundaries; owner-controlled staging may still exist. |

**Rationale:** The user journey is hosted and high-touch; forcing every enterprise to deploy the service in its VPC would create unnecessary sales friction. Preserve portability for larger customers who require sovereignty or a contractual non-retention constraint.

**Intermediate state is unavoidable in some jobs:** global privacy analysis, resumable execution, snapshot consistency, and atomic multi-file exports may require temporary state. All intermediate records remain protected and are never considered de-identified solely because a first transformation ran.

---

## 4. Service/module boundaries

| Module | Responsibility | Durable state allowed |
|---|---|---|
| `owner-api` / operator CLI | authenticated owner jobs, imports, setup, approvals, exports | owner/job metadata only |
| `spec-intake` | convert owner-supplied output requirements to typed rules; flag conflicts | policy spec and sanitized reference |
| `connector-sdk` | source authentication, snapshot planning, inventory, bounded extraction | scoped references/checkpoints, not raw rows |
| `discovery-classifier` | enumerate nested fields; detect personal data, secrets, confidentiality | customer/job-side controlled profiles |
| `policy-compiler` | type-check and freeze YAML to signed job policy | rule digest, approvals |
| `transform-engine` | Arrow streaming, field redaction, generalization, controlled tokens | bounded memory and approved temporary processing outputs |
| `reconciler` | unique partition dispositions and loss/duplicate detection | protected counts, commitments and source proof refs |
| `privacy-validator` | independent detector/rule set and dataset-wide disclosure risk | controlled risk statistics/evidence |
| `utility-validator` | measure schema conformance, field survival, distribution and join fidelity | aggregate metrics reviewed for leakage |
| `rights-preflight` | confirm owner's asserted legal authority to access/process; evidence references | rights declarations, operator/legal approvals |
| `release-controller` | owner approval, destination allowlist, final gates, atomic publish | release metadata and signed final manifest |
| `infra-verifier` | independently verify permissions, logging, storage/evidence visibility | configuration, control results, job-scoped audit pointers |
| `evidence-signer` | sign distinct source, processing, assurance, and final evidence | signatures and non-sensitive manifest pointers |

No service has buyer profiles, buyer account records, recipient APIs, direct buyer notifications, or an end-buyer deletion interface.

---

## 5. Source connectors, snapshots and scaling contract

**Initial source types:** CSV, JSONL, Parquet; S3-style owner source connections and PostgreSQL (expand only based on owner demand). Later: MySQL, Snowflake, BigQuery, GCS, SFTP, select SaaS.

A connector must support: `plan_snapshot(selection)`, `enumerate_partitions(snapshot)`, `stream_partition(partition, max_bytes)`, `verify_snapshot(snapshot)`, and `close()`. Record explicit snapshot semantics: immutable object versions, stable database snapshot or export ID, or a connector-specific repeatable-read mechanism.

**Important:** A best-effort SaaS pagination API without replay or a bounded extraction snapshot cannot receive a “complete source snapshot” assurance. Either use an owner-authorized immutable extract or mark the job's completeness boundary and limitations explicitly. Fail closed when a strong completeness guarantee was contracted but cannot be met.

Workers process Arrow partitions using adaptive **byte/memory** limits, not fixed row counts. Each partition has a deterministic logical ID and idempotent destination key; retries must not duplicate output. Use container workers and a job queue first, measure before adopting Spark. NLP throughput is separately benchmarked because ten million short rows and ten million long documents are different workloads.

---

## 6. Owner-supplied output specifications

A specification consists of **owner-approved, job-scoped requirements**, such as output schema, format (Parquet/CSV/JSONL), required non-sensitive fields, excluded fields, acceptable loss of analytical information, allowed linkage within the owner's approved export set, and a privacy posture.

Sources of requirements: owner form, uploaded requirements document, or an internal operator-entered configuration. They may reflect preferences stated by an external AI data buyer, but **the buyer is not a system actor**.

Processing order:

1. Ingest specification under the owner's authorization; store original specs only as controlled documents according to policy.
2. Normalize into a typed, versioned `OutputSpec` (format, schema, utility tolerances, field handling, release mode).
3. Resolve conflicts against mandatory privacy, confidentiality, and legal/rights rules. External requirements **never** bypass these rules.
4. Compile to `DeidentificationPolicy`, show the owner a preview using only synthetic/redacted samples where possible, and collect explicit owner approval.
5. Record `output_spec_digest`, `compiled_policy_digest`, and approvals in the run evidence.

Avoid hardcoded AI training formats, buyer-specific connectors, or assumptions about a purchaser's downstream AI workload. Only produce the owner's requested export representation.

---

## 7. Classification and transformation

Classification is **multidimensional**: (a) structure (scalar, array, nested object, free text, unsupported binary); (b) personal information (direct/quasi/sensitive/none detected); (c) enterprise confidentiality (public/internal/confidential/secret); (d) source-use permission (allowed/restricted/unknown); (e) transformation action and confidence.

Use schema heuristics plus validators/content profiling plus local text NER/custom recognizers. Sampling helps discover risk, but the full transform must inspect every field/value covered by policy. New schema fields and unsupported content fail closed. Separate personally identifying values from business-confidential details like credentials, deal terms, or proprietary internal plans.

**Initial actions:** suppress, redact, bucket, generalize, owner/job-scoped one-way pseudonymization, optionally justified date shift, and explicit keep. Avoid general noise/DP promises, reversible vaults, automatic FF3/FF3-1 selection, and universally stable identities. Each action is defined in validated YAML compiled to a strict typed policy; versions/digests are immutable per run.

**Token scope:** stable tokens within one owner-approved **job, dataset family, or project** only when utility requires joins. Different owner projects are unlinkable by default. Owner-managed KMS key references, key versions, and rotation strategy are documented. These tokens generally indicate **pseudonymization**, not guaranteed legal anonymization.

---

## 8. Two-pass execution and global release gates

**Pass 1 (discovery/plan):** stable source inventory; classification; owner output specification; protected aggregate risk profiles; operator/owner review and approved policy. If repeatable source reads are unavailable, require an owner-authorized secure extract/staging profile.

**Pass 2 (execution):** reread same snapshot; streamed Arrow batches; policy transforms; partition dispositions and protected commitments; stage output in nonpublic quarantine; retry/reconcile safely.

**Global validation:** independently check source coverage, record identity/duplicate accounting, residual personal/confidential data, dataset-wide linkage risk, privacy posture, utility versus owner-approved spec, infrastructure controls, and rights/owner approval.

**Publication:** stage all output, independently verify, produce signed approval, publish through an owner-controlled access pointer, destination transaction or commit manifest; never allow a partial file to appear as a final approved delivery. Destinations without atomic publication require a documented adapter-specific safe release protocol.

### Required gates

| Gate | Evidence | Fail condition |
|---|---|---|
| A. Source scope | immutable snapshot manifest, partition plan, connector receipts | missing/changed source scope |
| B. Processing accounting | unique source-record dispositions, counts, partition commits | missing, unexplained duplicate, inconsistent count |
| C. Privacy/confidentiality | independent scanner, global risk assessment, approved exceptions | unacceptable residual identifiers/business secrets/risk |
| D. Output quality/utility | requested schema, field survival, distributions, joins, file validity | misses mandatory owner spec or approved utility thresholds |
| E. Non-retention controls | independently collected storage/network/logging/cleanup evidence | unapproved persistence, visibility gap where strict claim required |
| F. Owner authorization | access-to-process evidence pre-ingest; owner export approval at release | missing rights/approval or illegal policy request |

The release controller requires **all mandatory gates** to pass. An operator cannot silently override a compliance or critical security failure. Exceptions require an explicitly separate scope and legally/technically authorized review.

---

## 9. Reconciliation and evidence verification

Counts alone are insufficient. Under one verified source snapshot, identify each logical record using a source-stable position/key plus job-scoped protected commitment, including duplicate content at distinct legitimate locations. Every in-scope input has exactly one terminal disposition: `EXPORTED`, `SUPPRESSED`, `EXCLUDED_BY_OWNER_POLICY`, or `ERROR`; unresolved `ERROR` blocks export unless the approved contract expressly defines a narrower partial-export scope.

Partition processing emits source receipt, attempt/retry IDs, counts, protected commitment root, transform version, output content commitment, and infrastructure execution identity. Source evidence should, where feasible, be independently generated or verified by the connector/owner environment, not accepted solely from the transformation worker.

Sign separate evidence for **source scope**, **transformation execution**, **independent assurance**, and **final owner release**, with distinguishable signing identities. A customer-run verifier checks signatures, logical partition coverage, ledger totals, output hashes/object versions, policy versions, release state, and evidence availability. Signing attests to evidence provenance—not necessarily that its claims are true.

No unsalted hash of names, emails, IDs, low-entropy values, or raw sample text may enter general job metadata or telemetry.

---

## 10. Privacy and data-utility assurance

The privacy engine does **not** simply rerun the same Presidio detector. It uses independent rules/model components, targeted adversarial cases, protected owner-specific dictionaries, and (where appropriate) human-labeled evaluation samples. Report detector precision and recall **on named benchmark populations**, not as if actual per-export recall is known without ground truth.

Release-level findings include: residual pattern findings; sampled-review results if performed; uniqueness/quasi-identifiers; external linkage assumptions; risk from prior **owner-generated exports** when known; and uncertainty/limitations. K-anonymity and l-diversity are optional context-specific tests, not universal certification. `PSEUDONYMIZED` and `ASSESSED_DEIDENTIFIED` are distinct output classifications. A claim of legal anonymization requires a suitable contextual assessment and any necessary qualified review.

Utility is checked against **owner-supplied specifications**, measuring retained columns, categorical/numeric drift, suppressed rows, date interval utility, valid join relationships, and useful text remaining after redaction. More commercial value never compensates for unacceptable privacy risk.

---

## 11. Non-retention and secure hosting

**Control design**: use tenant/job segregation; narrowly scoped workload identities; encrypted transport and storage; secret manager and KMS; controlled source/export endpoints; deny arbitrary network egress; no external LLM/DLP API by default; read-only root filesystems where supported; restrict host mounts, swap, core dumps and diagnostic retention; allow-listed structured telemetry only; no raw values in stack traces or operator troubleshooting; short-lived credentials; and staged data lifecycle rules.

**Evidence**: infrastructure-side storage inventories and access logs, tested egress denies, container image identities, temporary-object/version inventory and cleanup checks, audited import bucket lifecycle, job-level worker permissions, and independently observed policy snapshots. Explicitly record cloud logging coverage and any evidence limitations. A worker's self-reported written paths are **not** sufficient proof of non-retention.

**Hosted-import truth:** uploaded raw objects may exist temporarily in our segregated intake bucket. The report identifies retention period, encryption, access policy, deletion status, and residual version/backup limitations. For customers requiring zero durable vendor raw-data staging, route through direct source streams or the optional owner-cloud deployment, subject to independent verification.

---

## 12. Owner export and lifecycle

**Only owner-facing destinations:** secure authenticated owner download, owner-controlled S3/GCS/Azure storage, or owner-authorized internal database/warehouse. Do not create native buyer delivery connectors. The owner chooses what happens after receiving the approved result.

Every approved job produces exactly two logical deliverables:

1. **Clean export package:** data files and machine-verifiable dataset manifest, with the output classification and explicit limits.
2. **Owner assurance package:** source snapshot reconciliation, transformation policy/digests, privacy/confidentiality findings, output utility vs owner spec, authorization records, infrastructure non-retention observations, object integrity, and worker cleanup evidence.

The platform retains only necessary owner/job/release-level metadata under a defined retention schedule. No buyer recipient registry, no buyer profiles, no cross-owner subject-token graph. An owner may use these job artifacts to manage any external legal and contractual obligations, but this platform does not track data sales, buyer possession, or buyer-requested deletion.

---

## 13. Security, privacy and commercialization responsibility

**Before source access**, require the enterprise to identify its authority to permit processing and confirm applicable restrictions. The service must still evaluate its own roles/obligations under applicable law. A contractual assertion is evidence, not a universal waiver of our responsibilities.

**Before release**, the owner approves the exact output and export destination. The service independently checks that the chosen transformation and release mode satisfy its policy and that no prohibited contents are identified under its controls. The owner handles commercial negotiations, buyer licensing and delivery independently. If our firm separately brokers a transaction outside this product, analyze regulatory responsibilities for that **actual business activity** rather than claiming the service structure automatically removes them.

The service must not represent pseudonymous data as anonymous or imply that any de-identified export is automatically legal to resell. Applicable requirements may include U.S. state privacy and data-broker laws, GDPR/UK GDPR, trade secrets, source licenses, confidentiality contracts and sector restrictions. This document is an architecture design, not legal advice or a compliance certification.

---

## 14. Interfaces and job state

```python
class OwnerOutputSpec:
    # desired schema, format, utility constraints, owner purpose, release mode
    ...

class SourceConnector:
    def plan_snapshot(self, selection): ...
    def enumerate_partitions(self, snapshot): ...
    def stream_partition(self, partition, max_bytes): ...
    def verify_snapshot(self, snapshot): ...

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

Job states:

`DRAFT -> AUTHORIZED_TO_ACCESS -> SNAPSHOTTED -> DISCOVERED -> POLICY_APPROVED -> PROCESSING -> STAGED -> ASSURANCE_REVIEW -> OWNER_APPROVED -> PUBLISHED_TO_OWNER -> CLOSED`

Failure states: `BLOCKED`, `QUARANTINED`, `FAILED`, `CANCELED`. Publication is available only from an approved, verified state, and a later missing cleanup verification may leave the job `CLOSED_WITH_EXCEPTION` rather than falsely marking its non-retention evidence complete.

Separate early authorization **to access/process** from later authorization **to release the exact output**.

---

## 15. Scale and engineering budgets

Targets refer to input size and text volume, not record count alone. Benchmark **structured**, **text-heavy**, and **mixed relational** datasets. Report rows/s, MB/s, tokens/s, peak memory, end-to-end elapsed time, per-job cost, NLP share of compute time, evidence overhead, error recovery, and throughput at 1M then 10M rows.

Use Python/PyArrow/Polars, deterministic local transformations, selective Presidio+custom recognizers, bounded-memory worker containers, and a simple asynchronous queue. Build the partition/idempotency contract from day one. Add independent validation from the outset. Defer Spark, Kafka, Ray, Kubernetes, and large connector catalogs until load testing demonstrates a need.

No enterprise SLA on 10M text-heavy records should be promised before representative benchmarking and cost measurement.

---

## 16. Recommended engineering phases

| Phase | Build | Exit criterion |
|---|---|---|
| **0. Owner-centric product and threat model** | Clarify hosted retention modes, exact owner workflow, compliance/rights responsibilities, security boundaries, source connector semantics, policy/evidence contracts | Approved threat model and owner-only product scope |
| **1. Reference owner job** | File import, owner output spec, CLI/headless API, Arrow partition processing, basic structured/text redaction, policy compiler, reconciliation and manifest | Synthetic end-to-end owner export and evidence; no buyer-facing surface |
| **2. Enterprise assurance** | Independent scanner, global risk/utility checks, non-retention infrastructure verifier, owner approvals, fail-closed publication | Negative tests demonstrably block unsafe exports |
| **3. Hosted production and scale** | Authenticated owner onboarding, S3/Postgres connectors, hosted import isolation, queue + parallel workers, KMS, lifecycle, 1M/10M benchmarks | Documented throughput and complete job evidence |
| **4. Owner-facing enterprise readiness** | Customer-cloud optional deployment, RBAC, operational tooling, more requested connectors, security review, pilot-ready reports | Enterprise data-owner pilot passes security review |
| **5. Demand-driven additions** | Additional connector formats, specific lawful anonymization methods and compliance policies | Prioritized by data-owner demand, not buyer integrations |

A managed human-assisted initial workflow is intentional. An owner-facing dashboard may be built later; **a buyer-facing dashboard is permanently out of scope** unless the business explicitly changes direction.

---

## 17. Required negative-path acceptance tests

- Source changes between discovery and transformation -> abort/re-snapshot.
- Connector omits a partition -> no complete-processing certification.
- Duplicate records offset dropped records while aggregate counts still match -> lineage/accounting fails.
- New schema field or nested property appears -> block until classified and approved.
- Primary scanner misses a planted PII or secret in evaluation -> independent testing exposes the limitation; no fabricated perfect-recall claim.
- Owner's requested format retains prohibited PII or confidential content -> block or require privacy-safe revised spec.
- Worker attempts to write source data outside allowed paths or send it to unapproved endpoint -> infrastructure denies and logs.
- Hosted upload staging remains after retention deadline -> cleanup evidence fails and escalates.
- Export crashes halfway -> no `PUBLISHED_TO_OWNER` state and no incomplete approved package.
- Operator attempts to publish without exact owner approval -> reject.
- Buyer-specific identity, login, delivery destination, transaction, or billing feature is introduced -> reject as out of scope.
- Evidence verifier sees signature mismatch, missing snapshot proof, incomplete ledger, or changed output object -> fail verification.

---

## 18. Implementation decisions locked and open

**Locked:** data owner is the only platform customer; managed hosted service first; owner-cloud optional; owner supplies export requirements; all final outputs go to owner; no buyer integrations or marketplace; Python/Arrow; headless engine + CLI/API; partition and reconciliation design from day one; independent assurance; signed owner-facing export evidence; key-scoped pseudonymization; no re-ID vault by default.

**Phase 0 decisions to validate with real owners:** precise supported dataset types and formats; source connection auth patterns; whether hosted raw upload is acceptable; minimum retention windows; first two native connectors; typical text/structured mix; whether project-wide linkability is needed; specific accepted output classification and disclosure-risk methodology; required operator versus owner approvals; billing model for our managed service (not buyer transaction processing).

---

## 19. Standards and reference materials

- NIST SP 800-188: *De-Identifying Government Datasets: Techniques and Governance* (September 2023), https://csrc.nist.gov/pubs/sp/800/188/final. Applies as technical guidance and risk-analysis reference, not a blanket certification.
- European Data Protection Board: *Anonymisation / pseudonymisation*, https://www.edpb.europa.eu/topics/ai-and-technology/anonymisation-pseudonymisation_en. Distinguishes pseudonymisation from anonymisation.
- EDPB Guidelines 02/2026 on Anonymisation: draft under consultation as of 21 October 2026, https://www.edpb.europa.eu/documents/guideline/guidelines-022026-on-anonymisation_en. Do not label it final guidance.
- California Civil Code §1798.140 definition of deidentified information and related obligations. Apply according to actual business role and circumstances, with counsel.

**This is a product architecture, not a guarantee of perfect anonymity, complete third-party rights clearance, or compliance certification.**
