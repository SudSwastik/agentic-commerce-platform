# Agentic Commerce Platform: services and agents

The planned platform covers checkout, fulfillment, returns and recovery. No services or agents are currently implemented. Build inventory, payments and orders as Java Spring Boot services one at a time, verify checkout, then add a Python agent. The wider topology below describes later extensions with service-owned persistence.

## Microservice ownership

| Service | Owns | Commands and queries | Important invariant |
|---|---|---|---|
| Catalog | Product definitions, SKU metadata, price versions | Get product, read price version | Accepted order lines keep their price snapshot |
| Orders | Order intent, immutable accepted totals, lifecycle, recovery references | Create/get/cancel order; apply versioned transitions | A state transition must be valid for the current order version |
| Inventory | Stock by SKU/warehouse; holds, expiry and allocation | Check, reserve, release, consume stock | Available stock never becomes negative; a reservation is consumed/released at most once |
| Payments | Provider adapter, payment attempts, capture/refund state, reconciliation references | Authorize/capture/refund/query by stable key | Capture/refund are deduplicated; unknown outcome remains unknown until reconciled |
| Fulfillment | Shipment plans, shipment attempts and shipment status | Quote, create/cancel shipment, query carrier | A shipment consumes a valid allocation and has a deduplicated request key |
| Returns | Return eligibility, return lifecycle, refund requests | Propose/approve return; request refund | Refund amount cannot exceed captured amount less earlier refunds |
| Notifications | Notification intents, templates and delivery receipts | Enqueue, query delivery | Replayed events do not create duplicate delivery intents |
| Agent orchestration | Investigation runs, plans, evidence references, approvals, checkpoints and evaluation metadata | Investigate, propose, approve, execute, resume recovery | Every executed write is authorized, deduplicated and linked to an approved/current plan |

Identity and access control are cross-cutting contracts. Begin with a synthetic tenant identity; add an authenticated boundary before exposing service APIs. Agents receive scoped service clients, never unrestricted database credentials.

Business data lives in each owning service. A LangGraph checkpoint holds agent/workflow execution state, not authoritative stock, money or shipment status. Graph replay must query current service state and use stable command keys rather than reissuing unsafe writes.

## Complex agent system: investigate → plan → validate → approve → execute → verify

The supervisor receives an incident such as “Order O-1042 was paid but cannot ship.” Evidence specialists can run in parallel; writes obey dependencies and service preconditions.

| Agent/role | Reads | Produces | Write access |
|---|---|---|---|
| Order investigator | Order history, service events and versioned status | Incident timeline and missing/contradictory facts | None |
| Inventory allocator | Warehouses, stock, holds and substitution catalog | Feasible allocation candidates | Initially proposals only; execution uses inventory command tools |
| Payment reconciler | Payment attempts and provider-status query | Known/unknown financial outcome and reconciliation evidence | Initially none; refund/capture goes through controlled executor |
| Policy analyst | Versioned shipping/refund/substitution policies | Cited constraints and approval requirements | None |
| Fulfillment planner | Carrier quotes and shipping commitments | Shipment candidates and expected delivery | Proposal only |
| Recovery planner | Validated evidence and constraints | Typed action plan with dependencies, costs and expected outcomes | None |
| Plan validator | Plan and current service versions | Feasible/invalid/stale decision with reasons | None; deterministic code and optional solver |
| Executor | Approved plan with scoped permissions | Command receipts and observed outcomes | Narrow allowlisted service commands |
| Outcome verifier | Fresh order, inventory, payment and shipment state | Recovered, partially recovered, unresolved or escalated | None |

Some roles are deterministic functions rather than LLM agents. Start with a single agent and introduce a specialist only when context, permissions or measured quality justifies it. The final system can contain multiple agents while transaction boundaries stay in normal service code.

## Typed recovery plan

Each action includes `action_id`, `service`, `command`, validated arguments, required resource versions, dependency IDs, idempotency key, expected effect, compensation, and required approval. The plan also includes `incident_id`, evidence IDs, total proposed spend/refund, policy version, expiry, tool budget and replan budget.

The validator rejects unsupported commands, dependency cycles, missing evidence and plans that violate money, stock or policy constraints. A human decision binds to the exact plan hash and limits. Changed amounts, recipients, product substitutions or stale facts require revalidation and potentially fresh approval.

A proposed action is not a completed action. A command receipt may still mean pending, not successful. Query current authoritative state before reporting recovery.

## Transaction and event boundaries

Each Java service commits its own business state and command result in a local database transaction. Initial checkout uses HTTP calls and a durable step ledger; no database transaction spans a remote call. When reliable events are introduced, each service commits its state and outbox record atomically; a publisher delivers events and consumers deduplicate by event ID. Do not claim a global ACID transaction across service databases.

An event envelope includes event ID/type/version, aggregate ID/version, tenant ID, timestamp, correlation ID and causation ID. Consumers handle duplicate and late delivery and refuse invalid lifecycle changes. A broker acknowledgment is not proof that all downstream work completed.

Saga state tracks forward actions and compensations. Compensation is a business action that can itself fail: a refund is not the same operation as rolling back a database row. Preserve unresolved work for retry/reconciliation or human review.

## Advanced stress scenario

An order has two items. One warehouse loses stock after payment, the alternate warehouse can fulfill at a higher shipping cost, the carrier API times out after accepting a shipment, and a duplicate payment webhook arrives. The recovery system must reconcile ambiguous outcomes, compare split/substitution/refund options, obtain required approval, reserve valid stock and avoid duplicate charges or shipments. If facts remain contradictory, escalate with evidence instead of making a confident guess.

See the [scenario fixture](../scenarios/complex_order_recovery.json) and [failure cases](FAILURE_CASES.md).
