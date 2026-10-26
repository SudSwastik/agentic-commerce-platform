# Complex agent use cases

The first two milestones build simple agents. These are the deeper targets for the same platform, introduced only after service contracts and failure handling exist. Each scenario has a measurable business outcome beyond producing a plausible explanation.

## 1. Paid-order recovery — the main use case

**Incident:** a two-item order is paid. A warehouse cannot supply one item. An alternate warehouse has stock but adds shipping cost. The carrier accepted a previous shipment request, but its response timed out.

1. The order investigator builds a timeline with event IDs and resource versions.
2. Inventory, payment, fulfillment and policy specialists gather independent evidence in parallel.
3. The payment specialist confirms the existing capture and deduplicates a repeated webhook. The fulfillment specialist queries the original shipment key before proposing another shipment.
4. The planner compares: reconcile an existing viable shipment, split fulfillment, request an approved substitution, or cancel/refund. An accepted shipment does not prove that the required stock allocation exists.
5. Code validates action dependencies, current allocations, totals, policy limits, existing shipment state and the plan's evidence. Conflicting facts trigger refresh or escalation.
6. A human approves the exact plan if extra shipping or substitutions require it. On resume, expired reservations and resource versions are checked again.
7. The executor uses scoped service tools and stable action keys. A replanner handles an observed failure within a finite budget.
8. The verifier queries authoritative service state and reports recovered, partially recovered or escalated.

**Pass:** no duplicate payment/shipment, no negative stock, no unsupported substitution, and an evidence-backed final outcome. The agent must escalate contradictory facts rather than force a successful result.

**Why multiple agents:** separate context and permissions for money, stock, policy and shipment evidence; parallel independent reads; explicit synthesis of conflicting observations. Compare against one well-designed agent before claiming improvement.

## 2. Ambiguous payment reconciliation

**Incident:** checkout timed out, the customer retried, and webhooks arrived late and out of order.

A reconciler groups attempts by stable provider/order references. Deterministic code identifies duplicate events and valid payment transitions. A specialist explains the evidence and proposes unresolved actions. The executor can query or request an approved refund; it cannot create a new capture merely because a previous request timed out.

**Pass:** every provider transaction maps to the correct local intent; one capture has one effect; refund totals remain bounded; missing evidence stays unresolved.

**Deep extension:** add an immutable simulated double-entry ledger for captures/refunds, balancing every entry and reconciling ledger totals with provider reports. The LLM can investigate discrepancies, but ledger posting and balancing remain deterministic.

## 3. Scarce inventory and multi-order allocation

**Incident:** ten orders compete for five units across two warehouses. Some orders have delivery commitments; stock holds are expiring.

Agents collect customer commitments, warehouse options, substitution rules and shipping costs. A constraint solver or deterministic validator checks feasibility and produces/validates allocation candidates. The planner explains alternatives and requests exceptions where allowed. The executor applies versioned reservations; conflicts trigger refresh and replan.

**Pass:** no overselling, valid reservation transitions, explicit unmet orders and measured cost/service tradeoffs. Report feasibility and policy compliance independently of the agent's narrative quality.

## 4. Cancellation races and return/refund coordination

**Incident:** a cancellation arrives while fulfillment is creating a shipment; later, only one of two items is returned.

The investigator determines whether fulfillment was cancelled, accepted or still unknown. Policy and payment specialists compute permitted partial-refund candidates from accepted order values and prior refunds. A human reviews exceptions. The returns and payment services coordinate the approved action through a saga and reconciliation record.

**Pass:** no cancellation of an already completed shipment without a supported compensating process; no refund beyond the remaining captured amount; retries do not duplicate refunds. An unavailable refund adapter leaves the case pending rather than “resolved.”

## 5. Operations incident supervisor

**Incident:** a payment provider outage causes a growing checkout queue while the agent provider is rate limited.

The supervisor reads redacted metrics and traces, classifies affected incidents and proposes bounded operational responses: reduce worker concurrency, pause an eligible recovery class or route cases to review. Deterministic rate limits and circuit breakers enforce limits; a human approves deployment/configuration changes.

**Pass:** bounded queue growth under a declared workload, no dropped durable intents, no retry storm, and a documented recovery after the injected outage. More concurrent agents must not bypass a shared provider quota.

## Shared agent contracts

- Every factual claim includes evidence IDs and the version/time of observation.
- Plans contain permitted typed actions, dependencies, preconditions, costs, compensations and budgets.
- Code validates constraints; specialists do not grant their own permissions.
- Approval binds to the plan hash, actor, expiry and action limits.
- Never hold a database lock while waiting for an LLM or a human. Revalidate and acquire short transaction locks when committing.
- Writes use service-side idempotency and audit records; graph checkpoints are not the business source of truth.
- Tool failure, unknown outcome and business rejection are distinct states.
- Evaluation measures final service state, invariant violations, unresolved cases, model/tool calls, latency and cost per resolved incident.
