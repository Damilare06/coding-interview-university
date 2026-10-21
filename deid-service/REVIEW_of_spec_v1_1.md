# Review of Architecture Specification v1.1 (Owner-First)

Date: 2026-10-21. Reviewer: second agent.

## Verdict

Approve as the baseline product boundary and architecture. It correctly removes
every buyer-facing element, keeps the buyer as a processing input only, and adds
material that improves on v1.0: the HOSTED_STREAM / HOSTED_IMPORT / OWNER_CLOUD
profiles each paired with the claim they can truthfully support, separate
authorisation to access versus authorisation to release, four distinct signing
identities, the CLOSED_WITH_EXCEPTION state, gate F for owner authorisation, the
typed OutputSpec normalisation step, and a negative test that rejects buyer
features if they creep in.

Previous open items now resolved: launch profile (hosted), rights gate as owner
attestation, per-project token scoping, global validation as its own stage,
snapshot semantics per connector type, SaaS best-effort mode excluded from the
completeness claim.

## Items to settle in Phase 0

1. **Gate D (utility) should not be a hard block.** A utility shortfall against
   the owner's own spec is the owner's decision. Make D advisory: it produces a
   warning the owner must acknowledge in the release approval. A, B, C, E and F
   stay mandatory, and owner acknowledgment of D never relaxes C.

2. **Third-party assurance of the hosted platform is missing.** In the hosted
   profiles all four signing identities belong to the firm, so "independent
   assurance" is independent code, not an independent party. The owner's real
   trust anchor is external: SOC 2 Type II, penetration testing, and an auditor
   with access to the infra-verifier evidence. Section 11 should say so and
   Phase 3 or 4 should carry the audit-readiness exit criterion.

3. **Operator access to raw data in a managed, high-touch service.** Assisted
   setup and troubleshooting invite firm staff to look at owner data. Define the
   default as no operator access to raw values; uncertain-field review happens
   via an owner-side reviewer seeing samples, or via synthetic and redacted
   previews. Any break-glass access is owner-consented, time-boxed, and logged
   into the job's evidence. Scope the minimal review tooling into Phase 1.

4. **Multi-tenancy is underweighted.** A hosted platform processes many owners
   concurrently. Needs its own ADR: per-tenant keys, no worker shared across
   tenants within a run, tenant-scoped quarantine, evidence and queue, and
   tenant-isolation tests in section 17.

5. **Key custody is ambiguous.** Section 7 says owner-managed KMS references;
   section 11 describes the firm's KMS. Decide the default: per-tenant keys in
   the firm's KMS, with owner-held keys (BYOK) as the enterprise option. Keep
   pseudonymisation, commitment and signing keys distinct with separate rotation.

6. **Cumulative linkage across an owner's exports should be default-on.**
   Section 10 says prior owner exports are considered "when known". The platform
   holds the job metadata for every release of the same source, so it always
   knows. Per-project keys stop token joins but not quasi-identifier linkage
   across exports; assess it by default.

7. **Record commitment mechanism still unspecified.** Section 9 names a
   "protected commitment root" but not the construction. The ADR should pick:
   per-partition keyed row commitments aggregated into a Merkle root or a keyed
   multiset hash, with multiplicity preserved, commitment key separate from the
   pseudonymisation key.

8. **Processor obligations are not listed as artifacts.** As a processor under
   GDPR Art. 28 and a service provider under CCPA, the firm needs a data
   processing agreement template, a sub-processor list (cloud provider at least),
   a DPIA input package for owners, and a procedure for owner deletion requests
   against the evidence the firm retains. These are Phase 0 or Phase 4 deliverables.

9. **Owner spec documents are commercially sensitive.** A buyer-informed spec
   may carry buyer names and terms. Store under tenant encryption, owner-
   deletable, excluded from telemetry, with a retention period in the schedule.

10. **Self-description.** The header calls the business an "AI data brokerage
    firm" while section 13 disclaims brokerage inside the product. Regulatory role
    follows activity, not labels, but owner-facing material should describe the
    firm as a managed de-identification service provider to avoid confusing the
    role analysis for owners' counsel.

## Minor

- Queue payloads and job specs must carry references only, never row data; add
  to section 11.
- ERROR-disposition retries re-read the source partition rather than caching the
  failed rows; state this in section 9 so HOSTED_STREAM remains truthful.
- Section 2 step 7 gives exports an expiry and deletion schedule; clarify this is
  for the authenticated download path only, since owner-controlled destinations
  are the owner's to manage.

## Facts still to verify before quoting to owners

- EDPB Guidelines 02/2026 consultation status (reference 3).
- NIST SP 800-38G rev 1 second public draft and FF3 removal (section 7 relies on it).
