# 🔍 fintech-activity-logging

> **Fintech Activity Logging & SIEM Architecture** — End-to-end user activity capture, fraud risk scoring, and compliance archiving.
> React · Laravel · MySQL · AWS S3 · Kafka · FRM · SIEM

[![Version](https://img.shields.io/badge/version-2.0-00d4ff?style=flat-square)](/)
[![Stack](https://img.shields.io/badge/stack-React%20%7C%20Laravel%20%7C%20MySQL%20%7C%20S3%20%7C%20Kafka-7c3aed?style=flat-square)](/)
[![Compliance](https://img.shields.io/badge/retention-7%20years-10b981?style=flat-square)](/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow?style=flat-square)](LICENSE)

---

## 📑 Table of Contents

- [Overview](#overview)
- [Architecture Layers](#architecture-layers)
- [End-to-End Log Flow](#end-to-end-log-flow)
- [Project Structure](#project-structure)
- [Frontend — React Logging Layer](#frontend--react-logging-layer)
  - [IndexedDB Local Queue](#indexeddb-local-queue)
  - [Sync Worker](#sync-worker)
- [Backend — Laravel Ingestion API](#backend--laravel-ingestion-api)
- [Message Broker Layer](#message-broker-layer)
- [Storage Strategy](#storage-strategy)
  - [MySQL Hot Storage](#mysql-hot-storage)
  - [S3 Cold Archive](#s3-cold-archive)
- [FRM — Fraud Risk Management](#frm--fraud-risk-management)
- [SIEM & Alert Engine](#siem--alert-engine)
- [Log JSON Structure](#log-json-structure)
- [Compliance & Security Controls](#compliance--security-controls)
- [On-Prem Deployment](#on-prem-deployment)
- [Log Volume & Cost Analysis](#log-volume--cost-analysis)
- [Getting Started](#getting-started)
- [Environment Variables](#environment-variables)
- [Commit Message Convention](#commit-message-convention)

---

## Overview

`fintech-activity-logging` is a **production-grade, multi-layer activity logging and fraud detection system** for fintech applications. It captures every meaningful user interaction from the React frontend, queues it offline-safely in IndexedDB, batches and syncs to a Laravel API, and fans the data out to both a durable storage layer (MySQL + S3) and a real-time fraud risk engine (FRM + SIEM).

**Applications instrumented:** MMS · OPS · WegoX · IssureX · S3 Services · SMTP Services

---

## Architecture Layers

```
┌──────────────────────────────────────────────────┐
│                  CLIENT LAYER                    │
│   React UI  ←→  IndexedDB Offline Queue          │
│   LogBuilder · SyncWorker · Retry Engine         │
└──────────────────────┬───────────────────────────┘
                       ↓  HTTPS POST /batch
┌──────────────────────────────────────────────────┐
│           LOG INGESTION GATEWAY                  │
│   Laravel REST API                               │
│   Schema Validation · SHA256 Integrity Check     │
│   Rate Limiting · Timestamp · Trace ID           │
└──────────────────────┬───────────────────────────┘
                       ↓  200 OK (non-blocking)
┌──────────────────────────────────────────────────┐
│           MESSAGE BROKER LAYER                   │
│   Kafka / RabbitMQ  — Durable Event Stream       │
└────────────────┬─────────────────┬───────────────┘
                 ↓                 ↓
       Log Processor         FRM Consumer
       Worker                Risk Engine
                 ↓                 ↓
          MySQL (Hot)        SIEM / Alert Engine
          S3 (Cold)          Case Management
```

---

## End-to-End Log Flow

```
User Action
    │
    ▼
React Event Listener → LogBuilder (builds structured JSON log)
    │
    ▼
IndexedDB Local Queue  (offline-safe, survives tab close)
    │
    ▼
Sync Worker  (batch ≤ 100 logs, every 30s or on threshold)
    │
    ▼  POST /api/v1/activity/logs/batch
Laravel Ingestion API
    │
    ├──► Queue (Redis / DB Queue)
    │          │
    │          ▼
    │    Log Processor Worker
    │          │
    │    ┌─────┴──────┐
    │    ▼             ▼
    │  MySQL (Hot)   S3 (Cold Archive)
    │  30–90 days    7 years / WORM
    │
    └──► Kafka / MQ Fan-out
               │
               ▼
         FRM Microservice
               │
         ┌─────┴──────────────┐
         ▼                    ▼
   Risk Scoring Engine   Rule Evaluation Engine
         │
         ▼
   Threshold Check  (score ≥ 70 → ESCALATE)
         │
         ▼
   Alert Dispatcher  →  Email / SMS / Webhook
   Case Management   →  Fraud Investigation Ticket
         │
         ▼
   SIEM Dashboard
```

---

## Project Structure

```
fintech-activity-logging/
│
├── README.md
│
├── docs/
│   ├── architecture/
│   │   ├── Activity_Log_Architecture.html   ← Full architecture reference
│   │   ├── dfd-level-0.svg                  ← Context diagram
│   │   ├── dfd-level-1.svg                  ← Primary dataflow
│   │   └── dfd-level-2.svg                  ← Backend ingestion detail
│   ├── api/
│   │   └── openapi.yaml                     ← API specification
│   └── compliance/
│       └── data-retention-policy.md
│
├── frontend/                                ← React Application
│   ├── src/
│   │   ├── logging/
│   │   │   ├── LogBuilder.js                ← Constructs structured log objects
│   │   │   ├── IndexedDBQueue.js            ← Offline-safe local queue
│   │   │   └── SyncWorker.js                ← Batch sync + retry engine
│   │   └── ...
│   └── package.json
│
├── backend/                                 ← Laravel API
│   ├── app/
│   │   ├── Http/Controllers/
│   │   │   └── ActivityLogController.php    ← Batch ingest endpoint
│   │   ├── Jobs/
│   │   │   └── ProcessActivityLogBatch.php  ← Async queue worker
│   │   └── Services/
│   │       ├── RiskScoringService.php
│   │       └── AlertDispatcherService.php
│   ├── database/migrations/
│   │   └── create_activity_logs_table.php
│   └── ...
│
├── frm/                                     ← Fraud Risk Management Microservice
│   ├── RuleEngine.php
│   ├── CaseManagement.php
│   └── config/
│       └── severity_rules.json
│
├── infrastructure/
│   ├── docker/
│   │   └── docker-compose.yml
│   ├── kafka/
│   │   └── topics.yaml
│   └── s3/
│       └── lifecycle-policy.json
│
└── .gitignore
```

---

## Frontend — React Logging Layer

### IndexedDB Local Queue

The browser stores all activity logs locally before sync. This guarantees **zero log loss** even on network failure or tab close.

```
Database:     fintech_activity_logs
Object Store: logs
Key Path:     audit_id

Indexed Fields:
  timestamp · event_type · user.id · severity
  channel · synced · retry_count

Stored Per Log:
  audit_id              ← Primary UUID
  timestamp             ← ISO 8601
  event_type            ← e.g. UI_ACTIVITY, API_CALL
  action                ← e.g. CLICK, SUBMIT, NAVIGATE
  severity              ← LOW | MEDIUM | HIGH | CRITICAL
  user.id / user.role
  channel               ← WEB | MOBILE | API
  trace.trace_id
  risk_analysis.risk_score
  risk_analysis.anomaly_detected
  payload               ← Full JSON log
  synced                ← Boolean (default: false)
  retry_count           ← Number (default: 0)
  last_sync_attempt_at
  batch_id              ← Assigned during sync

Retention Rules:
  Auto-delete synced logs  → after 7 days
  Hard delete all logs     → after 30 days
  Storage safety cap       → 50 MB
```

### Sync Worker

```
Sync Triggers:
  • Every 30 seconds
  • OR unsynced log count ≥ 100
  • OR user comes back online
  • OR before tab close (visibilitychange / beforeunload)

Batch Payload:
  {
    "batch_id":   "uuid",
    "total_logs": 42,
    "logs":       [ ...full JSON logs ]
  }

Integrity Hash (per log):
  SHA256(audit_id + timestamp + user.id + trace.trace_id)

On Success:
  • Mark logs synced = true
  • Store server acknowledgement timestamp + trace_id

On Failure — Exponential Backoff:
  Retry 1  →  30 sec
  Retry 2  →   2 min
  Retry 3  →   5 min
  Retry 4  →  10 min
  Retry 5  →  Stop & flag as failed
```

---

## Backend — Laravel Ingestion API

```
Endpoint:  POST /api/v1/activity/logs/batch

Pipeline:
  1. Validate JSON schema
  2. Verify SHA256 integrity hash
  3. Rate limit check (per client / IP)
  4. Stamp server_received_at timestamp
  5. Attach server-side trace_id
  6. Push batch → Queue (Redis / DB Queue)
  7. Return 200 OK immediately (non-blocking)

Queue Worker (async):
  • Consume batch from queue
  • Persist structured columns → MySQL (hot)
  • Serialize full JSON → S3 (cold archive)
  • Fan-out risk feed → Kafka → FRM consumer
```

---

## Message Broker Layer

Kafka (or RabbitMQ) acts as the durable, fault-tolerant backbone between ingestion and processing.

| Topic | Consumer | Purpose |
|---|---|---|
| `activity.logs` | Log Processor Worker | MySQL + S3 persistence |
| `activity.risk` | FRM Consumer | Risk scoring + rule evaluation |
| `activity.alerts` | Alert Dispatcher | Email / SMS / Webhook dispatch |

---

## Storage Strategy

| Layer | Technology | Retention | Purpose |
|---|---|---|---|
| Client | IndexedDB | 7–30 days | Offline-safe local queue |
| Hot | MySQL (RDS) | 30–90 days | Searchable operational store |
| Cold | AWS S3 (WORM) | 7 years | Immutable compliance archive |

### MySQL Hot Storage

**Table:** `activity_logs`

```sql
-- Core (all indexed)
id               AUTO_INCREMENT PK
audit_id         UNIQUE INDEX
timestamp        INDEX
event_type       INDEX
action           INDEX
severity         INDEX
channel          INDEX

-- User & Risk
user_id          INDEX
user_role
risk_score       INDEX
anomaly_detected BOOL INDEX

-- Distributed Tracing
trace_id         INDEX
span_id
parent_span_id

-- Device
ip_address       INDEX
user_agent
screen_resolution
timezone

-- Business Context
loan_id          INDEX
product_type
amount
currency

-- Performance
api_duration_ms
total_transaction_time_ms

-- Full payload
payload          JSON column (complete original log)

-- Partitioning
PARTITION BY RANGE (YEAR_MONTH)
Hot retention:   30–90 days → archive to S3
```

**Common fraud detection queries:**

```sql
-- Rapid click detection
WHERE event_type = 'UI_ACTIVITY'
AND   payload->'$.behavior.rapid_click_flag' = true

-- High risk users
WHERE risk_score > 70
ORDER BY risk_score DESC

-- Multi-IP anomaly (geo fraud)
GROUP BY user_id
HAVING COUNT(DISTINCT ip_address) > 3
```

### S3 Cold Archive

```
Bucket per application:
  s3://mms-activity-logs/
  s3://frm-activity-logs/
  s3://wegoX-activity-logs/
  s3://issureX-activity-logs/

Directory structure:
  s3://wegoX-activity-logs/
    └── 2026/
        └── 01/
            └── 04/
                └── user_42/
                    ├── 10_11.json.gz
                    ├── 11_12.json.gz
                    └── 12_13.json.gz

File format:
  • Batched JSON activity logs, sorted by timestamp
  • gzip compressed
  • Immutable once written (S3 Object Lock / WORM)

Lifecycle policy:
  Day 0–90    → S3 Standard
  Day 90+     → S3 Glacier Instant Retrieval
  Day 2555+   → Delete (7-year compliance window)
```

---

## FRM — Fraud Risk Management

The FRM microservice consumes the risk feed from Kafka and runs every event through a configurable rule engine.

```
FRM Processing Chain:

  P7.1  Consume Event      ← From Kafka / MQ
          ↓
  P7.2  Evaluate Rules     ← Active rule set (severity_rules.json)
          ↓
  P7.3  Compute Risk Score → LOW / MEDIUM / HIGH / CRITICAL
          ↓
  P7.4  Threshold Check    ← score ≥ threshold?
          ↓
  P7.5  Alert + Case       ← Email · SMS · Webhook + Case ticket
```

**Risk Severity Matrix**

| Score | Level | Action |
|---|---|---|
| 0–30 | `LOW` | Log only |
| 31–50 | `MEDIUM` | Flag for review |
| 51–70 | `HIGH` | Alert ops team |
| 71–100 | `CRITICAL` | Block session + open fraud case |

**Fraud Rule Examples**

```
Rule 1 — Rapid Click
  click_speed_ms < 100  →  risk_score +10

Rule 2 — Multiple IP Change
  3+ distinct IPs within 10 minutes  →  risk_score +25

Rule 3 — Repeated API Failures
  status_code ≥ 400 repeated 5× in session  →  anomaly_detected = true

Rule 4 — Bot-like Behavior
  mouse_movement_pattern = 'BOT_LIKE'  →  risk_score +30

Escalation:
  IF risk_score > 70     → CRITICAL alert + open Case Management ticket
  IF anomaly_detected    → Immediate SIEM notification
```

---

## SIEM & Alert Engine

| Channel | Trigger |
|---|---|
| Email | risk_score > 70 |
| SMS | CRITICAL severity events |
| Webhook | All HIGH / CRITICAL events |
| Dashboard | Real-time risk feed |
| Case Ticket | Auto-opened for fraud investigation |

---

## Log JSON Structure

Every log object follows this canonical schema:

```json
{
  "audit_id":   "uuid-v4",
  "timestamp":  "2026-01-04T10:32:15.123Z",
  "event_type": "UI_ACTIVITY",
  "severity":   "LOW",
  "channel":    "WEB",

  "user": {
    "id":         "user_42",
    "role":       "MERCHANT_ADMIN",
    "session_id": "sess_abc123"
  },

  "trace": {
    "trace_id":       "trace_xyz",
    "span_id":        "span_001",
    "parent_span_id": null
  },

  "context": {
    "page":                "/dashboard/merchants/create",
    "action":              "CLICK",
    "coordinates":         { "x": 960, "y": 820 },
    "duration_on_page_ms": 43000
  },

  "behavior": {
    "click_speed_ms":             210,
    "keystroke_interval_avg_ms":  260,
    "mouse_movement_pattern":     "NORMAL",
    "focus_switch_count":         1,
    "rapid_click_flag":           false
  },

  "api_call": {
    "endpoint":    "/api/v1/merchants/create",
    "method":      "POST",
    "status_code": 201,
    "duration_ms": 640,
    "retry_count": 0
  },

  "device": {
    "ip_address":        "[masked]",
    "user_agent":        "Chrome 120",
    "screen_resolution": "1920x1080",
    "timezone":          "Asia/Kolkata"
  },

  "risk_analysis": {
    "risk_score":      5,
    "risk_flags":      [],
    "anomaly_detected": false
  },

  "compliance": {
    "data_residency":     "IN",
    "pii_exposed":        false,
    "log_integrity_hash": "sha256-merchantcreate-xyz789"
  }
}
```

---

## Compliance & Security Controls

| Control | Detail |
|---|---|
| PII masking | Sensitive fields masked before persistence — no raw PII stored |
| Integrity hash | SHA256 per log, verified on ingestion |
| Immutable archive | S3 Object Lock (WORM) — logs cannot be altered after write |
| Retention | 7-year audit trail (regulatory compliance) |
| Transport | HTTPS only — no plaintext transmission permitted |
| Access control | RBAC on all storage layers |
| Service auth | mTLS for all service-to-service communication |
| Backup | Daily snapshot + weekly offline backup |
| Observability | OpenTelemetry trace correlation for full request lineage |

---

## On-Prem Deployment

For environments without AWS, the full architecture runs on-premises:

```
Application Services
        ↓
Log Ingestion API  (Microservice)
        ↓
Message Broker  (Kafka / RabbitMQ)
        ↓
─────────────────────────────────────────────
│           Processing Layer                │
│  1. Log Processor Worker                  │
│  2. Risk Scoring Engine                   │
│  3. Rule Evaluation Engine                │
│  4. Alert Dispatcher Service              │
│  5. Case Management Service               │
─────────────────────────────────────────────
        ↓
Storage Layer
   ├── MySQL         (Hot Searchable Logs)
   ├── Elasticsearch (Optional SIEM Index)
   └── Archive Svc   → NAS / NFS / MinIO

Archive path:
  /mms/log-archive/YYYY/MM/DD/user_id/HH_HH.json.gz
```

**On-Prem Security Controls**

- Append-only WORM storage (immutable write)
- SHA256 integrity verification per log
- RBAC — role-based access control
- mTLS — service-to-service authentication
- RAID storage redundancy
- Daily snapshot + weekly offline backup

---

## Log Volume & Cost Analysis

**Assumptions:** 100 users × 50 logs/day × 3 KB avg

```
Daily logs    →   5,000
Daily data    →   15 MB
Monthly data  →  450 MB
Yearly data   →    5.4 GB
```

### Option A — MySQL on AWS RDS (Mumbai, db.t3.micro)

| Component | Monthly | Yearly |
|---|---|---|
| RDS db.t3.micro | ₹1,500–₹2,000 | — |
| Storage (10 GB) | ₹80–₹120 | — |
| Backup | ₹150 | — |
| **Total** | **₹1,800–₹2,300** | **₹22,000–₹27,000** |

### Option B — S3 Cold Archive Only

| Component | Monthly | Yearly |
|---|---|---|
| S3 Storage (~0.45 GB) | ₹10–₹15 | — |
| Requests | ₹10 | — |
| Transfer | ₹10 | — |
| **Total** | **₹25–₹40** | **₹300–₹500** |

> **Recommended:** Use both — MySQL for hot operational queries (30–90 days) + S3 for long-term compliance archiving (7 years).

---

## Getting Started

### Prerequisites

- Node.js 18+ · npm / yarn (frontend)
- PHP 8.1+ · Composer (backend)
- MySQL 8+ or PostgreSQL 14+
- Redis (queue backend)
- Kafka or RabbitMQ (message broker)
- AWS credentials or MinIO (S3-compatible storage)
- Docker & Docker Compose (recommended)

### Docker Compose

```bash
# 1. Clone
git clone https://github.com/your-org/fintech-activity-logging.git
cd fintech-activity-logging

# 2. Configure environment
cp backend/.env.example backend/.env
cp frontend/.env.example frontend/.env

# 3. Start all services
docker compose up --build

# 4. Run migrations + seed
docker compose exec backend php artisan migrate --seed

# 5. Start the queue worker
docker compose exec backend php artisan queue:work

# Frontend  →  http://localhost:3000
# API docs  →  http://localhost:8000/api/documentation
```

### Manual Setup

```bash
# Backend
cd backend && composer install
cp .env.example .env && php artisan key:generate
php artisan migrate --seed
php artisan queue:work

# Frontend
cd frontend && npm install && npm start
```

---

## Environment Variables

### Backend (`backend/.env`)

| Variable | Description |
|---|---|
| `DB_HOST` / `DB_DATABASE` | MySQL connection details |
| `REDIS_HOST` | Redis for queue + rate limiting |
| `QUEUE_CONNECTION` | `redis` or `database` |
| `LOG_INTEGRITY_SECRET` | Secret for SHA256 hash verification |
| `KAFKA_BROKER` | Kafka bootstrap server address |
| `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` | S3 credentials |
| `AWS_DEFAULT_REGION` | e.g. `ap-south-1` |
| `AWS_BUCKET_PREFIX` | Prefix for per-app log buckets |
| `FRM_RISK_THRESHOLD` | Score at which CRITICAL alert fires (default: `70`) |
| `ALERT_EMAIL` | Ops team email for critical alerts |
| `ALERT_SMS_MOBILE` | Ops team mobile for SMS alerts |

### Frontend (`frontend/.env`)

| Variable | Description |
|---|---|
| `REACT_APP_API_BASE_URL` | Laravel API base URL |
| `REACT_APP_SYNC_INTERVAL_MS` | Sync interval in ms (default: `30000`) |
| `REACT_APP_BATCH_SIZE` | Max logs per batch (default: `100`) |
| `REACT_APP_INDEXEDDB_QUOTA_MB` | Storage safety cap (default: `50`) |

---

## Commit Message Convention

```
feat:      new feature
fix:       bug fix
docs:      documentation update
refactor:  code restructure without behaviour change
chore:     build / config changes
security:  security-related change

Examples:
  feat(logging):    add retry exponential backoff to sync worker
  fix(frm):         correct risk score threshold for CRITICAL boundary
  docs(arch):       add DFD Level 2 ingestion pipeline diagram
  security(api):    add SHA256 integrity hash verification on batch ingest
  chore(s3):        update lifecycle policy for Glacier transition
```

---

## Contributing

1. Fork the repository
2. Create a feature branch: `git checkout -b feat/my-feature`
3. Follow the [commit convention](#commit-message-convention)
4. Open a pull request against `main`

---

## License

[MIT](LICENSE) © Your Organization

---

<p align="center">
  <code>React · Laravel · MySQL · AWS S3 · Kafka · FRM · SIEM · On-Prem</code><br/>
  <sub>Fintech Activity Logging & SIEM Architecture v2.0 — Production Reference</sub>
</p>
