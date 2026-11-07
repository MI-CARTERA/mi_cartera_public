# Mi Cartera

**Mi Cartera** is a personal finance web platform built as a microservice ecosystem. It centralizes expenses, savings, investments, bank accounts and goals in a single application, and automates the most tedious part: turning a PDF bank statement into structured, classified transactions.

This repository is the project's public entry point: it explains how the system is built and where each piece lives.

🇦🇷 [Leer en español](./README.md)

---

## Why it exists

The project started as a replacement for a fragmented way of handling personal finances: first a monthly expense spreadsheet, then a separate investment spreadsheet, and finally the need to relate expenses, savings, investments, bank accounts and goals in one place.

That origin drives the two decisions behind the whole design:

1. **Related entities instead of loose sheets**: transactions, currency boxes, fixed expenses and goals are queried and cross-referenced against each other.
2. **Less manual entry**: transactions come in from the bank's PDF rather than by hand, and the system proposes the classification for the user to review.

## What it does today

- Manage the month's transactions per currency box
- Record savings and investments as transactions, not as separate accounts
- Keep one bank account with several operating currency boxes (peso savings box, dollar savings box)
- Manage recurring fixed expenses with month-by-month status (`pending`, `paid`, `unpaid`, `not_applicable`)
- Define financial goals and generate an action plan
- Query history, balances, monthly totals and per-category aggregates
- Upload PDF bank statements, review the proposed classification and confirm it
- Review reports on where each PDF's interpretation failed and which lessons were learned

The frontend organizes this into sections: fixed expenses, transactions, savings, investments, financial goals, historical queries, files and classification, interpretation reports, and accounts.

---

## Architecture

```text
          Frontend (React + Vite)
                   |
                   v
        API Gateway (Spring Cloud Gateway)
                   |
   +---------------+----------------+
   |               |                |
   v               v                v
ms_users    ms_bank_accounts    ms_files   ms_transactions
ms_goals    ms_fixed_expenses   ms_notifications_cartera
                ms_ai_classifier (Python + FastAPI)
                   |
      PostgreSQL / Redis / RabbitMQ
```

Architectural traits:

- **One service per business capability**, each with its own database and its own Postgres user
- **A single HTTP entry point**: the gateway validates the JWT, applies per-IP rate limiting, adds security headers and propagates identity downstream
- **No service registry**: services are discovered through Docker Compose DNS (`ms-users`, `ms-transactions`, `ms-ai-classifier`, ...)
- **Two Docker networks**: a public one for frontend and gateway, and an internal one for service-to-service traffic
- **HTTP for CRUD and queries, RabbitMQ for cross-service reactions**, with retries, DLQs and idempotency in every consumer
- **A shared, language-agnostic event contract**, so the Spring and Python services speak the same language

## Repository map

| Repository | Role | Stack |
| --- | --- | --- |
| [`mi_cartera_infra`](https://github.com/MI-CARTERA/mi_cartera_infra) | Local orchestration, event contract, Postgres init, observability, generators | Docker Compose |
| [`frontend`](https://github.com/MI-CARTERA/frontend) | Web SPA: dashboard, classification review, reports | React 18 + Vite |
| [`api_gateway`](https://github.com/MI-CARTERA/api_gateway) | Single HTTP entry point, auth, CSRF, rate limiting, headers | Spring Boot 4 + Spring Cloud Gateway |
| [`ms_users`](https://github.com/MI-CARTERA/ms_users) | Users, login, TOTP MFA, session cookie | Spring Boot 3.5 |
| [`ms_bank_accounts`](https://github.com/MI-CARTERA/ms_bank_accounts) | Bank account, per-currency boxes, balances | Spring Boot 3.5 |
| [`ms_files`](https://github.com/MI-CARTERA/ms_files) | PDF uploads, metadata and workflow state | Spring Boot 3.5 |
| [`ms_transactions`](https://github.com/MI-CARTERA/ms_transactions) | Transactions, monthly balances, per-category aggregates, duplicate detection | Spring Boot 3.5 |
| [`ms_fixed_expenses`](https://github.com/MI-CARTERA/ms_fixed_expenses) | Recurring fixed expenses with monthly status | Spring Boot 4 |
| [`ms_goals`](https://github.com/MI-CARTERA/ms_goals) | Financial goals and action plans | Spring Boot 3.5 |
| [`ms_notifications_cartera`](https://github.com/MI-CARTERA/ms_notifications_cartera) | Persisted notifications and SSE stream | Spring Boot 3.5 |
| [`ms_ai_classifier`](https://github.com/MI-CARTERA/ms_ai_classifier) | PDF extraction, classification via rules + memory + LLM | Python 3.12 + FastAPI |

---

## The flow that sets the project apart

Uploading a bank statement and ending up with classified transactions crosses six services and three RabbitMQ hops:

```text
User uploads the PDF, choosing account and currency box
  -> API Gateway
  -> ms_files stores the file and its metadata (bankAccountId, bankAccountBoxId, currency)
     and publishes FILE_UPLOADED
  -> ms_ai_classifier reads the PDF from the shared volume
       extracts rows (pdfplumber; OCR as a last resort)
       validates each row against the printed balance
       classifies movement type and category
       publishes AI_CLASSIFICATION_STARTED / COMPLETED / FAILED
  -> The user reviews and corrects the pending classifications, then confirms
  -> ms_ai_classifier publishes TRANSACTIONS_CONFIRMED
  -> ms_transactions creates the transactions and publishes TRANSACTIONS_RECORDED
  -> ms_bank_accounts updates the currency box balance
  -> ms_files closes the workflow state
  -> ms_notifications_cartera persists the notification and pushes it over SSE
```

Because the real impact is asynchronous, the frontend does not assume a fixed delay after confirming: it polls the imported transactions and the balance change, with a retry cap.

### What the AI does and does not do

- **Extraction is local**: dates, amounts, signs and balances are resolved in code with `pdfplumber`, preferring tables that carry their own header and reading each column from its own cell. OpenAI neither reads the PDF nor performs OCR.
- **Classification resolves in three layers, in order**: fixed rules, the user's correction memory, and only then the LLM with Structured Outputs for whatever is left.
- Every result records **which layer decided it** (`rule`, `user_correction`, `historical_classification`, `llm`, `fallback`) and what the original proposal was, even if the user corrects it later.

### The system records its own mistakes

Every interpreted PDF leaves an auditable trail, because the first read of a new statement format tends to repeat the same mistake:

- **Reading errors**: each row is checked against `previous balance + amount = balance`. Rows that do not add up are stored with a code (`inverted_sign`, `amount_is_balance`, `balance_mismatch`, `balance_chain_break`) and flagged in the review panel before confirmation.
- **Classification errors**: only a user-made change of movement type or category counts as an error.
- **Lessons**: corrections are grouped by normalized description and injected into the prompt so the AI generalizes to similar descriptions; identical ones are already resolved by the correction memory before reaching the LLM. When a correction repeats, the system suggests promoting it to a fixed rule; when a rule itself produces the error, it flags that rule for review.

All of this is available in the interpretation reports section and exposed as Prometheus metrics.

## Event contract

The source of truth lives in `contracts/events/mi_cartera_events.yaml`, inside the infrastructure repository.

| Event | Routing key | Producer | Consumers |
| --- | --- | --- | --- |
| `FILE_UPLOADED` | `files.file_uploaded` | `ms_files` | `ms_ai_classifier` |
| `AI_CLASSIFICATION_STARTED` | `ai.classification_started` | `ms_ai_classifier` | `ms_files` |
| `AI_CLASSIFICATION_COMPLETED` | `ai.classification_completed` | `ms_ai_classifier` | `ms_files` |
| `AI_CLASSIFICATION_FAILED` | `ai.classification_failed` | `ms_ai_classifier` | `ms_files`, `ms_notifications_cartera` |
| `TRANSACTIONS_CONFIRMED` | `ai.transactions_confirmed` | `ms_ai_classifier` | `ms_transactions` |
| `TRANSACTIONS_RECORDED` | `transactions.recorded` | `ms_transactions` | `ms_bank_accounts`, `ms_files`, `ms_notifications_cartera` |

Consumer guarantees: retry with backoff, a durable DLQ per queue, and idempotency backed by database constraints (`source_event_id` in the classifier and in notifications, `source_job_id` + `source_sequence` for transactions). `ms_bank_accounts` applies the balance as an absolute value rather than an incremental sum, so a redelivery cannot shift the balance.

---

## Domain decisions worth telling

The value of the project is not only in the stack: it is in the rules that keep the numbers honest.

**One bank account, several per-currency boxes.** Expenses, savings and investments are not separate accounts: they are transactions inside the matching currency box. The entire dashboard is scoped to the active box.

**An outflow is not an expense.** Moving pesos into the dollar box produces two independent transactions, one per box, imported from two different statements. The outgoing leg lowers the source balance but is not an expense: the real expense happens when the destination box takes the debit. Counting both legs would double the monthly total, so internal movements are accumulated separately and cannot mark a fixed expense as paid.

**The discriminator is the category, not the type.** The incoming leg of an internal transfer arrives as `deposit` + `transfer_in`. `deposit` alone is not enough: it can also be money genuinely coming in from outside, such as a cash deposit at a teller. Salary, by contrast, is classified as `income`.

**A positive amount never adds to expenses**, even when its type looks like an expense. The real case is a VAT refund, which arrives with the same type as the expense that caused it. A single helper decides whether a transaction is an outflow, and its mirror on the backend computes the aggregates: if the two drift apart, a summary card would flag rows its own total does not include.

**Several statements for the same month do not re-import what is already there.** A transaction's identity is date + description + amount. The running balance is not a condition, because the bank may deliver the same day's transactions in a different order across two PDFs; it is used only as a tie-breaker preference. The movement type does not take part either, since it is classifier output and the user may have corrected it. One single implementation feeds both the real import and the review panel, so the panel cannot announce one thing while the import does another.

---

## Security

The model was reviewed against the OWASP Top 10, and the decisions below are implemented, not merely documented:

- **Identity comes from the token, never from the header.** The gateway validates the JWT and propagates `X-User-Id`, but every service re-validates the signature and rewrites that header from the token's `sub`. Trusting the gateway alone would let anyone with access to the internal network read another user's portfolio.
- **Per-user scoping on every endpoint**, and another user's resource returns 404 rather than 403, so the response does not confirm that the id exists.
- **The session lives in an HttpOnly cookie**, not in `localStorage` nor in the response body, so an XSS cannot exfiltrate it. Since the browser sends the cookie on its own, there is double-submit CSRF protection, and the check runs only when the token came from the cookie.
- **Two layers against brute force**: per-IP rate limiting at the gateway backed by Redis, and per-identity failed-attempt counting in `ms_users`. An invalid MFA code counts too.
- **No credentials in versioned files**: each service connects to Postgres with its own user, secrets come from the environment, and a rotation script operates on the live environment without touching the data.
- **Minimal surface**: only the gateway and the frontend are published to the host; Postgres, Redis, RabbitMQ and the microservices are not. Every image runs as a non-root user with `no-new-privileges`.
- **Generic errors outward, full stack traces in the server log.**

Remaining gaps are tracked openly in the internal documentation (migrating the remaining services to Spring Boot 4, open registration, optional MFA, token revocation and versioned schema migrations).

## Observability

The local environment ships a full stack for following the flow end to end:

- **Prometheus** scrapes Actuator from the Spring services, metrics from the classifier, and from RabbitMQ, PostgreSQL, Redis and the containers
- **Loki + Grafana Alloy** for container logs
- **Tempo + OpenTelemetry Collector** for HTTP, JDBC and RabbitMQ traces, with W3C context propagated when publishing and consuming events as well
- **Grafana** as the single pane of glass
- An `X-Request-Id` is born in the frontend, travels as the event's `correlation_id` and survives all the way to transaction confirmation

No JWTs, PDFs, extracted text, prompts, amounts or personal data are exported to the observability stack.

## Tech stack

| Layer | Technology |
| --- | --- |
| Frontend | React 18, React Router, Vite, a centralized `fetch` layer, `EventSource` for SSE |
| Gateway | Java 21, Spring Boot 4, Spring Cloud Gateway, Redis for rate limiting |
| Domain services | Java 21, Spring Boot 3.5 / 4, Spring Data JPA |
| AI service | Python 3.12, FastAPI, SQLAlchemy, Pydantic, pdfplumber, PyMuPDF, Tesseract, OpenAI Structured Outputs |
| Data | PostgreSQL (one database and one user per service), Redis |
| Messaging | RabbitMQ with an event exchange, a DLX and a DLQ per queue |
| Orchestration | Docker Compose with public and internal networks |
| Observability | Prometheus, Grafana, Loki, Alloy, Tempo, OpenTelemetry |

## Local environment

The whole stack comes up from the infrastructure repository:

```bash
git clone git@github.com:MI-CARTERA/mi_cartera_infra.git
cd mi_cartera_infra
cp .env.example .env     # fill in secrets and OPENAI_API_KEY
docker compose up -d --build
```

Main entry points:

| Port | Service |
| --- | --- |
| `5173` | Frontend |
| `8080` | API Gateway |
| `3000` | Grafana |
| `15672` | RabbitMQ Management |

The microservices are not published to the host: they are reached through the gateway. Business data lives in the `pg_data` volume, and the PDFs in `files_data`, shared between `ms_files` (read/write) and `ms_ai_classifier` (read-only).

Public gateway routes: `/api/auth`, `/api/users`, `/api/bank-accounts`, `/api/files`, `/api/transactions`, `/api/fixed-expenses`, `/api/goals`, `/api/notifications` and `/api/ai` (rewritten to the classifier's `/api/v1`).

---

## Where it is going

Today the AI plays an operational role: extracting, normalizing and classifying transactions. The next step is moving from interpreting the past to assisting with the future:

- detecting spending patterns and analyzing financial habits
- suggesting realistic financial goals based on history
- proposing financial plans over time to reach them

The foundation is already there: the per-month, per-category aggregates exist precisely so historical questions can be answered without walking every transaction one by one.

---

A personal finance project of my own, built as an exercise in distributed architecture. Each repository in the organization is the source of truth for its own component.
