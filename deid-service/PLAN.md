# De-identification Service: Build Plan

Purpose: a pipeline that pulls data from company sources (connectors or file imports),
de-identifies it, exports the result, and produces evidence that nothing was retained
and nothing was missed. Not a training platform. Not HIPAA-scoped, but built so a
HIPAA-grade tenant could be added later without redesign.

## 1. Guiding constraints

1. Raw data is never persisted by the service. It exists only in memory inside a job
   and is gone when the job ends. The only durable outputs are: de-identified data,
   a signed run manifest, and an audit log that contains no record values.
2. Every column and every free-text span is accounted for. A column that is not
   classified is a hard failure, not a pass-through. "Missed" is impossible by
   construction, and the manifest proves it.
3. De-identification is policy-driven, not code-driven. Policies are versioned YAML.
   A change in policy is a change in evidence, so the manifest records the policy hash.
4. Scale by partitioning, not by a bigger box. Each partition (file, table chunk,
   date range) is an independent unit of work with its own reconciliation counts.
5. Compliance target: GDPR/UK GDPR (Art. 4(5) pseudonymisation, Recital 26 anonymisation
   test), CCPA/CPRA as amended by the California Delete Act (data-broker registration,
   DROP deletion requests from Aug 2026), Colorado, Virginia, Texas and Oregon broker
   laws. The pipeline must be able to honour a deletion/opt-out list at ingest.

## 2. Architecture

```
 Sources                     Job runner (ephemeral worker)                      Sinks
 ─────────                   ─────────────────────────────────────────          ─────────
 Postgres/MySQL  ─┐          ┌──────────┐  ┌──────────┐  ┌─────────────┐        ┌─ S3/GCS (Parquet/CSV)
 Snowflake/BQ    ─┤ connect  │ Ingest   │─▶│ Classify │─▶│ Transform   │─▶ ... ─┤  Snowflake / warehouse
 S3/GCS/SFTP     ─┼────────▶ │ (Arrow   │  │ (schema +│  │ (policy     │        └─ SFTP / signed download
 CSV/Parquet/JSON─┘ batches  │ batches) │  │ content) │  │ engine)     │
                             └──────────┘  └──────────┘  └──────┬──────┘
                                                                ▼
                                            ┌────────────────────────────────┐
                                            │ Quality gates                   │
                                            │  • residual-PII scan on output  │
                                            │  • k-anonymity / risk score     │
                                            │  • schema + null-rate drift     │
                                            └──────────────┬─────────────────┘
                                                           ▼
                                            ┌────────────────────────────────┐
                                            │ Governance                      │
                                            │  • per-partition reconciliation │
                                            │  • signed run manifest          │
                                            │  • hash-chained audit log       │
                                            │  • no-retention attestation     │
                                            └────────────────────────────────┘
 Control plane (separate, stores NO record data): tenants, connector credentials
 (in a KMS/secret manager, never in the DB), policies, job queue, manifests, keys.
```

Data plane and control plane are separated so that the thing holding customer data
is stateless and disposable, and the thing that is stateful never touches records.

### Processing model
- Read in Apache Arrow record batches (50k–500k rows). Columnar, zero-copy, and the
  same code path runs on one laptop or a 200-worker cluster.
- Reference implementation in Python with PyArrow (single process, multi-core).
  Scale-out path: the same partition/transform contracts run inside Spark or Ray
  workers. Millions of records fit on one machine in Arrow; billions need the cluster.
- Partitions are idempotent. A retried partition overwrites its own output and its
  own reconciliation row, so failures never double-count.

## 3. Classification (how we know nothing is missed)

Three layers, all recorded in the manifest:
1. Schema catalogue: column name + type heuristics (email, phone, SSN, name, DOB,
   address, IP, device IDs, account numbers, geo coordinates).
2. Content sampling: regex + validators (Luhn, email syntax, E.164, IPv4/6) over a
   sample of each column; disagreement with layer 1 is flagged.
3. Free-text detection: NER over text columns (Presidio / spaCy locally; a cloud
   DLP API as an optional backend) producing spans tagged with entity types.

Output: a column classification table where every column has exactly one of
`direct_identifier | quasi_identifier | sensitive | free_text | non_identifying`,
plus a confidence. Unclassified or low-confidence columns block the run until a human
approves the policy mapping. Approval is recorded in the manifest.

## 4. Transform library (policy actions)

| Action              | Use for                          | Notes |
|---------------------|----------------------------------|-------|
| suppress            | SSN, exact address, raw names    | Column dropped or nulled |
| pseudonymize        | customer_id, email as join key   | HMAC-SHA256 with per-tenant, per-purpose key in KMS; rotation supported; optional re-identification vault for the client only |
| tokenize (FPE)      | account numbers that must keep format | FF3-1 format-preserving encryption |
| generalize          | ZIP→ZIP3, age→5-year bands, city→region | Hierarchies in policy |
| date_shift          | event timestamps                 | Consistent per-entity random shift, keeps intervals |
| bucket / round      | income, counts                   | Configurable bins |
| perturb             | numeric measurements             | Laplace noise, optional differential-privacy budget tracking |
| redact_text         | notes, support tickets           | Replace detected spans with `<EMAIL>`, `<PERSON>` etc. |
| hash_join_key       | cross-dataset linkage            | Salted, salt escrowed per contract |
| keep                | truly non-identifying            | Must be explicit |

Post-transform gates: k-anonymity on declared quasi-identifiers (default k≥5,
configurable per contract), l-diversity on sensitive columns, and a residual-PII
scan on the output using the same detectors as layer 2 and 3. Any hit fails the run.

## 5. Governance and "prove it" evidence

Each run emits a **run manifest** (JSON, signed with the tenant's key) containing:
- source descriptor (connector, query/path, snapshot time), source row count, and a
  rolling content hash computed while streaming (no copy retained)
- policy ID + version + hash; classification table; per-column action taken
- per-partition reconciliation: `rows_in = rows_out + rows_suppressed + rows_on_optout_list`
- quality gate results (k-anonymity, residual-PII hits = 0, schema drift)
- output descriptor, row count, content hash, sink location
- no-retention attestation: worker ID, start/end time, list of filesystem paths written
  (must equal the declared outputs only), temp-dir cleanup verification, memory-only flag
- operator approvals with identity and timestamp

Plus a **hash-chained audit log** (each entry includes the hash of the previous), so
tampering or gaps are detectable. Audit entries contain counts, hashes, and actions,
never values.

This gives a client's auditor three things: proof of coverage (every column
classified and acted on), proof of integrity (counts reconcile, hashes match), and
proof of non-retention (worker wrote only the declared outputs and was destroyed).

## 6. Compliance mapping (non-HIPAA)

- GDPR: pseudonymisation with keys held separately (Art. 4(5), Art. 32); DPIA template
  shipped with the service; records of processing generated from manifests (Art. 30).
- CCPA/CPRA + Delete Act: honour the DROP deletion list at ingest (rows matching a
  hashed deletion identifier are dropped and counted, never exported); data-broker
  registration is the client's obligation, but we provide the processing evidence.
- Contractual: per-client policy, per-client keys, no cross-client key reuse, output
  re-identification risk score recorded for each delivery.
- Security baseline: TLS everywhere, secrets in KMS/Vault, workers with no persistent
  disk, encrypted temp space, least-privilege connector roles (read-only), SOC 2 style
  logging. ISO/IEC 27559 (privacy-enhancing de-identification framework) used as the
  checklist.

## 7. Delivery phases

Phase 1 (this session, reference implementation):
- `deid/` Python package: policy loader, classifier (layers 1 and 2), transform
  engine on Arrow batches, file connectors (CSV/Parquet/JSON), k-anonymity and
  residual-PII gates, manifest + audit log, CLI.
- Synthetic test dataset generator and a test suite proving: reconciliation holds,
  unclassified columns fail, residual PII fails, pseudonyms are stable and keyed,
  manifests are signed and verifiable.
- ARCHITECTURE.md, policy examples, DPIA outline.

Phase 2:
- Database and warehouse connectors (Postgres, Snowflake, BigQuery), S3/GCS sinks,
  opt-out list ingestion, free-text NER backend, FPE tokenizer.

Phase 3:
- Control plane API (FastAPI), job queue, Spark/Ray execution backend, manifest
  verification tool for clients, Terraform for ephemeral worker infrastructure.

## 8. Open decisions for you

1. Language/runtime: Python + Arrow for the reference, with Spark for scale-out.
   Fine, or do you want JVM-first?
2. Pseudonymization reversibility: do clients ever need to re-identify (requires a
   vault we must protect), or is one-way only acceptable? One-way is far simpler to
   prove non-retention for.
3. Deletion/opt-out list handling: do you receive DROP-style hashed lists today?
4. Where does this live: a new repository? Working inside the current repo is only
   a scratch location.
