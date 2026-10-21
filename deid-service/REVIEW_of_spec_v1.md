# Review of Consolidated Architecture Specification v1.0

Date: 2026-10-21. Reviewer: second agent (author of PLAN.md v1 to v3).

## Verdict

Adopt as the baseline. It supersedes PLAN.md, which is retained as history.
The specification resolves every finding from the two earlier review rounds and
adds material that neither plan had: deployment profiles with an explicit
CUSTOMER_STAGING exception, a NON_RETENTION_UNVERIFIED status, a job state
machine, typed evidence artifact contracts, a negative-path test plan, metadata
minimisation for counts and schema labels, and an ADR sign-off list.

## Open items to resolve in Phase 0 ADRs

1. Who is "the customer" (ADR-001). The business pulls data from source
   companies' systems. A customer-controlled data plane therefore means the
   source company runs our signed container in its own cloud. Many enterprises
   will decline to operate a broker's workload. If the brokerage instead receives
   raw data and processes it in its own infrastructure, VENDOR_HOSTED is the v1
   profile, not a future one, and every non-retention claim is about our
   infrastructure. The spec must say which is the launch model, or specify both.

2. Engine integrity in a customer-run data plane (ADR-001, ADR-007). The customer
   signs the evidence with its KMS. Nothing stops a customer from modifying the
   engine and producing a well-formed bundle. Recipients buying from the broker
   rely on the broker's word. Mitigation: vendor-signed image digest plus
   customer-signed run, with the verifier checking both; record the limitation;
   confidential computing closes it later.

3. Record commitment mechanism (ADR-007). "Protected multiset validation" is
   named but not designed. Specify: per-partition keyed row commitments,
   aggregated into a Merkle root or a keyed homomorphic multiset hash, with
   multiplicity preserved so legitimate duplicate source rows are not reported
   as loss, and with the commitment key held customer-side.

4. Human review surface (ADR-004, Phase 1). The default puts review of uncertain
   fields inside the customer environment, but no phase builds the tool that
   shows samples to an authorised reviewer without vendor exposure. Scope a
   minimal CLI review flow into Phase 1.

5. Global validation is a third pass. Dataset-level uniqueness and linkage
   analysis over the combined release requires reading all quarantined output.
   Section 15 should budget it alongside Pass 1 and Pass 2.

6. Snapshot lifetime for databases (ADR-003). A PostgreSQL repeatable-read
   snapshot cannot be held for hours against a busy production database.
   Section 5 names the problem; the ADR must decide the default: customer-
   authorised immutable extract, or bounded-time snapshot with abort.

7. Dual-use of HMAC output as join key and as commitment. Keep the
   pseudonymisation key, the commitment key, and the signing key distinct in
   ADR-005 and ADR-007, with separate rotation.

## Facts to verify before quoting to customers

- DROP processing start date and the 45-day cadence (section 12, reference 5).
- EDPB Guidelines 02/2026 consultation window (reference 4).
- NIST SP 800-38G rev 1 second public draft removing FF3 (reference 7).
