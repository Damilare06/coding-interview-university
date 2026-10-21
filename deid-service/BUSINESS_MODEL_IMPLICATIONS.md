# Business Model Implications for the Architecture Specification

Date: 2026-10-21. Amends ARCHITECTURE_SPEC_v1.md. To be folded into the Phase 0 ADRs.

## The model as stated

Enterprise customers come to the firm with de-identification needs. The firm
connects their company data to the firm's secure platform, de-identifies it, and
produces clean exports that are sold for cash to AI data buyers. The firm is a
two-sided operator: processor for the data owner, and seller to the buyer.

## Consequences for the specification

### 1. VENDOR_HOSTED is the launch profile (ADR-001, ADR-002)

The spec made the customer-controlled data plane the default and VENDOR_HOSTED a
future profile. Reverse that. The firm's platform is where raw data is processed,
so every non-retention promise is about the firm's own infrastructure, and the
firm must prove it with its own enforced controls and independently collected
evidence (spec section 11), plus a third-party audit path (SOC 2 Type II at
minimum). Customer-controlled processing becomes a premium enterprise option for
customers who refuse to send raw data; the portable engine design is kept so that
option costs little later.

Hosted import buckets, connector staging, queues, and quarantine storage are all
inside the firm's boundary. The STRICT_EPHEMERAL profile still applies to raw data
and to transformed-but-unreleased data. It does not apply to released exports,
which the firm retains in order to sell them. Say so plainly.

### 2. Two retention regimes, declared separately

| Data | Where it may persist | Claim |
|---|---|---|
| Raw customer data | Nowhere in the firm's infrastructure beyond ephemeral processing and lifecycle-bounded ingest staging | Non-retention, qualified as in spec section 11 |
| Quarantined transformed output | Firm quarantine storage until release or failure, then deleted | Protected as personal data until released |
| Released exports | Firm catalogue storage, encrypted, access-controlled, for the contract term | Retained for sale; retention period and deletion in contract |
| Evidence bundles | Firm control plane | Retained; contain no raw values |

### 3. The buyer is a first-class party (ADR-008)

The spec's delivery registry and rights gate were designed around one customer
and one destination. Now add:

- **Buyer-facing evidence bundle**: a role-scoped subset of the assurance package
  showing provenance, licensing chain, transformation summary, privacy assessment
  outcome, and limitations. No sensitive samples, no customer-identifying metadata
  the customer has not approved for disclosure.
- **Buyer contract gate**: before delivery, the buyer has accepted re-identification
  prohibition, no onward redistribution beyond licence, no linkage with other
  datasets beyond licence, and audit rights. These are required for the CPRA
  statutory definition of de-identified information, and they are what makes a
  sale defensible.
- **Per-buyer tokenisation at delivery**: the catalogued export carries tokens
  under a platform-internal key. At delivery, tokens are re-derived under a
  buyer-and-contract-scoped key so that two buyers cannot join their purchases.
  Catalogue storage never holds buyer-scoped tokens.
- **Revocation and deletion propagation**: the delivery registry records which
  release went to which buyer so that a revocation, a customer withdrawal, or a
  legally required deletion can be notified and, where contractually required,
  enforced.

### 4. AI training as the downstream purpose

Buyers are acquiring data to train or evaluate models. This raises obligations
beyond PII removal that the spec does not address:

- **Purpose and lawful basis**: the data owner's original collection purpose must
  be compatible with disclosure for AI training, or the data must meet the
  anonymisation standard, or consent must exist. Record which applies per release.
- **Intellectual property and third-party content**: free text may contain
  copyrighted material, third-party trade secrets, or content the customer does
  not own. Add an IP and third-party-content dimension to the rights gate.
- **Free text is the valuable part and the hardest part**: buyers want text; text
  is where detection recall is lowest. The detection effectiveness assurance and
  the business confidentiality gate are commercially load-bearing, not compliance
  decoration. Budget the NLP throughput work accordingly (spec section 15).
- **Model-inversion and memorisation risk**: pseudonymised records in training
  data can be memorised and regurgitated. The privacy release gate should prefer
  the assessed de-identified mode for AI-training buyers and treat pseudonymised
  output as controlled-recipient only.

### 5. Conflict of interest and separation of duties

The firm earns when exports sell. Customers want high utility; buyers want rich
data. Both pressures push toward weaker de-identification. Controls:

- Release approval authority is separated from sales. The release-controller
  decision cannot be overridden by a commercial role.
- "Utility never overrides an unacceptable privacy risk" (spec section 10) is
  enforced in code, not policy text: the gate decision object has no utility input.
- Risk thresholds are set per data-owner contract, not negotiated per buyer deal.
- Evidence bundles are immutable and available to the data owner for every sale
  of their data.

### 6. Regulatory position of the firm

The firm processes raw personal information on behalf of customers (processor or
service provider) and then sells derived data. Whether the firm is a data broker
under California, Texas, Oregon, or Vermont law depends on whether released
exports meet the statutory de-identified definition. If any released export is
pseudonymised rather than de-identified under the statute, the firm is very likely
a covered broker and the full DROP lifecycle (spec section 12) moves from
"future" to "before first pseudonymised sale". Counsel must decide this before
the first release mode is chosen. Default until then: release only in the
assessed de-identified mode.

### 7. Changes to the phase plan

- Phase 0: add ADR-009 buyer delivery and licensing, ADR-010 catalogue retention
  and per-buyer re-tokenisation, and the counsel determination on broker status.
- Phase 1: unchanged, with VENDOR_HOSTED infrastructure controls as the target.
- Phase 2: add the buyer-facing evidence bundle and the buyer contract gate.
- Phase 3: add catalogue storage with per-buyer delivery derivation.
- Phase 4: SOC 2 Type II readiness replaces "customer-controlled cloud deployment"
  as the enterprise exit criterion; customer-controlled becomes Phase 5.
