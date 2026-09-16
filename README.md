# Sentryon — Fraud Detection for Online Transactions

This package contains a **working interactive demo** of the platform (`fraud-detection-app.html`) plus the reference design for the full production stack described in the brief. The live demo runs entirely in the browser: it generates a synthetic transaction pool and scores it with real statistical anomaly-detection math (robust z-scores, device-recognition flags, location-density estimation) so every module — dashboard, alerts, graph, reports — reacts to genuinely computed scores. It does not require a server, a database, or installing Python/TensorFlow to explore.

The sections below describe how the same product is architected as a full React + Flask/FastAPI + PostgreSQL + ML system, for implementation beyond this demo.

## What's included

| File | Description |
|---|---|
| `fraud-detection-app.html` | Complete interactive application (landing page, auth, dashboard, transaction monitoring, fraud detection upload, ML model comparison, graph analysis, alert center, admin panel, reports) |
| `sample_transactions.csv` | 200-row sample dataset for the Fraud Detection module's upload feature |
| `README.md` | This document — architecture, database schema, API design, installation and deployment guidance |

## Folder structure (production system)

```
fraud-detection-platform/
├── frontend/                  React + Tailwind SPA
│   ├── src/
│   │   ├── pages/              Landing, Auth, Dashboard, Transactions, Detection,
│   │   │                       Models, Graph, Alerts, Admin, Reports, About, Contact
│   │   ├── components/         Charts, tables, cards, nav
│   │   ├── hooks/               useAuth, useTransactions, useAlerts
│   │   └── api/                 REST client
├── backend/                    Flask (or FastAPI) service
│   ├── app/
│   │   ├── routes/              auth.py, transactions.py, detection.py, alerts.py, reports.py, admin.py
│   │   ├── models/               SQLAlchemy models matching the schema below
│   │   ├── services/             scoring_service.py, alert_service.py, report_service.py
│   │   └── auth/                  JWT issuing/verification, role-based access control
│   └── ml/
│       ├── train_isolation_forest.py
│       ├── train_lof.py
│       ├── train_autoencoder.py
│       ├── train_xgboost.py
│       └── inference_pipeline.py    combines the four model outputs into one score
├── database/
│   └── schema.sql
└── docs/
    └── api.md
```

## Database schema

```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(120) NOT NULL,
  email VARCHAR(160) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  role VARCHAR(20) NOT NULL DEFAULT 'User',      -- Admin | Analyst | User
  status VARCHAR(20) NOT NULL DEFAULT 'Active',
  created_at TIMESTAMP DEFAULT now()
);

CREATE TABLE transactions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  transaction_ref VARCHAR(40) UNIQUE NOT NULL,
  user_id UUID REFERENCES users(id),
  amount NUMERIC(12,2) NOT NULL,
  merchant VARCHAR(160),
  device_id VARCHAR(80),
  location VARCHAR(120),
  occurred_at TIMESTAMP NOT NULL,
  created_at TIMESTAMP DEFAULT now()
);

CREATE TABLE fraud_scores (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  transaction_id UUID REFERENCES transactions(id),
  isolation_forest_score NUMERIC(5,2),
  lof_score NUMERIC(5,2),
  autoencoder_score NUMERIC(5,2),
  xgboost_score NUMERIC(5,2),
  ensemble_score NUMERIC(5,2) NOT NULL,
  risk_level VARCHAR(20) NOT NULL,               -- safe | low | medium | high | critical
  explanation TEXT,
  scored_at TIMESTAMP DEFAULT now()
);

CREATE TABLE alerts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  transaction_id UUID REFERENCES transactions(id),
  alert_type VARCHAR(60) NOT NULL,               -- threshold | spike | new_device | location
  severity VARCHAR(20) NOT NULL,
  acknowledged BOOLEAN DEFAULT false,
  created_at TIMESTAMP DEFAULT now()
);

CREATE TABLE reports (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  generated_by UUID REFERENCES users(id),
  report_type VARCHAR(60) NOT NULL,              -- fraud | monthly | risk_assessment
  format VARCHAR(10) NOT NULL,                   -- csv | pdf | xlsx
  file_path VARCHAR(255),
  created_at TIMESTAMP DEFAULT now()
);

CREATE TABLE audit_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id),
  action VARCHAR(255) NOT NULL,
  created_at TIMESTAMP DEFAULT now()
);
```

## API endpoints

```
Auth
  POST   /api/auth/register
  POST   /api/auth/login
  POST   /api/auth/forgot-password
  POST   /api/auth/refresh

Transactions
  GET    /api/transactions            ?search=&risk=&sort=&page=
  GET    /api/transactions/:id
  POST   /api/transactions/export     -> CSV/PDF

Fraud detection
  POST   /api/detection/upload        multipart CSV -> scored rows
  GET    /api/detection/:batchId

Models
  GET    /api/models/metrics          accuracy/precision/recall/F1/ROC-AUC per model

Alerts
  GET    /api/alerts                  ?severity=&acknowledged=
  POST   /api/alerts/:id/acknowledge

Admin
  GET    /api/admin/users
  POST   /api/admin/users
  DELETE /api/admin/users/:id
  GET    /api/admin/thresholds
  PUT    /api/admin/thresholds
  GET    /api/admin/logs

Reports
  POST   /api/reports/generate        {type, format}
  GET    /api/reports/:id/download
```

All endpoints except `/auth/*` require a `Bearer` JWT; role checks (`Admin`/`Analyst`/`User`) are enforced per route via middleware.

## ML pipeline (reference design)

1. **Feature engineering** — per-transaction features: amount z-score against the user's rolling baseline, device-recognition flag, location frequency/rarity, hour-of-day, velocity (transactions per hour).
2. **Unsupervised layer** — Isolation Forest and Local Outlier Factor (scikit-learn) flag statistical outliers without needing labeled fraud examples.
3. **Deep layer** — an autoencoder (TensorFlow/Keras) trained on legitimate transactions; high reconstruction error indicates anomalous behavior.
4. **Supervised layer** — XGBoost trained on historical labeled fraud/legitimate transactions for a precision-focused signal.
5. **Ensemble** — a weighted blend of the four scores produces the 0–100 `ensemble_score` stored against each transaction; weights are tunable, mirroring the sliders in the Admin Panel.
6. **Graph module (optional/advanced)** — a Graph Neural Network over a user–device–card–merchant graph to catch multi-hop fraud rings that per-transaction scoring misses.

## Installation guide (reference backend)

```bash
# Backend
cd backend
python -m venv venv && source venv/bin/activate
pip install flask flask-sqlalchemy flask-jwt-extended scikit-learn tensorflow xgboost pandas
createdb fraud_detection
psql fraud_detection < ../database/schema.sql
flask run

# Frontend
cd frontend
npm install
npm run dev
```

## Deployment notes

- Frontend: static build deployed to Vercel/Netlify/S3+CloudFront.
- Backend: containerize with Docker, deploy to Render/Railway/ECS behind HTTPS; environment variables for `JWT_SECRET`, `DATABASE_URL`, `MODEL_PATH`.
- Database: managed PostgreSQL (RDS/Cloud SQL); run `schema.sql` via migration tool (Alembic).
- ML models: train offline, persist with `joblib`/`SavedModel`, load in the `ml/inference_pipeline.py` service; retrain on a schedule as new labeled fraud data arrives.
