# MXP — Unified Product Specification

Scope: MXP as a single, unified rules-and-signals engine spanning credit cards and deposit/lending products, built as a **product any bank customer integrates against** — not a bespoke system coded against one specific bank's infrastructure. This document captures the product decisions made in scoping discussion, distinct from the earlier POC architecture and pitch materials, which addressed each product line separately and implicitly assumed a single-customer build.

**Product boundary, stated plainly:** MXP owns the rules engine, the data model, the APIs (both consumed and exposed), and the Bank Admin console. A customer bank owns Data Warehouse compliance with MXP's published input contract, the three fulfillment systems (Accrual, Offer Management, Spend/Voucher Engine), identity/SSO, and the customer-facing channel that consumes MXP's APIs. MXP is not built against any one bank's specific systems — it is built against contracts that any compliant bank environment can satisfy.

---

## 1. Data Ingestion

**Decision:** A single ingestion mechanism serves both product lines. MXP does not integrate with any specific customer's Data Warehouse, Core Product Processor, or Core Banking system directly. Instead, MXP publishes a **required input contract** — the fields, grain, and format MXP needs from a Data Warehouse — that any customer bank's environment must satisfy. MXP reads via scheduled batch pulls against this contract.

This resolves what was previously an open question between the two product lines' separate architectures: unification does not require two ingestion paths. One batch-pull mechanism, multiple upstream sources feeding one warehouse (on the customer side), satisfying one published contract, is sufficient.

All prior design principles from the original solution architecture continue to apply: the batch pull is transient (not persisted by MXP), evaluation outcomes are reproducible by re-running against the same warehouse data, and MXP holds no independent copy of account or transaction data.

**New requirement arising from the multi-tenant framing:** the input contract itself — field names, types, required grain — must be formally specified and versioned as a deliverable of the build, since it is what every customer integration is built against.

---

## 2. Rule Engine

**Decision:** One unified rule catalog spans both product lines, using the same four proven rule shapes (Aggregate & Compare, Match & Flag, Time Window, Frequency/Count). No product-specific rule engine or evaluation logic is required — a spend limit, category restriction, or timing rule behaves identically regardless of whether it is evaluated against a card transaction or a banking transaction.

### 2.1 Spend Controls (customer-configured)
- Daily Spend Limit
- Monthly Spend Limit
- Merchant Category Spend Limit
- Merchant Restrictions
- Transaction Type Restrictions
- Timing Restrictions

### 2.2 Bank-Defined Missions
- Average Balance Maintenance (non-standard: balance must be maintained above threshold for at least 20 days within a calendar month, not a simple monthly average)
- UPI Transaction Count (genuine transaction volume, distinct from UPI registration)
- Refer a Customer (reward fires only on confirmed approval of the referred customer, not on referral submission)
- Avail an Additional Product (reward fires only on confirmed approval of the new product, not on application)
- Recurring Deposit Payment Streak (consecutive on-time contributions)
- Recurring Deposit Timeline Completion (full-term milestone, date-based condition rather than a threshold-based one)
- FD Renewal / Rollover (proposed extension, not yet built)
- Dormancy Prevention (proposed extension, not yet built)
- Cross-Product Spend Diversity (proposed extension, not yet built)

Referral and Additional Product missions both require an external approval signal before reward fulfillment proceeds — see Section 4.4.

---

## 3. Cross-Product Missions and Reward Routing

**Decision:** Missions may span product lines (e.g. a savings-account behavior contributing to a reward reflected against the customer's credit card relationship). The underlying mechanism is the proven cross-apply pattern: a triggering account's customer_id is resolved via the KYC Repository, and the reward is routed to the customer's designated primary account.

**Critical clarification — reward sub-profiles are preserved, not merged.** The Rewards Accrual System already exists for credit cards. This product extends that same engine to also serve deposit and lending rewards, but the customer's reward ledger remains split into two sub-profiles under one customer identity: a card-rewards sub-profile and a deposits-rewards sub-profile. These are not merged into a single pool.

**Sub-profile assignment rule:** the sub-profile a reward is credited to is determined strictly by the reward's origin — the product type of the rule that triggered it — never by the product type of the account it happens to be displayed against. A reward triggered by a savings-account behavior is always credited to the deposits sub-profile, even when that reward is surfaced to the customer via their credit card relationship as the primary account.

This requires MXP's fulfillment calls to the Accrual System to carry an explicit origin-product tag, not just a target account identifier — a change from the original Phase 4 fulfillment client design, which only passed account, value, and reason.

---

## 4. Fulfillment Layer

**Decision:** Three distinct downstream system *roles*, not two — realized as customer-side systems that MXP integrates with via a published API contract, not systems MXP builds or owns. MXP does not fulfill rewards or offers itself — it decides, and calls out to whichever customer-side system fulfills each role. A customer bank's actual Accrual, Offer Management, and Spend/Voucher systems may be the same underlying platform, different vendors, or in-house systems — MXP is agnostic, provided each satisfies the relevant published API contract.

### 4.1 Rewards Accrual System
Credits points/cashback. MXP publishes the contract this system must expose (or calls, if MXP is the caller) to receive fulfillment requests. A customer bank's existing card-rewards engine, if already present, can serve both product lines under this product, with the sub-profile separation described in Section 3.

### 4.2 Offer Management System
Surfaces cross-sell offers and eligibility invitations (e.g. a pre-approved loan invitation, a premium card upgrade). MXP sends an eligibility signal via a published contract; the customer's Offer Management system owns presentation and any subsequent origination handoff.

### 4.3 Spend / Voucher Engine
A distinct system, separate from both of the above, responsible for issuing vouchers. Not to be conflated with Offer Management.

### 4.4 Routing logic
MXP infers which of the three systems a given reward routes to based on the reward's configured type (e.g. "points" routes to Accrual, "voucher" routes to the Spend/Voucher Engine, "cross-sell eligibility" routes to Offer Management) — the bank user configuring a rule selects a reward type, not a target system directly.

### 4.5 Approval-gated rewards
Referral and Additional Product missions (Section 2.2) do not fulfill automatically on condition-met. **Resolved:** no separate approval signal or integration is required. Approval status is inferred from the same Data Warehouse batch data MXP already reads — the referred customer's application record (or the additional product's account-opening record) carries an approval status field as part of ordinary banking data. The referral or product-adoption rule is a standard `Match & Flag` shape, matching a referral code or account reference against an approval status field already present in the batch. This uses the existing ingestion mechanism (Section 1) with no new integration point.

Concretely: Customer A refers Customer B via a code. Customer B applies. On MXP's next scheduled batch run, if the Data Warehouse extract shows Customer B's application as approved and linked to Customer A's referral code, the rule's condition is met and MXP proceeds to fulfillment for Customer A. No additional signal, webhook, or integration is needed beyond the standard batch contract.

---

## 5. Bank Admin Configuration Interface

**Decision:** A new, first-class interface for bank users to configure rules directly, replacing the current developer-facing tooling (Swagger, direct SQL) as the mechanism for rule management. This did not exist in any form prior to this scoping round.

### 5.1 Roles
Two distinct roles, strictly separated:
- **Config Author** — creates and edits rule configurations (thresholds, categories, operators, reward type and value, applicable customer segment(s))
- **Approver** — reviews and approves or rejects proposed changes

The same individual can never act as both Config Author and Approver on the same rule change (strict maker-checker separation).

**Authentication:** the Bank Admin console supports standard SSO protocols (SAML/OIDC); MXP does not build or own identity itself. Role assignment (Config Author vs. Approver) is provisioned within MXP's own authorization layer, keyed against the identity asserted by the customer's SSO — MXP does not manage credentials, only role-based access once identity is federated in.

### 5.2 Rule Lifecycle
Every state-changing action on a rule — creation, edit, activation, or deactivation — follows the same maker-checker workflow, with no action type exempted:

Draft -> Pending Approval -> Approved (rule goes live, or goes inactive, depending on the action type)

A rejected change returns to Draft for revision by the Config Author.

This reuses the same pending/decision state-machine shape already established in the reward_offers mechanism (Phase 6 of the POC), applied here to rule configuration rather than customer reward acceptance.

### 5.3 Core Actions
- Config Author: create or edit a rule; submit any state change for approval
- Approver: review pending changes of any type (new rule, edit, deactivation); approve or reject, with a reason recorded
- Either role: view all rules, filterable by product type, customer segment, status, and lifecycle state

### 5.4 Audit Trail
Every lifecycle transition is permanently logged: who performed the action, what changed, when, and — for approval decisions — the approver's identity and reason. This is a new, append-only log distinct from the signals table, which records rule evaluation outcomes rather than rule configuration changes.

---

## 6. Customer-Facing Channel

**Decision:** Unchanged in principle from the original architecture — Mobile Banking (or equivalent channel) composes a customer's view from multiple sources: live balance and transaction detail from Core Banking directly, and behavioral judgments, reward status, and mission progress from MXP via the proven query-descriptor pattern (Section 5.3 of the original solution architecture document).

Segment-based visual personalization (illustrated for Mass and Affluent segments during pitch development) is a presentation-layer concern for the channel, not a distinct MXP capability — MXP's output (which rules apply, what values, what rewards) already varies by segment; the channel's visual treatment of that output is a separate, channel-owned design decision.

---

## 7. Enterprise-Grade Build Approach

**Decision:** Given MXP is now scoped as a multi-tenant product — integrated against by potentially many bank customers, not built bespoke for one — the build bar is higher than the POC.

**Recommendation:**
- **The core engine should be rewritten, not extended, from the POC.** The POC's Java/Spring code (built across this project) proved the rule logic and mechanisms are sound — it should be treated as a validated specification, not a codebase to build production on directly. The POC was never built with multi-tenancy, a formal input contract, or production-grade error handling in mind.
- **A formally versioned, published API contract** is required in both directions: what MXP requires from a customer's Data Warehouse (Section 1), and what MXP exposes to a customer's channel and fulfillment systems (Sections 4 and 6). Multiple bank customers will integrate against the same contract — it must be documented and versioned as a first-class deliverable, not implied by code.
- **Multi-tenancy is a first-class concern from day one** — data isolation between different bank customers' rules, signals, and configuration must be architected in from the start, not retrofitted after a single-tenant build proves out.

---

## 8. Customer-Environment Responsibilities (Out of MXP's Build Scope)

For clarity, the following are explicitly the responsibility of each customer bank's own environment, not part of the MXP build:
- Data Warehouse compliance with MXP's published input contract
- The three fulfillment systems (Accrual, Offer Management, Spend/Voucher Engine) and their own infrastructure capacity
- SSO/identity provider
- The customer-facing channel (e.g. Mobile Banking) and its consumption of MXP's exposed APIs
- Non-functional targets (batch processing windows, uptime requirements, audit retention periods) — these depend on the specific customer's tech stack and transaction volumes, and are set per deployment rather than fixed in the product specification (see Section 9)

---

## 9. Explicitly Not Yet Resolved

The following remain open, flagged for the next round of scoping or for resolution during the production-readiness workstreams:

- Whether deactivation-only changes might later be reconsidered for a lighter-weight approval path, given the confirmed decision to keep them at full maker-checker parity for now
- The three proposed mission extensions (FD Renewal, Dormancy Prevention, Cross-Product Spend Diversity) are named but not designed to the same level of detail as the proven nine
- Segment definitions beyond Mass and Affluent (used illustratively in pitch material) have not been formally specified as configurable entities within the Bank Admin interface
- Non-functional requirements (batch processing windows, uptime, audit retention) are deferred by decision — dependent on each customer's tech stack and volumes, to be set per deployment rather than fixed in this specification
