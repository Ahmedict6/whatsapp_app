# Salla WhatsApp Engagement App — Full Spec (Clone-Ready)

> Production-ready blueprint for a Salla app that sends WhatsApp notifications (abandoned carts, new orders), runs WhatsApp campaigns, and adds an AI agent for smart replies and copy generation. Supports **two channels**: (A) WhatsApp Cloud API and (B) WhatsApp Web–style device linking (QR).

---

## 1) Product Overview

**Goal:** Recover revenue and increase engagement for Salla stores via automated, compliant WhatsApp messaging and AI-assisted operations.

**Core personas**
- **Merchant (Store Owner):** Installs app, configures templates, launches campaigns, reviews analytics.
- **Marketer/CS Agent:** Runs broadcasts, A/B tests, uses AI to draft copy and quick replies.
- **Customer:** Receives transactional and promotional messages; can chat with the store.

**Top Value**
- 12–25% average uplift on abandoned cart recovery (varies by vertical, compliance, templates).
- Lower CAC by retargeting existing audience with automated drip and win-back.
- Faster support with AI-suggested replies and intent detection.

---

## 2) Feature Matrix

### 2.1 Transactional Messaging
- **Abandoned Cart Recovery**
  - Trigger windows (e.g., 30min, 2h, 24h configurable).
  - Variables: `{customer_name}`, `{checkout_url}`, `{cart_total}`, `{coupon}`.
  - Multi-step drips with unique templates and opt-out handling.
- **Order Events**
  - New order confirmation, payment status updates.
  - Shipping updates (if Salla exposes shipment events / 3PL webhook).
  - COD reminders and delivery slot confirmations.
- **Customer Actions**
  - Welcome message on first order.
  - Post-purchase review request with deep link.
  - Refill/reminder for consumables (configurable intervals).

### 2.2 Campaigns / Broadcasts
- Segmentation (RFM, last order date, total spend, product tags, country).
- Scheduling (send-now, schedule, recurring).
- **A/B Testing** (template variants, send-time tests).
- **Rate limit & throughput controls** per channel/number.
- **Link tracking** (UTM, short links) and **conversion attribution**.

### 2.3 Two Send Channels
- **A) WhatsApp Cloud API (Official)**
  - Template management & approval states.
  - 24-hour customer service window logic.
  - Quality rating monitoring, phone number status, WABA metrics.
- **B) WhatsApp Web–style (QR session)**
  - Device link via QR.
  - Auto-reconnect with persistent sessions & backup.
  - Health checks & anti-stall watchdogs.
  - (Compliance note) For promos, enforce store-side consent and throttling.

### 2.4 Inbox & AI Agent
- Unified inbox (Cloud API + QR sessions).
- **AI Smart Reply Assistant**
  - Intent detection (pricing, availability, delivery time, return policy, product info).
  - Suggested replies with store tone & Arabic/English bilinguality.
  - Quick actions (send invoice link, share catalog link, check order status).
- **AI Campaign Copywriter**
  - Generate campaign copy (A/B variants), in Arabic/English.
  - Tone presets (formal, friendly, witty, premium, concise).
  - Emoji & length controls.
- **AI Knowledge Base**
  - Private embeddings of store FAQs, policies, catalog highlights.
  - Retrieval-Augmented Generation (RAG) for accurate answers.

### 2.5 Templates & Localization
- Visual template editor with variables.
- Arabic-first support; RTL UI; fallback to English.
- Store-specific tone presets saved at workspace level.

### 2.6 Analytics & Reporting
- Dashboard KPIs: delivered, read, replied, CTR, recovered revenue, opt-outs.
- Funnel breakdown per flow (e.g., cart recovery step 1 vs 2).
- Campaign analytics with variant comparisons.
- Export CSV and webhooks for BI tools.

### 2.7 Admin / Ops
- App licensing & billing (trial, monthly, yearly; VAT-aware).
- Role-based access (Owner, Marketer, Agent).
- Audit logs (who sent what, when).
- Security (IP whitelisting for Ops panel, 2FA).
- System health page (queues, sessions, WhatsApp status).

---

## 3) Architecture

```
+----------------+        +-----------------+       +------------------+
|   Salla Store  |<--OAuth|  Salla App API  |<----->|  PostgreSQL DB   |
| (Merchant)     |  & WBK |  (Backend)      |       |  + Timescale ext |
+----------------+        +---------+-------+       +------------------+
         |                           |                       |
         | Webhooks: orders,carts    |                       |
         v                           v                       v
  +---------------+           +--------------+         +-----------+
  |  Event Intake |--->Queue->|  Workers     |--API--> |  WA Cloud |
  |  (Webhook)    |    Redis  |  (BullMQ)    |         |  API      |
  +---------------+           +--------------+         +-----------+
                                        |                     ^
                                        | Puppeteer/          |
                                        v venom/open-wa       |
                                   +-----------+              |
                                   |  WA QR    |--------------+
                                   |  Sessions |
                                   +-----------+

                         +--------------------+
                         | Frontend (Next.js) |
                         | Inbox, Campaigns   |
                         +--------------------+

                         +--------------------+
                         |  AI Services       |
                         |  (RAG + LLM)       |
                         +--------------------+
```

**Key flows**
- **Salla OAuth** → store installs app → we get tokens + permissions.
- **Webhooks**: `order.created`, `cart.updated/abandoned`, `customer.created`.
- Events → **Redis queues** → **Workers** → select channel (Cloud API / QR) by routing rules → send.
- Replies inbound (Cloud API webhook or QR listener) → Inbox → AI suggests responses.

---

## 4) Recommended Tech Stack

### Backend
- **Node.js + TypeScript**
- **NestJS** (modular, DI, decorators, robust validation via Zod/DTOs)
- **PostgreSQL** (JSONB-friendly, strong relational features; TimescaleDB extension for analytics time-series)
- **Redis** + **BullMQ** (queues, retries, rate limits, scheduled jobs)
- **Prisma ORM** (type-safe data access) or TypeORM
- **Puppeteer** or **Playwright** (for QR/WA-Web if needed), with **open-wa** or **venom-bot**
- **gRPC/REST** internal services split (optional microservices)

### Frontend
- **Next.js (React 18)**
- **Tailwind CSS** (RTL ready), **shadcn/ui** for DX
- **TanStack Query** for data fetching
- **i18next** with RTL support

### AI
- **RAG Service**: Python FastAPI or Node service
- **Vector DB**: **PGVector** (in Postgres) or **Qdrant**
- **Embeddings & LLM**: OpenAI, Azure OpenAI, or Anthropic (abstract via LangChain/LlamaIndex where helpful)
- **Guardrails**: Content safety, PII redaction (Presidio), function-calling for tool use

### Observability & Ops
- **Docker** + **Compose** (dev), **Kubernetes** (prod)
- **GitHub Actions** CI/CD → container registry → deploy to k8s
- **Prometheus + Grafana** (metrics), **Loki** (logs), **OpenTelemetry** traces
- **Sentry** (errors) & **Datadog** (optional APM)
- **Vault** or SSM (secrets), **OPA** (policy, optional)

### Security
- OAuth 2.0 with Salla
- JWT (signed, short-lived) for frontend auth to backend
- mTLS between internal services (optional)
- Row-Level Security for multi-tenant data (tenant_id scoping)

---

## 5) Data Model (high-level)

- **tenants** (store_id, plan, locale, country, timezone, wa_channel_type)
- **users** (role, permissions)
- **wa_channels**
  - cloud: waba_id, phone_id, phone_number, quality_rating, status
  - qr: device_label, session_key, last_link_time, health_state
- **contacts** (customer_id, wa_number, consent_status, last_seen)
- **templates** (name, channel_type, language, status, body, variables[])
- **campaigns** (segments, schedule, variants[], status)
- **messages** (direction, template_id, content, status, error, timestamps, channel)
- **events** (type, payload, source=salla/webhook/manual)
- **analytics_daily** (delivered, read, clicked, replied, revenue_recovered)
- **ai_kb_docs** (doc_id, title, chunks[], embeddings)
- **ai_sessions** (chat_id, agent_state, tools_used, rating)

---

## 6) Integrations

### Salla
- **OAuth Install** → scopes for Orders, Carts, Customers, Webhooks.
- **Webhooks** we subscribe to:
  - `order.created`, `order.updated`, `cart.updated`, `customer.created`
- **Pull** endpoints (periodic): products (for catalog links), customers (for segments).
- **Rate-limit handling**: backoff & retry.

### WhatsApp Cloud API
- Phone number registration & template approval flow.
- Webhooks for delivery receipts and inbound messages.
- **24-hour window**: only session messages; outside requires approved templates.
- Profile setup (business logo, description).

### WhatsApp QR (Web Session)
- Headless browser/container with persistent session storage.
- Auto-reconnect and “stale session” alarms.
- Throttling per device; backpressure if queue surge.

---

## 7) AI Agent — Design & Docs

### 7.1 Capabilities
- **Intent classification** (return policy, shipping time, product availability, pricing, bulk order, warranty).
- **Response drafting** using store’s tone, bilingual.
- **Tool use (function-calling):**
  - Lookup order by phone/email.
  - Fetch shipping policy snippet.
  - Generate invoice or payment link (via Salla, if available).
  - Create coupon (scoped & expiring).
- **RAG** over **private KB** (FAQs, policies, top products, delivery SLAs).

### 7.2 Pipelines
1) **Inbound message → NLU → Router:**
   - Classifier model → pick policy or product lookup tool → draft response → human approve or auto-send (per rule).
2) **Campaign copy generation:**
   - Brief (goal, audience, product tags) → 3 variants (short/standard/long) → Arabic & English → A/B labels.
3) **Quality guardrails:**
   - Toxicity/offensive filter; PII redaction; brand-unsafe phrase filter.
   - Hard constraints for legal (refund periods, shipping times) from KB rather than hallucination.

### 7.3 Prompting Pattern (example)
- **System:** “You are a brand-safe retail support agent for {store_name} speaking {language}. Use the knowledge base only; if unknown, ask a clarifying question. Keep replies under 600 characters unless asked.”
- **Tools:** `get_order_status(order_id)`, `find_policy(section)`, `create_coupon(percent, expires_at)`
- **Memory:** recent 10 turns; customer profile snapshot.
- **Output schema:** `{intent, confidence, suggested_reply, tools_invoked[], next_step}`

### 7.4 Deployment Notes
- **LLM provider abstraction** (env switchable).
- **Cost controls**: cache embeddings, short context windows, smaller models for classification.
- **Privacy**: do not send PII to third-party LLMs unless contractually approved; consider self-hosted small models for intent (e.g., fast text-classifier).

---

## 8) Compliance & Consent

- Store-side **opt-in tracking** (source, timestamp).
- Easy **opt-out** (“STOP”/“إلغاء”) with enforcement across both channels.
- **Template governance** (promotional vs transactional).
- **Data retention** windows, deletion APIs.
- Merchant ToS & DPA; customer privacy notice snippet for stores.

---

## 9) Analytics & Attribution

- Track message → click → session → checkout attribution window (e.g., 7 days).
- UTM tags and short links (own shortener service to preserve metadata).
- Dashboard drill-down by campaign/flow/segment.
- Export & webhooks to external BI.

---

## 10) Testing Strategy

- **Unit**: template rendering, consent logic, windowing rules.
- **Integration**: Salla webhooks, WA Cloud send & receipt, QR session send.
- **E2E**: Abandoned cart flow on a test store.
- **Load tests**: k6 or Locust on queue and sender throughput.
- **Chaos**: Session drop, Cloud API outages, Redis failover.

---

## 11) Deployment Plan & Timeline (aggressive)

- **Week 1–2**: Salla OAuth + Webhooks; DB schema; basic admin UI; Redis queues.
- **Week 3–4**: WA Cloud API integration (send + webhooks); Templates; Abandoned cart v1.
- **Week 5**: QR session channel; device link UI; health monitors.
- **Week 6**: Campaigns (segmentation + scheduling); A/B; analytics v1.
- **Week 7**: Inbox + AI Agent (intent + RAG KB); smart replies.
- **Week 8**: Polishing, security audit, billing, docs; beta rollout.

*(Conservative plan: 12–14 weeks with hardening and scale tests.)*

---

## 12) Cost & Scaling Notes

- **WA Cloud API**: pay-per-template + conversation category pricing; plan for monthly spend modeling.
- **QR**: almost zero per-message cost but operational risk (session drops, rate limits, possible bans for aggressive promos).
- **Infra**: start with 3–5 modest instances (API/Workers), Redis HA, PostgreSQL managed (e.g., 2 vCPU / 8GB), S3-compatible storage for exports.
- **Sharding**: separate sender pools by region, category (txn vs promo).

---

## 13) Risks & Mitigations

- **Policy Compliance** → strict consent checks; throttle; content guardrails.
- **Session Stability (QR)** → multi-instance session managers, auto-relink UX.
- **Deliverability** → template content quality monitoring; safe send windows; reputation tracking.
- **Multi-tenant leakage** → strong tenant scoping; RLS; unit tests for authz.
- **AI Hallucination** → KB-first answers; tool-forcing; human-in-the-loop on sensitive replies.

---

## 14) Local Development

- `docker compose up` spins: postgres, redis, api, worker, frontend, rag-svc.
- Seed script creates a demo store, products, and test templates.
- Storybook for UI components.
- Playwright tests for inbox and campaigns.

---

## 15) API Sketches

### Webhooks (Salla → App)
- `POST /webhooks/salla/orders` → { order_id, status, customer, total }
- `POST /webhooks/salla/carts` → { cart_id, items[], total, abandoned_at? }

### Internal Send
- `POST /send` → { tenant_id, channel, template_id, to, variables }
- `POST /campaigns/:id/schedule` → { start_at, rate_limit, segments }

### AI
- `POST /ai/reply_suggest` → { chat_id, last_msg, lang } → { reply, intent, confidence }
- `POST /ai/ingest` → { docs[] }  // private KB
- `POST /ai/search` → { query } → { snippets[] }

---

## 16) Operational Dashboards

- **Sender Health:** queue depth, success/error rate, WA channel health.
- **Template Performance:** CTR, recovery rate by template.
- **AI Quality:** acceptance rate of suggestions, escalation rate.
- **Compliance:** opt-out trend, complaint rate.

---

## 17) Documentation (Internal)

- **Runbook:** relink QR session, rotate Cloud API tokens, handle WABA quality drops.
- **Playbooks:** peak sale events (Eid, White Friday), send caps, warm-up strategy.
- **On-call SOP:** error budgets, alerts, rollback procedures.

---

## 18) Resources (Getting Started)

> (General links; replace with your org accounts as needed.)

- **Salla Developers:** https://docs.salla.dev/
- **WhatsApp Business Cloud API:** https://developers.facebook.com/docs/whatsapp/cloud-api
- **open-wa (QR sessions):** https://openwa.dev/
- **venom-bot (QR sessions):** https://github.com/orkestral/venom
- **NestJS:** https://docs.nestjs.com/
- **BullMQ:** https://docs.bullmq.io/
- **Prisma:** https://www.prisma.io/docs
- **TimescaleDB:** https://docs.timescale.com/
- **PGVector:** https://github.com/pgvector/pgvector
- **LangChain:** https://python.langchain.com/ (Python) or https://js.langchain.com/ (JS)
- **LlamaIndex:** https://docs.llamaindex.ai/
- **OpenTelemetry:** https://opentelemetry.io/docs/

---

## 19) Licensing & Pricing (Example)

- Plans: **Starter**, **Growth**, **Pro**
- Tiers by monthly sends & features (A/B, AI seats, inbox seats).
- Overage pricing per 1k messages.
- Trial: 14 days or 1,500 sends (whichever first).

---

## 20) Roadmap (Post v1)

- Multi-number routing & load-balancing.
- Drip builder (visual flow canvas).
- WhatsApp catalog sync & product messages.
- LLM fine-tuning with store transcripts (opt-in).
- In-app NPS & auto-tagging of intents.
- Shopify/Zid connectors (multi-platform strategy).

---

## 21) Acceptance Criteria (Go-Live)

- 99%+ successful delivery for transactional messages over 7 days.
- Abandoned cart recovery flow end-to-end in prod for 5 pilot stores.
- Inbox median first response time < 1 minute with AI assist.
- Opt-out functioning across both channels with audit.
- Pen-test passed; secrets rotated; backup & restore tested.

---

**This README.md is your single source of truth to build a full-featured, scalable clone of the Salla WhatsApp notifications app with a modern AI layer.**

