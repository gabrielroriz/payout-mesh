# Payout Orchestrator — Project Scope

| | |
| --- | --- |
| **Status** | Draft v0.2 (first sketch, open for review) |
| **Author** | Gabriel Roriz |
| **Updated** | 2026-10-17 |
| **Companion** | [Design](02-design.md) |

## 1. Context

More and more companies pay people for **results instead of hours**. Creators get paid per view, affiliates per conversion, couriers per delivery, customers per loyalty milestone, and AI agents per completed task. A new category of startups sells this as an API, or *"performance-based payroll"* as [Grade](https://www.usegrade.com/) (YC W26) calls it: a tenant sets payment rules, the platform tracks performance, works out what each person is owed, and pays them worldwide (local bank, PayPal, Wise, stablecoins, …).

Any product like this rests on two hard engineering problems:

1. **Turning performance into money.** Rules have to be evaluated over a stream of events, and the result must stay correct when events arrive duplicated, late or reversed.
2. **Moving the money.** Thousands of people have to be paid across many providers and countries. Nobody can be paid twice, and the system cannot stall when one provider fails.

Payout providers are the weak link. Each one covers only some countries, currencies and methods, each has its own failure modes, and switching between them is expensive. Payout Orchestrator is the layer that makes depending on them survivable. It also makes the first problem a first-class, auditable part of the system.

## 2. Problem statement

> A platform must work out what each payee earned from raw performance events. It must then pay them through third-party providers exactly once, with a full audit trail, and without depending on any single provider.

**Running example:** *"Someone visits 10 restaurants → they earn 5% cashback."*

This looks trivial, but it isn't:

| What happens | What the system must do |
| --- | --- |
| Visit #10 arrives twice (a client retry) | Count it once: ingestion is idempotent |
| Visit #4 is refunded after the cashback was paid | Post a negative adjustment (a clawback). Never edit history |
| Visit #10 arrives 2 days after the month closed | Apply a window grace period, or carry the visit into the next period |
| The rule changes to "8 visits → 4%" mid-month | Version the rules: record which payee is on which version, and from when |
| 5% of $123.45 is $6.1725 | Apply an explicit rounding policy. No floats anywhere near money |
| The provider times out while paying the cashback | Find out whether the payment happened first. A blind retry pays twice |

**Pain points** (hypotheses to validate):

- **Single-provider dependency.** When the provider is down, the whole payout batch stalls.
- **Coverage gaps.** No single provider covers every country × currency × method at the best cost.
- **Ambiguous outcomes.** Timeouts and late or missing webhooks leave payouts in an unknown state, and teams end up reconciling by hand.
- **Batch-scale limits.** Weak idempotency guarantees and tight rate limits get in the way once batches grow.
- **Bespoke rule code.** Performance logic is hardcoded per customer, which makes it hard to audit or change safely.

## 3. Goals and non-goals

### Goals

- **G1.** Take in performance events at scale and turn them into earnings through declarative, versioned rules.
- **G2.** Keep a double-entry ledger as the single source of truth for every cent.
- **G3.** Route payouts across several providers, with automatic failover that cannot create duplicate payments.
- **G4.** Give tenants a clean API and signed webhooks. Give operators full traceability and replay.

### Non-goals (v1)

- KYC/AML, tax forms (W-8/1099) and DAC7 reporting. These are integrated through an interface, not built.
- Custody of funds or money transmission. Licensed providers move the money, and the orchestrator only sends them instructions.
- Performance tracking itself (for example, scraping social posts). Events arrive through the API.
- FX pricing. Provider quotes are used as-is.
- End-user dashboards. The product is API-first, with at most a minimal ops view.

## 4. Constraints

- **Language:** Go.
- **Asynchronous communication:** event-driven, over Apache Kafka.
- **Cloud:** AWS. The full stack must also run locally with one command.

## 5. Actors

| Actor | Needs |
| --- | --- |
| Tenant developer | Integrates through the API: sends events, defines programs, creates payouts, consumes webhooks |
| Tenant finance/ops | Funds the wallet, approves batches, reads statements |
| Payee | Gets paid the right amount, on time, through their preferred method |
| Operator (us) | Monitors providers, resolves exceptions, replays failed messages |
| Payout provider | Executes transfers and reports their status (Wise, PayPal, Dots, …) |

## 6. Glossary

- **Tenant:** a platform that uses Payout Orchestrator (a brand, marketplace or app).
- **Payee:** the person or entity being paid. A payee has payout methods and a verification status.
- **Performance event:** an immutable fact (`visit.completed`, `post.metrics_snapshot`) carrying a client-supplied unique `event_id`.
- **Program:** a named incentive ("Foodie Cashback") with one or more immutable **rule versions**.
- **Earning:** money a payee is owed, produced by a rule. It may be negative (an adjustment).
- **Ledger:** the append-only, double-entry record of every money movement.
- **Payout:** an instruction to move money from a payee's balance to one of their payout methods.
- **Provider / rail:** the external service (Wise, PayPal, …) and the method it uses (ACH, SEPA, PIX, PayPal, USDT, …).
- **Route:** the ordered list of providers eligible for a given payout.

## 7. Functional requirements

Priority: **P0** = MVP · **P1** = v1 · **P2** = later.

### 7.1 Tenancy, payees and compliance

| ID | Requirement | P |
| --- | --- | --- |
| FR-01 | Every resource is scoped to a tenant. API keys are issued per tenant and per environment (`test` / `live`). | P0 |
| FR-02 | Create and update payees and their payout methods (bank account, PayPal, wallet). Sensitive details are stored encrypted or tokenized. | P0 |
| FR-03 | A payout is blocked unless the payee is `verified`. The status is set by an external KYC provider or manually. | P0 |

### 7.2 Programs and rules

| ID | Requirement | P |
| --- | --- | --- |
| FR-04 | Define programs with declarative rules: event filter, aggregation (`count`, `count_distinct`, `sum`, `max`), window (`calendar_month`, `rolling_Nd`, `lifetime`), threshold or tiers, reward (`fixed`, `percent_of_sum`, `per_unit`) and caps. | P0 |
| FR-05 | A rule version becomes immutable once active and applies from its `effective_from` date. Changes never apply retroactively. | P0 |
| FR-06 | Dry-run a rule version against historical events to preview its cost (a backtest). | P1 |

The cashback example written as a rule is in the [design doc](02-design.md#3-rules-engine).

### 7.3 Performance events

| ID | Requirement | P |
| --- | --- | --- |
| FR-07 | Ingest events over REST, one at a time or in batches of up to 1,000, each carrying a client `event_id`. Duplicates are accepted and ignored. | P0 |
| FR-08 | A reversal event references an original event and undoes its effect. If the original was already accrued, the reversal produces a negative earning. | P0 |
| FR-09 | Events that arrive after the window's grace period are applied as adjustments to the current period. | P1 |

### 7.4 Earnings and ledger

| ID | Requirement | P |
| --- | --- | --- |
| FR-10 | The rules engine emits earnings, positive or negative, with their lineage: the contributing events and the rule version. | P0 |
| FR-11 | Every money movement is a balanced double-entry journal entry. Entries are never updated or deleted: corrections are new entries. | P0 |
| FR-12 | Payee balances: `pending → available → in_flight → paid`. Each tenant has a prefunded `wallet`. | P0 |
| FR-13 | A payout requires enough funds in the tenant's wallet. If a clawback leaves a payee balance negative, it is netted against future earnings. | P0 |
| FR-14 | Query balances and per-period earnings statements. | P0 |

### 7.5 Payouts

| ID | Requirement | P |
| --- | --- | --- |
| FR-15 | Create payouts on demand (P0), or through scheduled cycles per program, such as weekly with a minimum amount (P1). | P0/P1 |
| FR-16 | Every mutating API call takes an `Idempotency-Key`. A replayed call returns the original response. | P0 |
| FR-17 | Payouts follow an explicit [state machine](02-design.md#4-payout-state-machine). Every transition is stored with its actor and reason. | P0 |
| FR-18 | Batches are approved before they run, optionally by two people. | P1 |
| FR-19 | A payout can be canceled until it is submitted to a provider. | P1 |

### 7.6 Routing and providers

| ID | Requirement | P |
| --- | --- | --- |
| FR-20 | All providers implement one adapter interface. The MVP ships a fault-injecting **simulator** and one real sandbox adapter, with a second in P1 (candidates: Wise, PayPal). | P0/P1 |
| FR-21 | Per-tenant routing policies match on country, currency and method, and resolve to an ordered provider list with a cost or speed preference. | P0 |
| FR-22 | Fail over automatically on a definitive rejection or an open circuit breaker. **Never** fail over on an ambiguous outcome until it is resolved. | P0 |
| FR-23 | Respect each provider's rate limits, and use provider batch APIs where they exist. | P1 |
| FR-24 | Track provider status through webhooks, with a polling fallback for payouts stuck past an SLA. | P0 |

### 7.7 Notifications and operations

| ID | Requirement | P |
| --- | --- | --- |
| FR-25 | Send tenants outbound webhooks for earning and payout events. Webhooks are HMAC-signed, retried with exponential backoff, and replayable. | P0 |
| FR-26 | Reconcile daily against provider reports. Mismatches become exceptions for ops to handle. | P1 |
| FR-27 | Ops tooling can inspect and replay failed messages, and manually resolve a payout with an audit trail. | P1 |

## 8. Non-functional requirements

| ID | Quality | Requirement |
| --- | --- | --- |
| NFR-01 | Correctness | Zero duplicate payouts, and zero lost or double-counted earnings. **Correctness beats availability:** when in doubt, stop and flag. |
| NFR-02 | Ledger integrity | Every journal entry sums to zero per currency. Every balance can be rebuilt from the entries. |
| NFR-03 | Durability | No acknowledged event, earning or ledger entry is ever lost. RPO = 0 for the ledger, and point-in-time recovery for all data. |
| NFR-04 | Availability | Ingest API 99.95%, payout API 99.9%. Ingest keeps accepting events while downstream processing is down, buffering at least 7 days. |
| NFR-05 | Latency | Ingest p99 < 100 ms. Event → earning visible p99 < 5 s. Payout create p99 < 300 ms. Rerouting starts < 60 s after a provider is marked unhealthy. |
| NFR-06 | Throughput | 5k events/s sustained and 20k/s burst. A 500k-payout batch drains at the providers' rate limits, not ours. |
| NFR-07 | Scalability | Every processing stage scales horizontally. No global locks. |
| NFR-08 | Isolation | Per-tenant quotas and fair scheduling. One tenant's large batch cannot delay another tenant's payouts. |
| NFR-09 | Ordering | Per-payee ordering is preserved from ingest to ledger. |
| NFR-10 | Auditability | Full lineage: payout → ledger entries → earnings → events → rule version. Ledger retention is configurable (default 7 years). |
| NFR-11 | Security | Bank details are encrypted at rest with managed keys, and only the component that calls providers can decrypt them. API keys are stored hashed. Webhooks are signed with replay protection. Service-to-service traffic is authenticated and encrypted. Admin actions are audited. |
| NFR-12 | Observability | One distributed trace runs from event ingest to payout `paid`, across every asynchronous hop. SLO dashboards, plus alerts on stuck payouts, ledger drift, failed-message backlog and per-provider success rate. |
| NFR-13 | Operability | Zero-downtime deploys. Scaling or restarting workers does not stall processing. Rule and routing changes need no redeploy. Every failed message can be replayed. |
| NFR-14 | Determinism | The same events and rule version always produce the same earnings. |
| NFR-15 | Code quality | Idiomatic Go: `context` everywhere, no global state, wrapped and classified errors, `golangci-lint` clean, ≥ 80% coverage on domain packages. |

## 9. Capacity estimates

These are assumed design targets, still to be validated.

| Dimension | Assumption | Derived |
| --- | --- | --- |
| Tenants / payees | 100 / 2M | |
| Performance events | 20M/day | ~230/s average, ~2.5k/s peak (10×), so design for 5k/s |
| Event size | ~1 KB | ~20 GB/day raw, ~140 GB for the 7-day buffer (before replication) |
| Earnings | ~2M/day | ~4M journal lines/day ≈ 1.5B/year |
| Payouts | weekly cycles | peak of ~500k in one window |
| Provider throughput | ~200 payouts/s combined | a 500k batch takes ≈ 42 min. **Providers are the bottleneck**, so routing and rate limiting matter more than raw speed |

## 10. Out of scope for this document

How the system is built (architecture, Kafka topics, state machine, tech stack, milestones) is in the [design doc](02-design.md).

## 11. Open questions

1. **Provider failure modes:** which hurt most in practice? Outages, coverage, status or webhook reliability, idempotency, fees, speed, KYC friction? The answer decides which NFRs come first.
2. **Consumer-scale payees:** in a cashback program, payees are consumers, which means millions of small payouts. Are minimum thresholds and batching enough?
3. **Cashback semantics:** does "10 restaurants" mean distinct restaurants (assumed) or 10 visits? 5% of which spend?
4. **Funding model:** do tenants prefund a wallet (assumed), or settle afterwards?
