# Financial Fraud Detection Platform

A real-time financial fraud detection system that streams transactions through Apache Kafka, scores them using rule-based logic and machine learning, and visualises results on a live Next.js dashboard.

---

## Architecture

```
Producer (Faker)
     │
     ▼ Kafka topic: transactions_in
FastAPI Consumer ──► PostgreSQL (persist)
     │           ──► Redis (velocity + 3-strike)
     │           ──► MLflow (load model)
     │
     ▼ WebSocket /ws/alerts
Next.js Dashboard
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Streaming | Apache Kafka + Zookeeper (Docker) |
| Backend API | FastAPI + Python 3.10 |
| Database | PostgreSQL 15 (Docker) |
| Cache | Redis 7 |
| ML Tracking | MLflow 2.8 |
| ML Models | IsolationForest → RandomForestClassifier |
| Frontend | Next.js 16 + React 19 + Recharts + Tailwind CSS |

---

## Fraud Scoring Logic

Each transaction is scored out of 100:

| Rule | Points |
|---|---|
| Amount > $10,000 | +50 |
| Amount > $5,000 | +25 |
| Amount > $1,000 | +10 |
| Velocity > 100 tx/hr (same user) | +40 |
| Velocity > 50 tx/hr | +20 |
| ML anomaly flag | +35 |

- Score **> 40** → `MEDIUM` alert
- Score **≥ 80** → `CRITICAL` alert + notification dispatched
- **3 CRITICAL** alerts for same user → status set to `BLOCKED`

---

## ML Pipeline

1. **Phase 1 — Unsupervised** (< 50 labeled samples): trains `IsolationForest` on amount + hour features, registered in MLflow as `fraud_iforest`
2. **Phase 2 — Supervised** (≥ 50 labeled samples): trains `RandomForestClassifier` on analyst-labeled fraud data, registered as `fraud_classifier`
3. Restart the consumer after training to load the latest model

---

## Project Structure

```
fraud/
├── docker-compose.yml        # PostgreSQL, Kafka, Zookeeper, Redis
├── requirements.txt          # Python dependencies
├── producer.py               # Fake transaction generator (~2–10 tx/sec)
├── start.ps1                 # One-command start (Windows PowerShell)
├── stop.ps1                  # One-command stop
├── .env.example              # Environment variable template
├── db/
│   └── init.sql              # Schema: transactions, alerts, audit_logs
├── consumer/
│   ├── main.py               # FastAPI app + Kafka consumer + WebSocket
│   ├── models.py             # SQLAlchemy ORM models
│   ├── enrichment.py         # Redis velocity counters
│   ├── scoring.py            # Risk score calculator
│   └── notifications.py      # Critical alert dispatcher (console / SMS hook)
├── ml/
│   └── train.py              # MLflow training script
└── frontend/
    └── src/app/
        ├── page.tsx           # Dashboard UI
        ├── globals.css        # Dark glassmorphism theme
        └── layout.tsx         # Root layout
```

---

## Port Assignments

| Service | Port | Notes |
|---|---|---|
| FastAPI | 8001 | Remapped from 8000 |
| Next.js | 3002 | Remapped from 3000 |
| PostgreSQL | 5433 | Remapped from 5432 |
| Kafka | 29092 | External access |
| Zookeeper | 2181 | |
| MLflow | 5000 | |
| Redis | 6379 | |

---

## Setup

### Prerequisites
- Docker Desktop (running)
- Python 3.10+
- Node.js 18+

### 1. Clone and configure

```bash
git clone https://github.com/YOUR_USERNAME/financial-fraud-detector.git
cd financial-fraud-detector
cp .env.example .env
# Edit .env and set your POSTGRES_USER and POSTGRES_PASSWORD
```

### 2. Start everything (Windows PowerShell)

```powershell
.\start.ps1
```

This will:
- Create a Python virtual environment and install dependencies
- Run `npm install` for the frontend
- Start Docker containers (PostgreSQL, Kafka, Zookeeper, Redis)
- Apply DB migrations
- Launch MLflow, FastAPI, Next.js, and the producer in separate windows

### 3. Open the dashboard

| URL | Purpose |
|---|---|
| http://localhost:3002 | Live dashboard |
| http://127.0.0.1:8001/docs | FastAPI Swagger UI |
| http://localhost:5000 | MLflow experiment tracker |

### 4. Stop everything

```powershell
.\stop.ps1
```

---

## Manual Start (4 terminals)

```powershell
# Terminal 1 — Infrastructure
docker compose up -d

# Terminal 2 — MLflow
.\venv\Scripts\Activate.ps1
python -m mlflow server --host 0.0.0.0 --port 5000

# Terminal 3 — FastAPI Consumer
cd consumer
..\venv\Scripts\Activate.ps1
python -m uvicorn main:app --host 0.0.0.0 --port 8001

# Terminal 4 — Next.js Frontend
cd frontend
npm run dev -- --port 3002

# Terminal 5 — Transaction Producer
.\venv\Scripts\Activate.ps1
python producer.py
```

---

## Dashboard Features

- **Live Feed** — real-time transaction stream via WebSocket, pre-loaded with last 50 from DB on page load
- **Critical Alerts filter** — click the Critical Alerts card to show only score ≥ 80 transactions
- **Expand transaction** — click any card to see location, trigger traces, malicious tally, analyst label
- **Analyst labeling** — mark transactions as True Fraud or False Positive to build the ML training set
- **Live Risk Topology** — area chart of the last 15 risk scores
- **Generate Final Report** — downloads a CSV of all labeled transactions

---

## Training the ML Model

Once you have labeled 50+ transactions in the dashboard:

```powershell
.\venv\Scripts\Activate.ps1
python ml/train.py
```

Then restart the FastAPI consumer to load the new model. The model will automatically switch from IsolationForest to RandomForestClassifier.

---

## Environment Variables

Copy `.env.example` to `.env` and configure:

```env
POSTGRES_USER=your_db_user
POSTGRES_PASSWORD=your_db_password
POSTGRES_DB=fraud_db
DATABASE_URL=postgresql://your_db_user:your_db_password@localhost:5433/fraud_db
KAFKA_BROKER=localhost:29092
MLFLOW_URL=http://localhost:5000
```
