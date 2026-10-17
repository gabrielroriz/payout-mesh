# Payout Orchestrator

[![Go](https://img.shields.io/badge/Go-00ADD8?style=flat&logo=go&logoColor=white)](https://go.dev/)
[![Apache Kafka](https://img.shields.io/badge/Apache_Kafka-231F20?style=flat&logo=apachekafka&logoColor=white)](https://kafka.apache.org/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?style=flat&logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![CEL](https://img.shields.io/badge/CEL-555555?style=flat)](https://cel.dev/)
[![OpenAPI](https://img.shields.io/badge/OpenAPI-6BA539?style=flat&logo=openapiinitiative&logoColor=white)](https://www.openapis.org/)
[![gRPC](https://img.shields.io/badge/gRPC-244C5A?style=flat)](https://grpc.io/)
[![Protocol Buffers](https://img.shields.io/badge/Protocol_Buffers-555555?style=flat)](https://protobuf.dev/)
[![Buf](https://img.shields.io/badge/Buf-555555?style=flat)](https://buf.build/)
[![OpenTelemetry](https://img.shields.io/badge/OpenTelemetry-425CC7?style=flat&logo=opentelemetry&logoColor=white)](https://opentelemetry.io/)
[![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?style=flat&logo=prometheus&logoColor=white)](https://prometheus.io/)
[![Grafana](https://img.shields.io/badge/Grafana-F46800?style=flat&logo=grafana&logoColor=white)](https://grafana.com/)
[![Testcontainers](https://img.shields.io/badge/Testcontainers-555555?style=flat)](https://testcontainers.com/)
[![k6](https://img.shields.io/badge/k6-7D64FF?style=flat&logo=k6&logoColor=white)](https://k6.io/)
[![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat&logo=docker&logoColor=white)](https://www.docker.com/)
[![Terraform](https://img.shields.io/badge/Terraform-844FBA?style=flat&logo=terraform&logoColor=white)](https://developer.hashicorp.com/terraform)
[![Amazon MSK](https://img.shields.io/badge/Amazon_MSK-8C4FFF?style=flat)](https://aws.amazon.com/msk/)
[![Amazon RDS](https://img.shields.io/badge/Amazon_RDS-4053D6?style=flat)](https://aws.amazon.com/rds/)
[![Amazon EKS](https://img.shields.io/badge/Amazon_EKS-FF9900?style=flat)](https://aws.amazon.com/eks/)

Pay people for results, exactly once, across many payout providers.

Payout Orchestrator is a backend for performance-based payouts. Tenants send raw performance events, such as a restaurant visit, a conversion or a delivery. Declarative rules turn those events into earnings, a double-entry ledger records every cent, and payouts are routed across several providers with failover that never pays anyone twice.

> **Status:** design phase, no code yet. The [scope](docs/01-scope.md) and [design](docs/02-design.md) are first drafts, open for review.

## Why

More and more companies pay for results instead of hours: creators per view, affiliates per conversion, couriers per delivery, customers per loyalty milestone. Any product like this rests on two hard problems:

1. **Turning performance into money.** Rules run over a stream of events, and the result must stay correct when events arrive duplicated, late or reversed.
2. **Moving the money.** Each payout provider covers only some countries, currencies and methods, and each fails in its own way. Nobody can be paid twice, and one provider's outage cannot stall everything else.

Take the running example, *"someone visits 10 restaurants → they earn 5% cashback"*. Visit #10 may arrive twice, visit #4 may be refunded after the cashback was paid, and the provider may time out while paying it. The amount has to come out right in every case.

## What it does

- **Event ingest:** a REST API, one event at a time or in batches, idempotent on a client-supplied `event_id`.
- **Rules engine:** declarative, versioned programs with filters, windows, thresholds, tiers and caps. Reversals and late events become adjustments, never edits to history.
- **Ledger:** an append-only, double-entry journal that is the single source of truth for balances.
- **Payouts:** an explicit state machine, an `Idempotency-Key` on every mutating call, and a gate that blocks payees who are not verified or tenants without funds.
- **Routing:** per-tenant policies by country, currency and method. Payouts fail over automatically on a definitive rejection, and an ambiguous outcome is resolved before anything is sent elsewhere.
- **Notifications and operations:** signed webhooks to tenants, reconciliation against provider reports, and replay of failed messages.

It does **not** hold funds or transmit money (licensed providers do that, on its instructions), run KYC or tax reporting (these plug in through an interface), or track performance itself (events arrive through the API).

## Guarantees

- **Zero duplicate payouts, and zero lost or double-counted earnings.** Correctness beats availability: when in doubt, the system stops and flags.
- **No single-provider dependency.** When a provider fails, payouts are rerouted instead of stalling the batch.
- **Full lineage:** payout → ledger entries → earnings → events → rule version.
- **Determinism:** the same events and rule version always produce the same earnings.

## Architecture

A set of Go services that talk over Kafka. Events flow from the `api` through the `rules-engine` into the `ledger`. Payouts flow from `payouts` through the `dispatcher`, which owns routing, provider adapters, rate limits and circuit breakers. Postgres holds the ledger and every balance, and Kafka carries replayable facts between services, published through a transactional outbox and consumed idempotently.

All services live in one Go module and can run as a single process in development. The [design doc](docs/02-design.md) has the full diagram, Kafka topics, payout state machine and key decisions.

## Tech stack

| Concern | Choice |
| --- | --- |
| Language | Go |
| Streaming | Apache Kafka (franz-go), Amazon MSK on AWS |
| Database | PostgreSQL (pgx, sqlc, goose) |
| Rules | CEL (cel-go) |
| APIs | REST + OpenAPI (public), ConnectRPC/gRPC (internal), Protobuf + buf |
| Observability | OpenTelemetry, Prometheus, Grafana |
| Infrastructure | Docker Compose (local), Terraform (AWS) |

## Roadmap

| Milestone | Exit criteria |
| --- | --- |
| **M0** Foundations | `make up` starts the stack, and a test event flows end to end in a single trace |
| **M1** Money core | Fund a wallet → pay a payee → receive a signed webhook, with the ledger balanced |
| **M2** Orchestration | Kill a provider in the middle of a 100k batch: 0 duplicates, 0 lost |
| **M3** Performance engine | 10 restaurant visits, with duplicates, out-of-order events and one refund → exactly 5% cashback paid |
| **M4** Production-grade | Latency and throughput targets met under load, a clean reconciliation report, Terraform for AWS |

## Getting started

There is nothing to run yet. From M0, the full stack (Kafka, Postgres, Grafana and every service) will start locally with one command, `make up`, on Docker Compose.

## Documentation

- [Project scope](docs/01-scope.md): problem, goals and non-goals, requirements and capacity estimates.
- [Design](docs/02-design.md): architecture, Kafka topics, rules engine, payout state machine, key decisions and milestones.

## License

To be defined.
