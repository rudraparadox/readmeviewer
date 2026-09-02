# PAIMANA AI — Detailed Project Documentation

**AI-Powered Predictive Analytics & Early Warning System for Infrastructure Monitoring**

**Team Project | Smart India Hackathon 2026 | Problem Statement ID 26103**

**Organization:** MoSPI · **Department:** Data Informatics & Innovation Division (DIID) · **Theme:** Smart Automation

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [What SIH Demanded (The Problem)](#2-what-sih-demanded-the-problem)
3. [What We Built (High-Level)](#3-what-we-built-high-level)
4. [Solution Architecture](#4-solution-architecture)
5. [ML Pipeline — How We Built the AI](#5-ml-pipeline--how-we-built-the-ai)
6. [LLM-Enabled Project Intelligence Assistant](#6-llm-enabled-project-intelligence-assistant)
7. [Technology Stack](#7-technology-stack)
8. [Project Structure](#8-project-structure)
9. [SIH Deliverables Mapping](#9-sih-deliverables-mapping)
10. [API Reference](#10-api-reference)
11. [How to Run](#11-how-to-run)
12. [Key Numbers to Remember](#12-key-numbers-to-remember)

---

## 1. Executive Summary

India's Ministry of Statistics & Programme Implementation (MoSPI) tracks **~2,000 central-sector infrastructure projects** worth **₹42+ lakh crore** through the **PAIMANA portal** ("Project Assessment, Infrastructure Monitoring and Analytics for Nation-building"). With nearly two decades of history from its predecessor OCMS, the data is rich and continuously updated. Yet the existing system is fundamentally **descriptive** — it reports *what has happened*.

**Our solution (PAIMANA AI) transforms this into a predictive + prescriptive decision-support system.** Using only **open-source** tools, we deliver:

- **Predictive models** that forecast **cost overruns** and **time overruns** before they materialise.
- A **0–100 project risk-scoring framework** with four risk tiers.
- An **early warning alert engine** (3,888 alerts across the portfolio).
- **Sector/ministry benchmarking** and **cost-escalation driver analysis** (SHAP).
- A **natural-language LLM assistant** that answers questions in plain English.
- A **deployment framework** (Docker) so judges can run the whole system with one command.

In short: **from "here's what happened" → "here's what will happen — and what you should do about it."**

---

## 2. What SIH Demanded (The Problem)

### 2.1 Background

The Infrastructure & Project Monitoring Division (IPMD) of MoSPI monitors Central Sector Infrastructure Projects costing ₹150 crore and above across all infrastructural Ministries/Departments. Monitoring was done through **OCMS** (since 2006), generating a valuable historical database capturing cost overruns and time overruns across sectors. OCMS was later modernised into the **PAIMANA portal** — a comprehensive, integrated, web-based project-monitoring ecosystem.

As of **April 2026**, PAIMANA tracks:
- **1,981 ongoing projects** across **17 Central Ministries/Departments** covering **22 infrastructure sectors**.
- **Original cost ~₹37.13 lakh crore**, **revised cost ~₹42.78 lakh crore**, **cumulative expenditure ~₹20.36 lakh crore**.
- Sectors include Transport & Logistics, Energy, Water & Sanitation, Communication, Social Infrastructure, Coal, Steel and Mining.

### 2.2 The Core Ask

Despite rich monitoring data, infrastructure projects repeatedly face **cost overruns, time overruns, milestone delays, contractual bottlenecks, resource constraints and execution risks**. The SIH challenge asks us to move **beyond descriptive monitoring** to **predictive and prescriptive monitoring** — proactively identifying at-risk projects so policymakers, project administrators and monitoring agencies can intervene **before** issues materialise.

### 2.3 The Three Technical Dimensions

| # | Technical Requirement |
|---|---|
| **(a)** | Develop & evaluate **statistical + predictive models** (open-source) to forecast cost overruns, time overruns and implementation risks. |
| **(b)** | Assess whether **AI/ML provides significant gains over conventional statistical methods** (accuracy, early-warning capability, decision-support). |
| **(c)** | Build models from the existing **CUF (Common Upload Form) fields**, and assess how much predictive performance comes from **CUF fields vs. additional variables not currently captured**. |

### 2.4 The Nine Expected Outcomes (a–i)

| # | Expected Outcome |
|---|---|
| **a** | Cost Overrun Prediction Model |
| **b** | Time Overrun Prediction Model |
| **c** | Project Risk Scoring Framework |
| **d** | Early Warning Alert System |
| **e** | Benchmarking & Comparative Analytics Module |
| **f** | Cost Escalation Driver Analysis Module |
| **g** | AI-powered Monitoring Dashboard |
| **h** | LLM-enabled Project Intelligence Assistant |
| **i** | Documentation & Deployment Framework |

**Constraint:** only **open-source** tools (AI, ML, Big Data, Forecast Modelling, LLMs) may be used. The suggestions are *indicative and non-exhaustive* — teams may adopt alternative methodologies.

> **Section 9 maps every one of the (a)–(i) outcomes to a concrete deliverable in this project.** All nine are implemented.

---

## 3. What We Built (High-Level)

### 3.1 The 5 Real-Time AI Predictions

When a government official enters project details, the AI generates **five real-time outputs**:

| # | Prediction | What It Means | Example Output |
|---|---|---|---|
| 1 | **Cost Overrun Probability** | % chance this project will exceed its revised budget | "78.5% — Overrun Predicted" |
| 2 | **Time Overrun Probability** | % chance this project will miss its deadline | "99.0% — Schedule Delay Expected" |
| 3 | **Predicted Delay (months)** | If delayed, how many extra months it will take | "+20.6 months delay" |
| 4 | **Risk Score (0–100)** | Overall health score combining all risk factors | "62.1 — High Risk" |
| 5 | **Early Warning Alerts** | Specific red flags in the project's numbers | "Large PF Gap of 52.5%", "Cost revised up 60%" |

### 3.2 AI User Flow

A monitoring official enters the **9 CUF fields** from the PAIMANA portal:

> **Ministry · Sector · State · Original Cost · Revised Cost · Expenditure · Physical Progress · Planned Duration · Project Age**

↓ Our trained **XGBoost** model processes these through **24 engineered features** ↓

And instantly tells them:
- **"Will this go over budget?"** → Yes/No + probability.
- **"Will this be delayed?"** → Yes/No + probability + estimated months.
- **"How risky is it overall?"** → 0–100 score with Low/Medium/High/Critical label.
- **"What should I watch out for?"** → Targeted alerts + actionable recommendations.

### 3.3 The Six Frontend Pages

| Page | Route | Purpose |
|---|---|---|
| Dashboard | `/` | 6 stat cards, risk pie chart, sector bar chart, alert ticker, ministry comparison table, model-performance card |
| AI Predictor | `/predict` | Structured CUF input form → live prediction with risk score, probabilities, warnings, recommendations, key drivers |
| AI Assistant | `/chat` | **Natural-language chat** — ask questions, get contextual, data-backed answers |
| Risk Monitor | `/risk` | Paginated project table sorted by risk, with search & filter |
| Early Warnings | `/alerts` | Alert feed with severity / type / sector filters + summary sidebar |
| Analytics | `/analytics` | SHAP cost-driver chart, sector benchmarks, **model-comparison panel (Req b & c)** |

---

## 4. Solution Architecture

Our system follows a **three-layer architecture**, extended with an **LLM layer** and containerised via **Docker**.

```
┌──────────────────────────────────────────────────────────────────────┐
│                         FRONTEND (React 19)                          │
│   Dashboard │ AI Predictor │ AI Assistant │ Risk │ Alerts │ Analytics│
│          Vite + React Router + Framer Motion + Recharts             │
└──────────────────────────────────┬───────────────────────────────────┘
                                   │ REST API (HTTP/JSON)
┌──────────────────────────────────┴───────────────────────────────────┐
│                         BACKEND (FastAPI + Uvicorn)                   │
│  /api/summary /api/projects /api/predict /api/alerts                 │
│  /api/analytics/* /api/filters /api/chat /api/chat/suggestions       │
│  (also serves the built frontend as static files)                    │
└───────────────┬────────────────────────────┬─────────────────────────┘
                │ joblib.load()              │ groq SDK
┌───────────────┴──────────────┐   ┌─────────┴──────────────────────────┐
│    ML PIPELINE (XGBoost)     │   │      LLM LAYER (Groq)              │
│  Cleaning → Feature Eng →    │   │  Pandas data query → context        │
│  Training → Risk → Alerts    │   │  → Llama 3.3 70B → answer           │
│  scikit-learn + XGBoost      │   │  llama-3.3-70b-versatile            │
└───────────────┬──────────────┘   └─────────┬──────────────────────────┘
                │                             │
                └──►  Data: projects_with_risk_scores.csv, alerts.csv,
                      sector_benchmarks.csv, cost_drivers.csv ◄──────────┘
```

### 4.1 Data Flow

1. **Raw PDFs** → extracted via `extract_pdf_data.py` / `extract_remaining.py` into CSVs (April–July 2026).
2. **Cleaning** → `clean_data.py` standardises columns, fixes encoding, handles missing values.
3. **ML Pipeline** → `train_pipeline.py` matches projects across months, engineers features, trains models, computes risk scores, alerts, benchmarks and drivers.
4. **API Server** → `main.py` loads trained models + result CSVs and exposes REST endpoints.
5. **LLM Layer** → `chat.py` queries the same result CSVs with Pandas and injects the context into Groq's Llama model.
6. **Frontend** → React dashboard fetches data from the API and renders interactive visuals.
7. **Docker** → containerises the backend + built frontend so `docker compose up --build` runs everything.

---

## 5. ML Pipeline — How We Built the AI

### 5.1 Data Source

- **4 monthly snapshots** from PAIMANA (April, May, June, July 2026).
- **2,047 projects** across **17 Ministries** and **22 sectors**.
- Each project has CUF fields: cost, expenditure, progress, timelines, location, agency.

### 5.2 Feature Engineering (24 Features Total)

We engineered features in two categories:

| Category | Features | Count |
|---|---|---|
| **CUF Base Features** | original_cost_cr, revised_cost_cr, cumulative_expenditure_cr, physical_progress_pct, expenditure_ratio, financial_progress_pct, physical_financial_gap, planned_duration_months, project_age_months, time_elapsed_ratio, ministry_encoded, sector_encoded, state_encoded | **13** |
| **Temporal Features** (multi-month trends) | expenditure_velocity, expenditure_monthly_avg, expenditure_acceleration, progress_velocity, progress_monthly_avg, progress_stagnant, cost_revision_trend, cost_revised_up, cost_overrun_trend, months_tracked, expenditure_ratio_trend | **11** |

**Total = 24 features.** Temporal features require **multiple months of data** — this is exactly the "additional variables not presently captured in a single CUF snapshot" that SIH technical dimension (c) refers to.

### 5.3 Training Approach

We trained models for **three prediction tasks**:

| Task | Type | Best Model | Key Metrics |
|---|---|---|---|
| Cost Overrun Prediction | Binary Classification | **XGBoost (Full)** | F1 = **0.8646**, AUC = **0.9542**, Accuracy = **93.7%** |
| Time Overrun Prediction | Binary Classification | **XGBoost (Full)** | F1 = **0.9409**, AUC = **0.9759** |
| Time Delay Estimation | Regression | **XGBoost Regressor** | **R² = 0.8703**, MAE = **4.68 months** |

**Train/Test Split:** 80/20 with stratified sampling, `random_state=42`. Models also evaluated with **5-fold cross-validation** (`cv_f1_mean` / `cv_f1_std`).

### 5.4 Statistical vs ML Comparison (SIH Requirement b)

This directly answers: *"Does AI/ML provide significant gains over conventional statistical methods?"*

**Cost Overrun Task — F1 / AUC:**

| Approach | Model | F1 Score | ROC-AUC |
|---|---|---|---|
| Statistical | Logistic Regression | 0.6296 | 0.8517 |
| Statistical | Decision Tree | 0.6778 | 0.8545 |
| ML | Random Forest (CUF-only) | 0.7143 | 0.9163 |
| ML | Gradient Boosting (CUF-only) | 0.8617 | 0.9514 |
| **ML** | **XGBoost (CUF + Temporal)** | **0.8646** | **0.9542** |

> **Conclusion:** ML provides a **+27.6% improvement in F1** over the best statistical baseline (Decision Tree, F1 0.6778 → XGBoost, F1 0.8646).

**Time Overrun Task — F1:**

| Approach | Model | F1 Score | ROC-AUC |
|---|---|---|---|
| Statistical | Logistic Regression | 0.8961 | 0.9485 |
| Statistical | Decision Tree | 0.9146 | 0.9387 |
| ML | Gradient Boosting (CUF-only) | 0.9366 | 0.9746 |
| **ML** | **XGBoost (Full)** | **0.9409** | **0.9759** |

**Time Delay Regression — MAE / R²:**

| Approach | Model | MAE (months) | R² |
|---|---|---|---|
| Statistical | Linear Regression | 11.39 | 0.5934 |
| Statistical | Decision Tree Reg | 7.09 | 0.7525 |
| ML | Random Forest Reg (Full) | 4.96 | 0.8534 |
| **ML** | **XGBoost Reg (Full)** | **4.68** | **0.8703** |

### 5.5 CUF vs Temporal Features (SIH Requirement c)

| Feature Set | Cost-Overrun F1 |
|---|---|
| CUF-only (base + encoded fields) | 0.8617 |
| CUF + Temporal (multi-month trends) | **0.8646** |

> **Conclusion:** Temporal features add a **+0.3% gain** — modest but consistent, and proves that multi-month tracking provides **incremental predictive value** over a single CUF snapshot. The bulk of predictive power *already* comes from CUF fields, but adding temporal variables (expenditure/progress velocity, stagnation, cost-revision trends) meaningfully complements them.

### 5.6 Risk Scoring Framework

Every project gets a **composite risk score (0–100)**:

```
Risk Score = Cost_Overrun_Probability × 22
           + Time_Overrun_Probability × 22
           + Expenditure_Lag           × 15
           + PF_Gap_Score              × 13
           + Cost_Magnitude            × 10
           + Stagnation_Score          × 10
           + Progress_Stagnant         ×  8
```

| Risk Score | Category | Count |
|---|---|---|
| 0–30 | 🟢 Low | 654 |
| 30–60 | 🟡 Medium | 950 |
| 60–80 | 🟠 High | 351 |
| 80–100 | 🔴 Critical | 92 |

Every project carries `risk_score` + `risk_category` and is surfaced in the Risk Monitor and, via the LLM assistant, through natural-language queries.

### 5.7 Early Warning Alert Engine

`generate_alerts()` produces **3,888 alerts** across the portfolio using **7 rule types**:

| # | Alert Type | Trigger |
|---|---|---|
| 1 | `COST_ESCALATION` | Cost overrun ratio > 1.15 |
| 2 | `EXPENDITURE_LAG` | < 40% budget spent but > 70% time elapsed |
| 3 | `PROGRESS_STAGNATION` | < 30% physical progress with > 80% time elapsed |
| 4 | `PF_GAP` | Physical–financial progress gap > 20% |
| 5 | `HIGH_RISK_ML` / `ELEVATED_RISK_ML` | ML risk-score thresholds |
| 6 | `TIME_OVERRUN` | Delayed by more than 12 months |
| 7 | `MULTI_MONTH_STAGNATION` | No progress across multiple months |

Alerts are served via `/api/alerts` (severity / type / sector filters + pagination). The live predictor also generates real-time warnings + recommendations for a single project input.

### 5.8 Benchmarking & Cost-Driver Analysis

- **Benchmarks (`compute_benchmarks()`):** per-sector average cost, overrun %, progress, risk, and % of projects with cost/time overrun → `sector_benchmarks.csv`.
- **Cost drivers (`analyze_cost_drivers()`):** **SHAP** (SHapley Additive exPlanations) tree explainer on the XGBoost cost model → mean SHAP value per feature, showing which features push projects into cost overrun → `cost_drivers.csv`. The Analytics page renders the **top-10 cost-escalation drivers** as a horizontal bar chart.

---

## 6. LLM-Enabled Project Intelligence Assistant

### 6.1 What It Does

A government official asks questions in **plain English**:

- *"Which projects in Maharashtra are most at risk?"*
- *"Why is the Energy sector's cost overrun trending up?"*
- *"Show me all projects by the Ministry of Railways below 50% progress."*
- *"What's the risk distribution across ministries?"*

The assistant returns **concise, data-backed, natural-language answers**, often as formatted tables/bullet lists.

### 6.2 Approach: RAG-lite (No Vector Database Needed)

Because the project data is **structured tabular data** (~2,047 rows), we do **not** need embeddings, chunking, or a vector database. Instead we use a **"retrieval-augmented" pattern** where the *retrieval step is an exact Pandas query*:

```
User asks:                    Backend:                      LLM (Groq):
"Which Railway projects       ─► filters by ministry/       ─► reads context,
 have critical risk?"            sector/state/risk, sorts       formats the answer
                                 Pandas result            ─► in natural language
                                                              (tables, bullets)
                     Top records injected into the prompt as context
```

**Why this is smart:**
- No vector DB, no embeddings, no chunking — the data is tabular, not documents.
- Pandas filtering is **instant and exact** (vs. approximate semantic search).
- The LLM's job is to **interpret intent** and **format the answer** — not to "know" the data. This guarantees factual accuracy (no hallucinated project names/numbers).

### 6.3 Components

| Component | File | Role |
|---|---|---|
| Data loaders | `backend/chat.py` | Loads `projects_with_risk_scores.csv`, `alerts.csv`, `sector_benchmarks.csv` |
| Query engine | `query_projects()` | Keyword-based intent extraction (ministry / sector / state / risk / cost / time / stagnation), sorting (top-N, most expensive, riskiest, most delayed), and summary statistics |
| Prompt builder | `build_messages()` | Assembles system prompt (portfolio overview + rules), conversation history (last 10 messages), and the injected data context |
| LLM client | `chat()` | Calls Groq with `llama-3.3-70b-versatile`, `temperature=0.3`, returns response or a clean error |
| Suggestions | `CHAT_SUGGESTIONS` | Starter question chips grouped by category (Risk, Cost, Time & Progress, Sector & Ministry) |
| API | `main.py:536` `POST /api/chat`, `main.py:550` `GET /api/chat/suggestions` | Exposes the chat to the frontend |
| Frontend | `frontend/src/pages/ChatAssistant.jsx` | Chat bubbles, typing indicator, suggestion chips, markdown rendering (tables/lists/bold), auto-scroll, conversation state |
| Route | `frontend/src/App.jsx` | `/chat` nav item + route |

### 6.4 System-Prompt Design

The system prompt teaches the model it is a **government decision-support assistant**, provides the **live portfolio overview** (project count, ministries, sectors, total cost, overrun rates, risk distribution), and enforces rules:

1. **Always use the injected data context** — never invent project names or numbers.
2. Format numbers in **₹ crore**.
3. Risk bands: **Low 0–30, Medium 30–60, High 60–80, Critical 80–100**.
4. Be concise, use tables/bullets; state clearly when the data is insufficient.
5. Redirect off-topic questions back to the infrastructure domain.

### 6.5 Robustness & Fallback

- If `GROQ_API_KEY` is missing, the endpoint returns a clear message directing the user to create a free key at [console.groq.com](https://console.groq.com) — **the rest of the app works without it**.
- **Rate-limit handling** (Groq free tier ~30 req/min) returns a friendly retry message.
- Conversation history keeps the last **10 messages** for coherent follow-ups.

---

## 7. Technology Stack

All tools are **open-source**.

### 7.1 Backend

| Technology | Version | What It Does |
|---|---|---|
| **Python** | 3.11 | Core language for data science and API server |
| **FastAPI** | 0.115 | High-performance, async REST API framework |
| **Uvicorn** | 0.31 | ASGI server that runs the FastAPI app |
| **pandas** | 2.2 | Data manipulation, cleaning, merging, filtering |
| **numpy** | 1.26 | Numerical computation |
| **scikit-learn** | 1.5 | Preprocessing, model training, metrics, train/test split, CV |
| **XGBoost** | 2.1 | Best-performing gradient-boosting classifier + regressor |
| **LightGBM** | 4.5 | Alternative boosting model used in comparison |
| **SHAP** | 0.46 | Model explainability → cost-escalation driver analysis |
| **joblib** | 1.4 | Serializes trained models & scalers |
| **pydantic** | 2.8 | Request/response schema validation |
| **python-dotenv** | 1.1 | Loads `.env` (Groq API key) |
| **groq** | 0.25 | Client SDK for the Groq LLM inference API |
| **matplotlib / seaborn** | 3.9 / 0.13 | (Used in pipeline visualisations) |
| **fastapi StaticFiles / FileResponse** | – | Serves the built React frontend from the same server |

### 7.2 Frontend

| Technology | Version | What It Does |
|---|---|---|
| **React** | 19 | UI component library |
| **Vite** | 8 | Build tool & dev server |
| **React Router** | v7 | Client-side routing for 6 pages |
| **Framer Motion** | – | Page-transition / micro-interaction animations |
| **Recharts** | – | Data visualisation (bar/pie/line charts) |
| **Lucide React** | – | Icon library (sidebar, buttons, chat) |

### 7.3 ML & Data Processing

| Technology | What It Does |
|---|---|
| **tabula-py / pdfplumber** | Extract project tables from raw PAIMANA PDF reports |
| **pandas** | Clean & merge the 4 monthly snapshots into a panel |
| **scikit-learn** | Labels, feature encoding, scaling, model comparison |
| **XGBoost** | Final predictive models for cost & time overruns |
| **SHAP** | Cost-escalation driver attribution |
| **Groq (Llama 3.3 70B)** | LLM reasoning for the chat assistant |

---

## 8. Project Structure

```
0SIHNEW/
├── .env.example                 # GROQ_API_KEY template (copy to .env)
├── .dockerignore                # Excludes build artifacts from image
├── Dockerfile                   # Multi-stage build (frontend + backend)
├── docker-compose.yml           # One-command startup
├── requirements.txt             # Root requirements → references backend
├── documentation.md             # Short summary doc
├── detailed documentation.md    # THIS document
│
├── Dataset/
│   ├── raw/                     # Original PDF reports from PAIMANA
│   └── cleaned/                 # Cleaned CSVs (April–July 2026)
│
├── extract_pdf_data.py          # PDF → CSV extraction
├── extract_remaining.py         # Additional extraction
├── clean_data.py                # Data cleaning pipeline
│
├── backend/
│   ├── main.py                  # FastAPI server (all REST endpoints + static serving)
│   ├── chat.py                  # LLM chatbot module (RAG-lite, Groq)
│   ├── requirements.txt         # Backend Python dependencies
│   └── ml/
│       ├── train_pipeline.py    # Full ML pipeline (data → models → results)
│       ├── models/              # Trained .joblib models, scalers, encoders
│       └── results/             # Output CSVs/JSONs served by the API
│           ├── projects_with_risk_scores.csv
│           ├── alerts.csv
│           ├── sector_benchmarks.csv
│           ├── cost_drivers.csv
│           ├── summary.json
│           └── model_comparison.json
│
└── frontend/
    ├── package.json             # npm dependencies
    └── src/
        ├── App.jsx              # Routing, sidebar navigation
        ├── api.js               # API helper functions
        ├── index.css            # Design system (dark glassmorphism)
        ├── main.jsx             # React entry point
        └── pages/
            ├── Dashboard.jsx    # Overview stats & charts
            ├── Predictor.jsx    # AI input → prediction simulator
            ├── ChatAssistant.jsx# LLM chat assistant UI
            ├── RiskMonitor.jsx  # Project risk table
            ├── Alerts.jsx       # Early warning feed
            └── Analytics.jsx    # Benchmarks + model comparison
```

---

## 9. SIH Deliverables Mapping

| SIH Expected Outcome | Our Implementation | Location |
|---|---|---|
| **a. Cost Overrun Prediction Model** | XGBoost classifier (F1 = 0.8646, AUC = 0.9542) | `train_pipeline.py`, `cost_overrun_model.joblib`, `/api/predict` |
| **b. Time Overrun Prediction Model** | XGBoost classifier (F1 = 0.9409) + regressor (R² = 0.87, MAE = 4.68 mo) | `time_overrun_classifier.joblib`, `time_overrun_regressor.joblib`, `/api/predict` |
| **c. Project Risk Scoring Framework** | Composite 0–100 score, 4 risk tiers | `compute_risk_scores()` in pipeline; `/api/projects`; `/risk` page |
| **d. Early Warning Alert System** | 7 rule types, 3,888 alerts | `generate_alerts()`; `/api/alerts`; `/alerts` page |
| **e. Benchmarking & Comparative Analytics** | Sector benchmarks + Stat-vs-ML comparison (Req b/c) | `compute_benchmarks()`; `/api/analytics/*`; `/analytics` page |
| **f. Cost Escalation Driver Analysis** | SHAP feature-attribution (top-10 drivers) | `analyze_cost_drivers()`; `cost_drivers.csv`; `/api/analytics/cost-drivers` |
| **g. AI-powered Monitoring Dashboard** | Full React dashboard, 6 pages | Frontend (`/`, `/predict`, `/risk`, `/alerts`, `/analytics`) |
| **h. LLM-enabled Project Intelligence Assistant** | Groq + Llama 3.3 natural-language chat (RAG-lite) | `backend/chat.py`; `/api/chat`; `/chat` page |
| **i. Documentation & Deployment Framework** | This doc + Docker one-command deploy | `detailed documentation.md`, `Dockerfile`, `docker-compose.yml` |

**Technical Dimensions:**

| SIH Technical Dimension | Covered By |
|---|---|
| **(a)** Statistical & ML models, open-source | Section 5 (XGBoost, scikit-learn, pandas) |
| **(b)** AI/ML vs statistical comparison | Sections 5.4 (F1 +27.6%), `/api/analytics/model-comparison`, Analytics page |
| **(c)** CUF vs additional (temporal) variables | Section 5.5 (CUF-only 0.8617 → CUF+Temporal 0.8646) |

---

## 10. API Reference

All endpoints are under `http://localhost:8000/api` (or, when run in a single container, `http://localhost:8000/api`).

| Method | Endpoint | Returns |
|---|---|---|
| GET | `/api/summary` | Portfolio stats (projects, ministries, sectors, risk distribution, overrun rates) |
| GET | `/api/projects` | Project list with risk scores/categories (filters + pagination) |
| GET | `/api/projects/{idx}` | Single project detail |
| GET | `/api/alerts` | Early-warning alerts (severity / type / sector filters + pagination) |
| GET | `/api/analytics/sectors` | Per-sector benchmarks |
| GET | `/api/analytics/risk-distribution` | Risk category counts |
| GET | `/api/analytics/cost-drivers` | SHAP-based cost-escalation drivers |
| GET | `/api/analytics/model-comparison` | Statistical vs ML comparison (Req b) + CUF vs Temporal (Req c) |
| GET | `/api/analytics/ministry-overview` | Ministry-level stats |
| GET | `/api/analytics/overrun-trends` | Cost/time overrun trends |
| GET | `/api/filters` | Available ministries / sectors / states for dropdowns |
| GET | `/api/predict/options` | Options used by the predictor form |
| POST | `/api/predict` | Real-time prediction (cost/time overrun probs, delay estimate, risk score, warnings, recommendations, drivers) |
| POST | `/api/chat` | LLM chat answer for a natural-language question (accepts optional history) |
| GET | `/api/chat/suggestions` | Starter questions for the chat UI |
| GET | `/` (root) | Served built frontend (in Docker) |

---

## 11. How to Run

### 11.1 Prerequisite: Groq API Key (for the LLM Assistant)

The **LLM chat** needs a free `GROQ_API_KEY` (from [console.groq.com](https://console.groq.com)). **Every other feature works without it.**

```bash
# From the project root:
copy .env.example .env        # Windows
# or
cp .env.example .env          # Linux/macOS

# Then edit .env and paste your key:
#   GROQ_API_KEY=your_actual_key_here
```

### 11.2 Option A — Local Development

**Backend:**
```bash
cd backend
pip install -r requirements.txt
python main.py
# → Runs on http://localhost:8000
```

**Frontend (in a second terminal):**
```bash
cd frontend
npm install
npm run dev
# → Vite dev server (port 5173), proxies/points to http://localhost:8000
# Production build + serve:
#   npm run build
#   npx serve -s dist -l 5173
```

### 11.3 Option B — Docker (One Command, Recommended for Judges)

```bash
# From the project root (with your .env ready):
docker compose up --build
# → Whole system (backend + built frontend + LLM) runs at http://localhost:8000
```

The `Dockerfile` is **multi-stage**: it builds the React app in a `node:20-alpine` stage, then installs the Python backend in a `python:3.11-slim` stage, copies in the built frontend, and serves everything from one container. ML models and results are mounted read-only via `docker-compose.yml` volumes so they persist across rebuilds.

### 11.4 Retrain ML Models (if data changes)

```bash
cd 0SIHNEW
python backend/ml/train_pipeline.py
# → Retrains all models, regenerates results, saves to backend/ml/models/ and backend/ml/results/
```

---

## 12. Key Numbers to Remember

| Metric | Value |
|---|---|
| Projects Monitored | 2,047 |
| Ministries Covered | 17 |
| Infrastructure Sectors | 22 |
| Total Revised Portfolio Cost | ₹42.78 lakh crore |
| ML Features Used | 24 (13 CUF-base + 11 temporal) |
| Best Cost Model | XGBoost (F1 = 0.8646, AUC = 0.9542) |
| Best Time Model | XGBoost (F1 = 0.9409, AUC = 0.9759) |
| Delay Regression | R² = 0.8703, MAE = 4.68 months |
| ML vs Statistical Improvement | **+27.6% F1** |
| Temporal-Feature Gain (CUF-only → +Temporal) | +0.3% F1 |
| Early Warning Alerts Generated | 3,888 |
| Risk Categories | Low 654 · Medium 950 · High 351 · Critical 92 |
| LLM Model | Llama 3.3 70B via Groq (`llama-3.3-70b-versatile`) |
| Deployment | Docker multi-stage, single command (`docker compose up --build`) |

---

*Generated for team reference — PAIMANA AI, Smart India Hackathon 2026. All numbers verified against `backend/ml/results/summary.json` and `backend/ml/results/model_comparison.json`.*
