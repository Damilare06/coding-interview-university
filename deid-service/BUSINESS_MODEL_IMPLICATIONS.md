# Business Model Implications for the Architecture Specification

Date: 2026-10-21, revised same day after correction. Amends ARCHITECTURE_SPEC_v1.md.
Supersedes the earlier version of this file, which wrongly assumed the firm would
host a catalogue and deliver to buyers.

## The model as stated

Data owners come to the firm with de-identification needs. The firm connects
their company data to the firm's secure platform, de-identifies it to whatever
specification the owner chooses (often dictated by a buyer the owner has selected),
and returns the clean export to the owner. The owner sells it. The firm does not
host, deliver to, contract with, or build anything for buyers.

The firm is a processor acting for the data owner. Nothing more.

## Consequences for the specification

### 1. VENDOR_HOSTED is the launch profile (ADR-001, ADR-002)

The firm's platform processes raw data, so the non-retention promise is about the
firm's own infrastructure and is proved with the firm's enforced controls and
independently collected evidence (spec section 11), with a third-party audit path.
Customer-controlled processing stays as a later premium option; the portable
engine design is retained so it costs little then.

Ingest staging, queues, and quarantine are inside the firm's boundary and subject
to STRICT_EPHEMERAL or the explicitly authorised CUSTOMER_STAGING exception.

### 2. Retention: the firm keeps nothing but evidence

| Data | Where it may persist | Claim |
|---|---|---|
| Raw owner data | Nowhere beyond ephemeral processing and lifecycle-bounded ingest staging | Non-retention, qualified as in spec section 11 |
| Quarantined transformed output | Firm quarantine until released to the owner's destination or failed, then deleted | Protected as personal data until released |
| Released exports | Owner's destination only. The firm holds no catalogue and no copy | Not retained by the firm |
| Evidence bundles | Firm control plane | Retained; contain no raw values |

This is simpler and stronger than the earlier draft: once the export is published
to the owner's destination and quarantine is cleaned, the firm holds only metadata.

### 3. The buyer becomes a "target specification", not a party (ADR-004)

A buyer's requirements are captured as a reusable policy template the owner
selects or supplies: required fields, format, privacy standard to meet (for
example, statutorily de-identified under CPRA, or pseudonymised for a controlled
recipient), and any utility tolerances. The platform:

- compiles the target specification into the policy, with its own digest
- evaluates privacy gates against the standard the specification names
- evaluates utility against the fields and tolerances the specification names
- records the specification identifier in the release manifest

The platform does not validate buyer contracts, store buyer identities beyond an
owner-supplied label, deliver to buyers, or collect buyer acknowledgments. The
spec's delivery registry shrinks to: owner, release, target specification label,
owner destination, publication status, revocation or successor record.

### 4. Repeated exports of one source to different specifications

Owners will export the same source for several buyers. Cross-release linkage
(spec section 10) is therefore routine, not exceptional. Default: tokens are
keyed per owner and per target specification, so two exports cannot be joined
unless the owner explicitly requests a shared linkage scope. The cumulative
disclosure assessment considers all prior releases of the same source.

### 5. Rights gate becomes an owner attestation (ADR-008)

The firm cannot verify the owner's rights to sell, and does not try. The rights
gate records the owner's attestation of provenance, lawful basis, intended
purpose, and permission to export under the chosen specification, signed by an
authorised owner representative, and fails closed when absent. The owner's
downstream obligations (buyer contracts, re-identification prohibitions, deletion
propagation to buyers) are the owner's. The platform supplies the evidence bundle
the owner needs to meet them.

### 6. Conflict of interest

If the firm charges for processing, the incentive conflict is small. If the firm
takes a share of the owner's sale, the earlier controls apply: release approval
separated from commercial roles, utility has no input into the gate decision in
code, thresholds set per owner contract. Decide the pricing model in Phase 0 and
record it in the threat model either way.

### 7. Regulatory position of the firm

As a processor returning output to the owner and selling nothing, the firm is
very unlikely to be a data broker under California, Texas, Oregon, or Vermont law.
The owner may be. Counsel confirms. The DROP lifecycle remains out of scope for
the platform; the platform's contribution is the generic suppression-policy
interface so an owner can supply its own deletion list at ingest, and the
evidence showing it was applied.

The firm is a service provider or processor under CCPA and GDPR. Processing
agreements, sub-processor disclosure, and processor security obligations apply.

### 8. Changes to the phase plan

- Phase 0: ADR-001 and ADR-002 default to VENDOR_HOSTED. Add the target
  specification concept to ADR-004 and the owner attestation to ADR-008. Record
  the pricing model in the threat model. No buyer ADRs.
- Phase 2: the evidence bundle is for the owner; a role-scoped subset the owner
  may forward to a buyer is a formatting option, not a feature.
- Phase 3: per-owner, per-specification key scoping; cumulative release assessment.
- Phase 4: SOC 2 Type II readiness is the enterprise exit criterion.
- Removed from all phases: catalogue storage, per-buyer re-tokenisation at
  delivery, buyer contract gate, buyer acknowledgments, revocation notices to buyers.
