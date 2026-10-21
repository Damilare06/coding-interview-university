# Enterprise Data De-identification Service
## Consolidated Architecture Specification v1.0

**Date:** 2026-10-21  
**Status:** Proposed baseline for engineering review; not a compliance certification  
**Product owner:** AI data brokerage business  
**Primary outcome:** Ingest authorized enterprise data through connectors or imports; produce privacy-assessed, policy-compliant exports and independently verifiable processing evidence.

---

## 1. Executive decision

Build a **portable, customer-controlled data plane** with a lightweight **vendor-managed control plane**. The data plane classifies, transforms, validates, and exports records. The control plane manages tenants, policy versions, job orchestration metadata, approvals, and commercial delivery references, but neither handles raw record values nor retains identifying subject-level mappings.

The engine uses **Python, Apache Arrow, and Polars** for bounded-memory tabular processing and selective local NLP for unstructured text. Processes are partitioned and resumable from day one. Spark is **not** required for v1. The initial interface is a CLI and headless API; a web dashboard is not a prerequisite.

The service produces **two product deliverables**:

1. An authorized, privacy-assessed dataset in a customer-approved destination.
2. A signed **assurance package** documenting scope, permissions, transformations, reconciled dispositions, privacy and utility evaluations, infrastructure observations, and release authorization.

The service does **not** train models, own a data marketplace, operate a broad analytics platform, or target HIPAA-specific workflows in v1.

### Non-negotiable distinction

- **Complete accounting**: For a defined, verified source snapshot, every in-scope input record has a disposition. This is an engineering invariant we can test.
- **Detection quality**: Sensitivity detection has measured false-negative risk. Neither classification of every column nor a zero-hit scan proves perfect identification of all sensitive information.
- **Non-retention assurance**: Design and enforcement prohibit persistent *raw* customer data in vendor-controlled infrastructure; independent evidence verifies defined controls and observed activity. Absolute proof that no copies existed anywhere is not asserted.
- **Legal authorization**: Data protection and de-identification do not independently establish permission to distribute or license an export.

---

## 2. Product scope

### Supported use cases

- One-time and repeatable imports from CSV, JSONL, Parquet, and approved object storage.
- Read-only imports from relational databases; later from selected warehouses and SaaS systems.
- Dataset discovery, nested-schema inventory, identification of personal information, secrets, business-confidential content, and permitted use.
- Explicit policy-based suppression, generalization, redaction, pseudonymization, bucketing, and approved date adjustments.
- Preservation of analytical utility where consistent with the chosen privacy-release policy.
- Customer-controlled delivery as CSV, JSONL, Parquet, or approved database/warehouse sink.
- Full lineage, quality reports, independent release gates, evidence export, and recipient/delivery tracking.

### Exclusions for v1

- AI model training, vector database, marketplace or trading system.
- Re-identification vault, reversible universal identity service, or cross-buyer persistent identity graph.
- Audio, images, video, PDFs with embedded media, and arbitrary binary document formats; reject unsupported formats rather than silently processing them.
- Generic data catalog, MDM platform, streaming CDC, arbitrary ETL jobs, general-purpose BI.
- An unconditional “anonymous” or “100% PII-free” certificate.
- Generic Laplace-noise operation marketed as differential privacy without a formal privacy accounting design.

### Output labels

1. **Redacted** — documented removal or masking of selected items, without a general anonymity claim.
2. **Pseudonymized** — protected identifiers with controlled linkability; commonly remains personal data under relevant laws.
3. **Assessed de-identified / anonymized** — released only after an applicable, documented legal/statistical assessment for the intended recipient and context. Avoid displaying “legally anonymous” by default.

---

## 3. Trust boundaries and deployment

```text
                         VENDOR-MANAGED CONTROL PLANE
                  tenant/job metadata | versioned policies
               workflow/approval | delivery registry | API
                  NO RAW ROWS OR SUBJECT-LEVEL MAPPINGS
                                   |
                       signed job specifications
                                   v
+------------------ CUSTOMER-CONTROLLED DATA PLANE --------------------+
|                                                                      |
| source connectors / imports --> source snapshot and object inventory |
|                |                                                     |
|                v                                                     |
|        discover + classify + policy preflight                        |
|                |                                                     |
|                v                                                     |
|       parallel Arrow partition transformers                          |
|                |                                                     |
|                v                                                     |
|        quarantined customer-side output                              |
|                |                                                     |
|                v                                                     |
|    complete reconciliation + independent privacy/utility validation   |
|                |                                                     |
|                v                                                     |
|      rights and recipient authorization + atomic release             |
|           |                                     |                    |
|      export destination                    evidence package           |
|                                                                      |
| Customer KMS | customer logs | optional encrypted customer staging   |
+----------------------------------------------------------------------+
```

**Data plane**: Distributed as a signed, versioned OCI container image and open interfaces. Can execute in a customer account/VPC on AWS, Azure, GCP, or compatible on-prem infrastructure. Do not embed a hard dependency on SQS/Step Functions/Fargate in the processing engine. Platform adapters may use native queues, job runners, storage, and KMS.

**Control plane**: Job configurations, tenant references, contract IDs, connector configuration *references*, policy digests, schema metadata subject to confidentiality review, performance summaries, approvals, output/evidence references, and delivery records. Store secrets in a secret manager rather than database tables; prefer customer-side workload identities and KMS.

**Deployment profiles**:

- `STRICT_EPHEMERAL`: Source can be reread consistently; no raw intermediate data persisted. Workers use bounded memory, no swap/dumps, controlled SDK caches, no content telemetry. Fail on unsafe spilling or snapshot instability.
- `CUSTOMER_STAGING`: Where a stable reread is impossible, customer may explicitly authorize encrypted, time-limited **raw** staging in its own storage account. Manifest records this exception and cleanup verification. Never describe this mode as “raw data was never persisted.”
- `VENDOR_HOSTED` (future, if demanded): Separate product security profile and contract; cannot reuse customer-controlled non-retention claims without evidence matching this different trust boundary.

**Important:** Short-lived transformed outputs may still contain regulated or confidential information. Place them in customer-controlled quarantine, not an automatically public export path.

---

## 4. Services and boundaries

| Service/module | Responsibilities | State it may retain |
|---|---|---|
| `job-api` | submit/approve/cancel/status; authenticate tenants | metadata only |
| `policy-registry` | validate/version/approve immutable policy schemas | rules and digests; no row values |
| `source-adapter` | immutable object inventory, database snapshot, paged API extraction | customer-side cursors/checkpoints |
| `classifier` | schema inventory, profiling, PII/confidentiality and rights tags | aggregate profiles; sensitive ones stay customer-side |
| `policy-planner` | resolve classification to action and release mode | compiled policy + digest |
| `transform-worker` | bounded Arrow batches, local text detection, transformation | memory and approved customer temporary storage only |
| `reconciler` | partition and record disposition, duplicate/gap checks | protected commitments and counts |
| `privacy-validator` | independent detectors and whole-dataset risk checks | customer-side risk statistics |
| `utility-validator` | distributions, schema, joins, content survival | non-identifying aggregates where safe |
| `rights-gate` | provenance, purpose, licensing, recipient constraints | contracts/authorization references |
| `release-controller` | publish outputs atomically or quarantine/fail | release records, immutable manifest |
| `infra-verifier` | inspect enforced workload/storage/network policy and logs | customer-side security evidence and summaries |
| `evidence-signer` | sign canonical manifest, generate verifier bundle | keys in customer KMS or approved signing infrastructure |
| `delivery-registry` | track contracts, releases and recipients | vendor-side release metadata only |

The **independent** validator must not simply call the same detector twice with the same rules and thresholds. It may share serialization code but needs distinct detection evidence and targeted tests.

---

## 5. Source snapshot and connector contract

No completeness promise is possible until “what data counts as input” is precisely defined.

### Required source manifest

- `source_id`, `connector_type`, `access_scope`, `query_or_object_selection_reference` (redacted/sanitized if sensitive).
- Source schema and nested-field inventory; format/version information.
- Reproducible snapshot boundary (`object version + etag`, stable SQL snapshot or export ID, or supported SaaS cursor/watermark with repeatability semantics).
- Expected objects/partitions and, where independently obtainable, source counts.
- Extraction start/end and changes discovered during extraction.
- Source proof/commitments generated inside customer environment; no unsalted raw-value hashes in vendor metadata.

### Connector behavior

- **Files/object storage:** enumerate immutable object versions; checksum each object with an approved content integrity mechanism; support bounded streaming and deterministic chunk boundaries.
- **PostgreSQL/MySQL:** use suitable consistent read/snapshot semantics with deterministic keyset pagination; ensure snapshot lifetime is compatible with processing or use customer-authorized immutable extracts. Avoid unsafe `OFFSET` pagination over changing tables.
- **Warehouse:** use available snapshot/time-travel/export mechanisms and record version identifiers.
- **SaaS APIs:** require deterministic pagination and a stable snapshot/replay boundary. If unavailable, support explicit “best-effort extraction” only as a non-completeness-certified mode or require customer-side staging. Do not issue a full-completeness certificate.

The connector must expose methods conceptually equivalent to `plan_snapshot()`, `enumerate_partitions()`, `read_partition()`, `verify_snapshot()`, and `close()`.

The transformation phase should reject changes in source snapshot between discovery and actual processing.

---

## 6. Classification and policy semantics

Classification is **multidimensional**, not one of five mutually exclusive labels.

| Dimension | Examples |
|---|---|
| structure | scalar, nested struct, array, free text, binary/unsupported |
| personal-data role | direct identifier, quasi-identifier, special/sensitive attribute, none detected |
| business sensitivity | public, internal, confidential, trade secret, credential/secret |
| provenance/rights | owner assertion, licensed, externally restricted, unknown |
| permitted action | suppress, redact, generalize, tokenize, retain, quarantine |
| confidence and evidence | schema classifier, validator, content scan, manual approval |

**Classification pipeline**:
1. Recursively enumerate every field and unsupported format.
2. Use name/type signals, deterministic validators, controlled value profiling, and local NER for free text.
3. Sampling aids **discovery** only; production transform/validation must cover every in-scope value where the policy requires inspection.
4. Flag uncertain or newly introduced fields. An authorized reviewer may set explicit policy, but cannot silently waive mandatory release gates.
5. Map every field and applicable free-text category to explicit `keep`, `transform`, `suppress`, or `reject` actions.
6. Quarantine or reject unsupported nested types, binary blobs, or model-inaccessible content rather than assuming they are non-identifying.

**Policy format**: human-readable YAML as source of truth, compiled to a strict typed, versioned schema. Reject unknown properties, undefined action precedence, unsafe combinations, and incompatible output claims. Include `policy_id`, `revision`, `purpose_id`, `recipient_scope`, `release_mode`, `detector_profile`, `transform_rules`, `privacy_thresholds`, `utility_thresholds`, `exceptions`, and `approvers`. Record a canonical digest of the compiled policy, not only the YAML file name.

**Transformation priority**: prohibit export > suppress/reject > redact/generalize > purpose-scoped tokenize > explicitly keep. Tokenization is not automatically sufficient to permit retention of sensitive fields.

---

## 7. Transformation engine and cryptography

**v1 actions**: field/row suppression with reason; literal and structured secret masking; robust local free-text redaction; categorical and numeric bucketing; geographic/time generalization; optional bounded date shifting; contract-scoped one-way keyed pseudonymization; explicit passthrough with approved policy.

**Not default in v1**: reversible vault, FF3/FF3-1, arbitrary format-preserving encryption, generic differential privacy, global identity graph.

### Pseudonymization design

- Per-tenant, per-purpose, per-approved-linkage-scope key material, controlled by the data owner. Never reuse a universal secret across recipients.
- Use HMAC-SHA-256 or a vetted keyed PRF over an unambiguous canonicalized input and context (e.g., field namespace + tenant + purpose + key version + normalized identifier); avoid concatenation ambiguities.
- Derive output encoding and optional truncation only after a collision analysis suitable for dataset size; default to full-strength representation where possible.
- Persist key references and versions, **not** keys. KMS alone does not mean arbitrary raw key material never enters application memory; use provider-supported controlled key operations or securely managed envelope keys and audit their lifecycle.
- Scope stable tokens to one approved linkage context. Cross-contract linking disabled by default. Key rotation must specify whether historical linkage is intentionally broken or preserved through an owner-controlled bridge.
- Treat one-way tokens as potentially linkable personal information, not anonymous values.
- Do not build a re-identification vault for v1.

For transformations that require consistent state (e.g., per-entity date shifts), derive bounded deterministic values from scoped keying material where appropriate; ensure uniqueness and statistical disclosure constraints are independently evaluated.

---

## 8. Two-pass data flow and state machine

### Pass 1 — Plan

- Confirm contractual authority and intended recipient/use.
- Obtain stable source manifest, inventory and schema.
- Classify fields, run bounded profiling, determine data-type-specific risks, compile policy, estimate NLP workload and runtime budget.
- Require approval for uncertain or high-risk policies. Freeze source snapshot and policy digest.

### Pass 2 — Process

- Stream deterministic partitions with memory-budgeted Arrow batches; apply transforms and local detectors.
- Emit transformed partitions **only into customer-controlled quarantine**.
- Record disposition per logical input record and partition checks. Protect sensitive commitments in customer account.
- Support checkpointing and retries without externally visible double publication.

### Global release

- Verify snapshot invariants and *complete* source-to-partition-to-export accounting.
- Run independent residual PII/confidentiality detection, linkage/disclosure-risk assessment over the **combined** release, and utility checks.
- Check purpose, contract, recipient, deletion restrictions, and delivery authorization.
- Sign final evidence. Atomically publish a manifest pointing to exact approved immutable output objects; revoke/quarantine and prevent access on failure.

### State machine

```text
CREATED -> PREFLIGHT -> SNAPSHOT_VERIFIED -> POLICY_APPROVED
  -> PROCESSING -> QUARANTINED -> RECONCILING -> PRIVACY_REVIEW
  -> RIGHTS_REVIEW -> RELEASE_APPROVED -> PUBLISHED -> RETENTION_CLOSED

Every state may enter FAILED or CANCELLED before PUBLISHED.
A FAILED/CANCELLED job cannot publish. A PUBLISHED release is immutable;
revocation or correction produces an explicit successor event and recipient notice.
```

**No partial exports**: object uploads may exist only in private quarantine until the signed release manifest is committed. Consumers must resolve files through the release manifest; “object exists” never means “approved.”

---

## 9. Reconciliation and lineage

For each declared source partition:

```text
rows_read = rows_exported + rows_policy_suppressed + rows_opted_out
          + rows_quarantined_or_rejected

unaccounted_records = 0    (required before release)
```

Do not count retry attempts as logical records. Partition IDs and attempt IDs are different. Store one authoritative committed disposition for each partition with idempotency tokens.

**Counts alone are insufficient.** Detect missing/duplicate records with stable source identities where available and customer-held keyed record commitments / protected multiset validation where appropriate. Check key collision behavior. Confirm output structure, joins, expected object versions, row ordering where required, and cross-partition integrity. For aggregates/splits, use an explicit input-to-output lineage model rather than assuming one source row must produce one output row.

Do not store unsalted raw email/phone hashes in the vendor control plane: low-entropy hashes can enable guessing. Some counts, source names, and schema labels are themselves commercially sensitive; metadata minimization and tenancy apply to them too.

**Truthful claim**: “All records in the specified, verified extraction scope were reconciled.” This is **not** a claim that every record in an organization’s systems was discovered.

---

## 10. Independent privacy, confidentiality and utility validation

### Privacy release gates

- All fields (including nested fields) have policy coverage; unsupported fields fail closed.
- Transformation rule coverage and prohibited identifier checks.
- An independent output detector with different rules/model components, plus labeled benchmark recall/precision and adversarial cases.
- Dataset-level, recipient-appropriate uniqueness, quasi-identifier, and linkage analysis. K-anonymity/l-diversity are optional tools, never unconditional proof; there is **no universal k=5 threshold**.
- Examine repeated deliveries and potential joining with previous releases or external reference datasets.
- Explicit risk decision and, when necessary, qualified expert review before asserting a jurisdiction-specific anonymization outcome.

### Business confidentiality gates

- Secret patterns: passwords, keys, credentials, access tokens; enterprise-specific dictionaries.
- Confidential terms and contractual restrictions from source owners.
- Unknown formats and high-risk content require review or suppression.

### Utility metrics

- Retained fields and proportions; suppression rates; semantic content survival for text.
- Category frequencies, numeric distributions, null/drift, timestamp-interval fidelity where permitted.
- Referential integrity and permitted join success across tables.
- Flag utility loss above **contract-defined** tolerances but never permit a higher utility score to override an unacceptable privacy risk.

Both privacy and utility reports include evaluation methods, known limitations, policy version, metrics, and the affected release ID. No blanket “PII-free” or “GDPR compliant” certificate is emitted by an automated scan.

---

## 11. Non-retention assurance and security controls

### Enforced controls

- Customer-owned workload identity with read access only to approved source scopes and write access only to approved quarantine/export/evidence locations; deny all other storage paths.
- Root filesystem read-only where supported; no raw-data logging; swap/core dumps disabled; explicit policy against implicit Arrow/Polars/SDK spill or caching in `STRICT_EPHEMERAL` mode.
- Block arbitrary internet egress, including diagnostics and third-party LLM/DLP APIs by default; allow narrowly defined control, source, KMS and destination endpoints.
- Store metadata in restricted sinks, redact exceptions and traces, apply short-lived credentials and approved key use.
- Separate signing keys, operator privileges, and deploy permissions; apply image signing, SBOM, vulnerability management, and supply-chain attestation.
- Log configuration/policy changes; prevent unapproved worker image versions.

### Independently collected evidence

- Pre-run and post-run infrastructure policy snapshots (workload identity, storage ACLs, network routes, effective egress permissions).
- Provider-side access/activity events from customer accounts with needed object-level logging **explicitly enabled**.
- Image digest, signed job/policy digest, worker identities, approved endpoints, denied access events.
- Source/quarantine/export object inventory; customer-side cleanup and lifecycle verification for authorized staging.
- Audit checks for telemetry, SDK/debug caches, incident dumps, backups/versioned temporary objects and unapproved egress.

No system can retroactively prove that every memory byte was never copied somewhere based only on a worker's self-reported file paths. Report both **enforced prohibitions** and **observed evidence**, plus known visibility limitations. If evidence coverage is incomplete, mark `NON_RETENTION_UNVERIFIED` rather than `PASS`.

---

## 12. Export authorization, recipients and deletion

Before export require:

- Owner/processor identity; provenance assertions and documentation references.
- Allowed transformation, recipient, purpose, geography, retention period, and sublicensing/redistribution rights.
- Assessment of applicable privacy requirements and contractual restrictions.
- Approval for cross-dataset linkage and any extraordinary risk exception.
- Approved destination credential and exact resource scope.

**Delivery registry** (vendor-side): contract reference, release ID, recipient, destination reference, risk/utility decision reference, delivery time, recipient acknowledgment, restrictions, and revocation/successor records. **Do not** centrally retain a hashed identity graph or subject-level tokens.

**Customer-side matching service** (optional/future): maintain protected mappings necessary to service legally required subsequent deletion/restriction requests, and identify affected deliveries. The design must address whether recipients are actually obliged and technically able to act on those requests. Truly anonymous data may not support subject-specific deletion.

**California DROP:** Add an integration only when the business role and statutory applicability are established. As of August 2026, covered brokers must process DROP requests at least every 45 days; implementing row suppression at ingest alone is insufficient. Deletion fulfillment, suppression against reintroduction, reporting and applicable service-provider direction require an end-to-end process.

---

## 13. Evidence artifact contracts (v1)

Produce a machine-verifiable JSON bundle with a stable schema and canonical signing procedure. It must not embed source rows, sensitive free-text samples, raw identifiers, secret keys or guessable raw-value hashes.

**`SourceManifest`**: source kind, sanitized selection, immutable snapshot/version reference, object inventory, partition plan, source count/evidence references, snapshot check results.

**`PolicyManifest`**: compiled typed policy version and digest, classification decisions, approvals, detector/model versions, enabled transformations, privacy-release purpose and scope.

**`PartitionManifest`**: `partition_id`, `attempt_id`, source snapshot ref, logical source count, output count, disposition counters, protected integrity commitments, build digest, timestamps, success/error status.

**`PrivacyUtilityReport`**: measured classifier performance, remaining findings (non-sensitive summaries), quasi-identifier/linkage assessments, chosen thresholds and rationale, utility metrics, residual limitations, required review outcome.

**`InfrastructureEvidence`**: policy snapshots, workload/image identity, restricted egress evidence, log coverage status, observed violations, staging cleanup status, assurance limitations.

**`ReleaseManifest`**: `release_id`, exact output object references/version IDs, checksums of published output (consider content sensitivity and access controls), output row counts, links to constituent manifests, permitted recipients, signed approval, timestamp, publication status.

**Verifier**: independently checks schema versions, signatures, parent-child manifest links, partition coverage/dispositions, output version/checksums, policy/build digests, and status of all gates. The verifier must distinguish unsupported evidence from failed checks. Manifest signatures protect provenance/integrity of evidence, not the truth of every claim within it.

Example envelope (illustrative, not a final schema):

```json
{
  "schema_version": "1.0",
  "release_id": "rel_example_001",
  "source_snapshot_ref": "src_manifest_abc",
  "compiled_policy_sha256": "...",
  "dataset_mode": "pseudonymized",
  "logical_input_records": 1000000,
  "exported_records": 999200,
  "suppressed_records": 800,
  "unaccounted_records": 0,
  "gates": {
    "snapshot": "PASS",
    "record_accounting": "PASS",
    "privacy_risk": "PASS_WITH_DOCUMENTED_SCOPE",
    "utility": "PASS",
    "rights": "PASS",
    "non_retention": "VERIFIED_WITH_LIMITATIONS"
  },
  "publication": "APPROVED",
  "signature_ref": "customer-key-signature"
}
```

**Zero counts are not proof that all PII was found.** Unverified infrastructure checks or residual-risk approval requirements are explicitly represented; release policy determines whether those statuses block export.

---

## 14. Runtime abstractions and interfaces

Avoid hard-coding one cloud into the processing engine:

```python
class SourceConnector:
    def plan_snapshot(self, selection) -> SourceManifest: ...
    def list_partitions(self, snapshot) -> list[PartitionRef]: ...
    def stream_partition(self, partition, max_bytes) -> ArrowBatchIterator: ...
    def verify_snapshot(self, snapshot) -> VerificationResult: ...

class TransformEngine:
    def apply(self, batches, compiled_policy, key_provider) -> TransformedBatchIterator: ...

class EvidenceSink:
    def write_partition_manifest(self, manifest) -> EvidenceRef: ...
    def write_global_reports(self, reports) -> EvidenceRef: ...

class ReleaseSink:
    def stage(self, partition_id, transformed_batches) -> StagedRef: ...
    def publish(self, approved_release_manifest) -> PublishedRelease: ...

class InfrastructureEvidenceProvider:
    def collect(self, job_scope) -> InfrastructureEvidence: ...
```

Use strongly typed models for `JobSpec`, `SourceManifest`, `CompiledPolicy`, `PartitionRef`, `PartitionManifest`, `GateDecision`, and `ReleaseManifest`. No separate handwritten “success” flag may bypass gates. The final publisher consumes a signed and validated gate decision.

Cloud adapters implement source/sink/queue/KMS/logging contracts; the Arrow transform engine remains independent. CLI and API invoke the **same job state machine**, not separate code paths.

---

## 15. Throughput, scaling and error handling

**Primary scaling variable** is bytes and text token volume, not row count alone. Benchmark three workloads: wide structured rows, text-heavy support/event records, and mixed relational multi-table data. Track MB/s, rows/s, tokens/s, memory peak, inference cost/1M records, scan recall on held-out labels, end-to-end run cost, and evidence overhead.

- Batch by **byte and memory limits**, not a universal 50k–500k-row size.
- Use deterministic partition IDs, idempotent writes to private staging, atomic committed attempts, and deduped reconciler updates.
- Use local rules/validators for structured identifiers; batch NLP inference for relevant text; do not call a large LLM for every row.
- Autoscale portable container workers within customer resource quotas and the connector's rate limits.
- Introduce Spark/another distributed analytics engine only if actual measured global operations or scale require it; Spark may introduce spill/shuffle storage, which must satisfy the applicable retention profile.
- On crash/cancel, seal failed partition manifests, restrict partial outputs, retry from verified source partition, and remove orphaned quarantine artifacts under an authorized lifecycle policy.

There is **no supported claim** that 10M arbitrary text-heavy rows will fit a given time/cost budget until measured on representative records.

---

## 16. Test plan and required behavior

| Test | Required outcome |
|---|---|
| 10M-row synthetic import with stable source snapshot | bounded memory; all logical inputs accounted for; correct published output refs |
| loss of 100 records offset by 100 duplicates | record-level/commitment checks fail, not just count comparison |
| inconsistent connector pagination or changed snapshot | completeness claim blocked; job restarts or fails |
| new nested field or unsupported binary column | policy preflight fails closed |
| planted unusual identifiers in free text | detection benchmarks expose false negatives; independent detector may catch; no absolute recall guarantee |
| deliberately leaked credential in output | independent security/confidentiality gate blocks release |
| policy changes mid-job | frozen compiled digest enforced; job aborted or new job created |
| worker unauthorized write or network attempt | infrastructure denies and logs; gate reviews violation |
| job crashes after uploading quarantine object | safe retry; no duplicate committed partition; no partial public export |
| customer-staged source left behind | cleanup inspection detects it; retention gate cannot pass |
| a release lacks rights approval | no publication |
| cross-contract stable-token request | denied absent special scope and review |
| publication revoked | immutable correction/revocation event and delivery notification |
| evidence file modified | independent signature verifier rejects it |
| per-partition anonymity passes, global anonymity fails | global gate blocks release |
| identity request after prior delivery | correct customer-side rights process or documented non-applicability |

**Quality thresholds**: Set detection recall, privacy risk, text utility, accuracy, throughput, cost and deletion evidence limits with representative pilot data; do not invent universal numbers. A universal `k>=5` release rule is not approved.

---

## 17. Phased engineering program

**Phase 0 — Architecture/security contracts (must precede code):** finalize threat model; data types; deployment profiles; consistent snapshot contracts; customer-vendor responsibility matrix; metadata and keys policy; manifest schema; gate semantics; source rights. Exit: approved architecture record and negative test cases.

**Phase 1 — Headless working engine:** repository, signed container, typed Python package, CLI/API, CSV/JSONL/Parquet streaming, Arrow partition contract, rules-first classification, versioned YAML compiler, suppress/redact/generalize/HMAC transforms, private output staging, disposition manifests, synthetic test generator. Exit: fully reconciled synthetic runs, fail-closed policy tests, no unsupported format passthrough.

**Phase 2 — Independent assurance:** independent output scanning, global privacy-risk and utility reports, infrastructure checks, canonical signed manifests, offline verifier, authorization gate, atomic publication. Exit: every listed negative-path test either blocks publication or records an explicitly allowed, approved exception.

**Phase 3 — Connectors and measured scale:** S3/GCS-style object import, PostgreSQL and one customer-demanded warehouse, snapshot/resume, cloud exports, portable container autoscaling, benchmark 1M then 10M mixed and text-heavy records. Exit: benchmark evidence and complete processing accounting; defined accuracy/runtime/cost limitations.

**Phase 4 — Enterprise pilot:** customer-controlled cloud deployments, least-privilege templates, security review, service documentation, recipient delivery registry, retention/deletion operational runbooks, customer-facing assurance review. Exit: independent security review and at least one validated end-to-end enterprise pilot against agreed acceptance criteria.

**Phase 5 — Demand-led expansion:** targeted SaaS connectors, customer-side rights/deletion matching, additional privacy policy packs, formal assurance standards and certifications where relevant, optional Spark, optional UI. Do not expand scope on speculation.

### Repo organization (suggested)

```text
enterprise-deid/
  docs/architecture.md   docs/threat-model.md   docs/assurance-contract.md
  schemas/job.schema.json   schemas/policy.schema.json   schemas/manifest.schema.json
  src/deid/{api,cli,connectors,classification,policy,transform,validate}
  src/deid/{reconcile,evidence,release,crypto,adapters}
  tests/{unit,property,fixtures,negative,integration,load,security}
  deploy/{local,aws,azure,gcp}
  examples/{policies,synthetic-data,manifests}
```

---

## 18. Prioritized architecture risks and unresolved product decisions

1. **First buyers and datasets**: customer support text versus structured CRM/transactions materially change the NLP and privacy budget. **Default**: start structured and short-text data; benchmark long free text separately.
2. **Deployment responsibility**: who owns operational logs, keys, network configuration, and cleanup. **Default**: customer cloud owns sensitive data-plane infrastructure; vendor provides signed binaries, config and evidence verifier.
3. **Classification confidence and human review**: who is authorized to approve uncertain fields without exposing raw text to vendor operators. **Default**: authorized customer reviewer inside customer environment.
4. **Snapshot support**: not all SaaS APIs allow consistent extracts. **Default**: reject full assurance when connector can't establish scope consistency.
5. **Privacy output mode**: pseudonymized records may be suitable for controlled recipients, not open redistribution. **Default**: controlled-recipient mode; public/anonymized mode requires additional release review.
6. **Recipient restrictions and rights**: brokerage source permissions and deletion rules are business prerequisites, not late documentation. **Default**: fail closed when rights are unknown.
7. **Business model and disclosure of evidence**: what portions of manifests should the recipient, customer, and external auditor see. **Default**: role-scoped evidence bundles; no sensitive samples to recipients.
8. **Key rotation/linkability**: stable tokens across deliveries versus minimization. **Default**: stable only inside explicitly approved contractual scope.

No additional product question is needed to begin Phase 0 under these defaults. Decisions can be revised via explicit architecture decision records.

---

## 19. Regulatory and technical references

These are **design references**, not evidence that the software is certified or that any particular export complies with law:

1. NIST SP 800-188 (2023), *De-Identifying Government Datasets: Techniques and Governance*: https://csrc.nist.gov/pubs/sp/800/188/final
2. ISO/IEC 27559:2022, privacy-enhancing de-identification framework: https://www.iso.org/standard/71677.html
3. GDPR, Article 4(5), Recital 26, Article 30 and data-protection obligations: https://eur-lex.europa.eu/eli/reg/2016/679/oj
4. EDPB, Guidelines 02/2026 on Anonymisation (consultation open July 8–October 30, 2026; **draft**, not final): https://www.edpb.europa.eu/public-consultations/guidelines-022026-on-anonymisation_en
5. California Privacy Protection Agency, DROP requirements for data brokers (processing begins August 1, 2026; at least every 45 days for covered brokers): https://privacy.ca.gov/drop-for-data-brokers/
6. California CCPA statutory definition of de-identified information and restrictions on re-identification/recipients: https://leginfo.legislature.ca.gov/faces/codes_displayText.xhtml?division=3.&part=4.&lawCode=CIV&title=1.81.5.
7. NIST SP 800-38G Revision 1, second public draft (2025), FPE design caveats; FF3 removed from that draft: https://csrc.nist.gov/pubs/sp/800/38/g/r1/2pd
8. Google Cloud Sensitive Data Protection risk analysis, methodological reference for statistical disclosure-risk evaluation: https://docs.cloud.google.com/sensitive-data-protection/docs/compute-risk-analysis

**Legal review requirement**: Determine roles (controller, processor, data broker or other regulated entity) by actual activity, contracts, jurisdiction and data source. Never use a policy checkbox as a substitute for counsel's review of applicable rights and statutory obligations.

---

## 20. Architecture sign-off criteria

Engineering implementation may start once the following baseline records are approved:

- `ADR-001`: Customer-controlled data plane / portable OCI execution.
- `ADR-002`: Strict ephemeral versus explicitly approved customer staging profiles.
- `ADR-003`: Consistent source snapshot and deterministic partition contract.
- `ADR-004`: Compiled policy semantics, multidimensional classification and frozen policy hash.
- `ADR-005`: Purpose-scoped HMAC and customer-controlled key lifecycle.
- `ADR-006`: Independent privacy/utility assessment and global fail-closed release.
- `ADR-007`: Reconciliation/integrity protocol and canonical signed evidence schemas.
- `ADR-008`: Broker rights, recipient authorization and delivery registry separation.

**Recommended first engineering milestone:** A complete synthetic-data job using a stable Parquet source, schema inventory, a compiled policy, partitioned Arrow transformations, independent validation, perfect *accounting*, signed manifests and an atomic approved release—without any unsupported promises about perfect detection or absolute non-retention.
