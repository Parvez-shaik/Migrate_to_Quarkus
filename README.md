# Legacy → Quarkus: Event-Driven Order Platform

Migration of a blocking Spring Boot order monolith into a Quarkus microservice with Kafka events,
DynamoDB persistence, a Kafka-triggered AWS Lambda, an Express gateway with a LangChain assistant, and a React UI.

```
React (5173) ──► Express BFF (3000) ──► Quarkus order-service (8080) ──► DynamoDB  (orders)
                    │ LangChain                     │  ▲
                    └─ order summary (PII redacted) ▼  │ consume ORDER_CREATED -> PROCESSING
                                           Kafka topic "order-events"
                                                    │
                                                    ▼
                                        AWS Lambda (order-audit) ──► DynamoDB (order_audit)
```

## What changed in the migration

| Legacy (`legacy-monolith/`) | Quarkus (`quarkus-order-service/`) |
|---|---|
| Spring Boot + JPA + H2, one transaction | Quarkus REST + DynamoDB (AWS SDK v2) behind a repository port |
| Blocking SMTP call (300 ms) inside the request | Request returns after save; work is handed off as a Kafka `ORDER_CREATED` event |
| Fulfilment hard-wired into `create()` | Separate consumer advances `CREATED → PROCESSING` (idempotent for redelivery) |
| Platform threads, blocking | `@RunOnVirtualThread` handlers on Java 21 |
| No state rules | `OrderStatus` state machine rejects illegal transitions (HTTP 409) |
| No health/metrics/docs | `/q/health`, `/q/metrics` (Prometheus), `/q/swagger-ui`, `/q/openapi` |

## Run it

Prereqs: Java 21, Maven, Node 22, Docker.

```bash
# 1. Infra + services
(cd quarkus-order-service && mvn package -DskipTests)
docker compose up --build

# 2. UI
(cd react-ui && npm install && npm run dev)     # http://localhost:5173
```

Quarkus dev mode instead of the container: `docker compose up redpanda dynamodb`, then
`cd quarkus-order-service && mvn quarkus:dev` (dev profile points at localhost DynamoDB/Kafka).

Try the API directly:

```bash
curl -X POST localhost:8080/orders -H 'content-type: application/json' \
  -d '{"customerEmail":"jane@example.com","lines":[{"sku":"SKU-1","quantity":2,"unitPrice":19.99}]}'
curl localhost:8080/orders          # status flips CREATED -> PROCESSING within a second
```

Legacy baseline for comparison: `cd legacy-monolith && mvn spring-boot:run` (port 8081, each POST takes ~300 ms).

## Tests

```bash
(cd quarkus-order-service && mvn test)   # JUnit 5 + Mockito: totals, state machine, event publishing
(cd lambda-order-audit && npm install && npm test)
(cd express-bff && npm install && npm test)   # proxy, 502 handling, PII redaction, LangChain chain
```

## Deploying the Lambda

`lambda-order-audit/template.yaml` is a SAM template (`sam build && sam deploy --guided`) that wires the
function to an MSK cluster and creates the `order_audit` table. Audit keys are `topic-partition-offset`,
so redelivered batches overwrite instead of duplicating.

## LangChain assistant

`POST /api/assistant/summary` builds a LangChain chain (`ChatPromptTemplate → model → StringOutputParser`).
Order data is trimmed and emails/phones are redacted before reaching the model. Set `OPENAI_API_KEY`
(and optionally `OPENAI_MODEL`) to use a real model; without it a deterministic offline model keeps the demo and tests working.

## Measuring your own results

Add your own numbers after running both services (e.g. `hey`, `wrk`, or k6 against `POST /orders` on :8081 vs :8080).
Don't quote figures on a resume that you haven't measured.
