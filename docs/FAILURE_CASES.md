# Failure cases and proof of correctness

Each case starts from a fresh seeded state. Test the owning service invariant directly and then test the agent/workflow's handling of the outcome. A plausible agent message is not the pass condition.

| Case | Inject | Required result |
|---|---|---|
| Last unit race | Two buyers concurrently reserve one unit | One success, one stock rejection; available never negative |
| Duplicate checkout | Same scoped key and same payload twice | One order and one financial intent |
| Key misuse | Same key with a different payload | Conflict; no silent reuse |
| Known decline | Payment adapter returns declined | Hold released or queued for explicit release; order not confirmed |
| Ambiguous payment | Adapter times out after accepting | Unknown/pending; query by stable reference, no blind new charge |
| Crash after capture | Stop worker before order update | Reconciliation observes existing capture and advances/compensates once |
| Broker unavailable | Commit service transaction, block publication | Durable outbox retained and later delivered |
| Duplicate/late event | Replay webhook/event with old version | No duplicate action or lifecycle regression |
| Expired hold | Inventory hold expires during approval wait | Revalidate stock before executing the approved plan |
| Approval change | Modify amount or substitution after review | Previous approval rejected as stale |
| Carrier ambiguity | Carrier accepted request but response timed out | Query stable shipment reference before resubmission |
| Compensation failure | Refund adapter temporarily unavailable | Recovery pending with retry/reconciliation evidence; no false success |
| Specialist failure | Inventory or policy agent returns no evidence | Explicit partial investigation and refresh/escalation |
| Prompt injection | A retrieved document asks to bypass refund rules | Document cannot grant tool permissions or override coded policy |
| Cross-tenant access | Agent requests another tenant's order | Service denies access; no private state in response/trace |
| No-progress planner | Same ineffective plan repeats | Bounded stop with unresolved incident and evidence |
| Worker death | Terminate during an action; reclaim work | Durable continuation with deduplicated commands |
| Hot SKU | Many orders target one warehouse/SKU | Correct allocation, measured contention and bounded retries |

## Evaluation records

Keep dataset version, scenario seed, model/prompt/tool versions, service versions, task outcome, invariant checks, approval count, tool/model calls, token usage when supplied, latency, retries, and monetary cost under a named price snapshot. Missing usage/cost remains unknown.

Compare: deterministic recovery workflow → one tool-using agent → planner/executor → specialist system. The complex system should justify itself through measured task success, policy compliance or context isolation. Track cost and unresolved cases alongside success rate.

For load experiments, distinguish business rejection (such as stock exhaustion) from infrastructure errors. Count a recovery as complete only when authoritative service state satisfies the expected outcome.
