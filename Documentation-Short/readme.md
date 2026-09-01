# PAIMANA AI — Project Documentation

**AI-Powered Predictive Analytics & Early Warning System for Infrastructure Monitoring**

**Team Project | Smart India Hackathon 2026**

---

## 1. What Problem Are We Solving?

India's Ministry of Statistics & Programme Implementation (MoSPI) tracks **~2,000 infrastructure projects** worth ₹42+ lakh crore through the **PAIMANA portal**. Despite having rich monthly data, the current system only provides *descriptive reporting* — it tells you *what happened*, not *what will happen*.

The SIH problem statement asked us to transform this from a **reporting system** into a **predictive and prescriptive decision-support system** that can:

- **Predict** which projects will face cost overruns and schedule delays *before* they happen
- **Score** every project's risk level on a 0–100 scale
- **Generate early warning alerts** for projects showing red flags
- **Compare** whether AI/ML actually outperforms traditional statistical methods
- **Assess** which data fields (CUF vs temporal) drive prediction accuracy

---

## 2. What Does Our AI Predict?

When a government official enters project data on our dashboard, the AI model generates **5 real-time predictions**:

| # | Prediction | What It Means | Example Output |
|---|---|---|---|
| 1 | **Cost Overrun Probability** | % chance this project will exceed its revised budget | "78.5% — Overrun Predicted" |
| 2 | **Time Overrun Probability** | % chance this project will miss its deadline | "99.0% — Schedule Delay Expected" |
| 3 | **Predicted Delay (months)** | If delayed, how many extra months it will take | "+20.6 months delay" |
| 4 | **Risk Score (0–100)** | Overall health score combining all risk factors | "62.1 — High Risk" |
| 5 | **Early Warning Alerts** | Specific red flags detected in the project's numbers | "Large PF Gap of 52.5%", "Cost revised up 60%" |

### How It Works (User Flow)

A government official enters the **9 CUF fields** from the PAIMANA portal:
> Ministry, Sector, State, Original Cost, Revised Cost, Expenditure, Physical Progress, Planned Duration, Project Age

↓ Our trained **XGBoost model** processes these through 24 engineered features ↓

And instantly tells them:
- **"Will this project go over budget?"** → Yes/No + probability percentage
- **"Will this project be delayed?"** → Yes/No + probability + estimated months of delay
- **"How risky is this project overall?"** → 0–100 score with Low/Medium/High/Critical label
- **"What specific problems should I watch out for?"** → Targeted alerts + actionable recommendations

This is exactly what SIH asked for — moving from *"here's what happened"* (descriptive monitoring) to *"here's what will happen and what you should do about it"* (predictive + prescriptive monitoring).

---

## 3. Solution Architecture

Our system follows a **three-layer architecture**:

```
┌─────────────────────────────────────────────────────┐
│                   FRONTEND (React)                  │
│    Dashboard │ Predictor │ Risk Monitor │ Alerts    │
│              │ Analytics                            │
│         Vite + React 19 + Framer Motion             │
└────────────────────┬────────────────────────────────┘
                     │ REST API (HTTP)
┌────────────────────┴────────────────────────────────┐
│                 BACKEND (FastAPI)                    │
│    /api/summary │ /api/projects │ /api/predict      │
│    /api/alerts  │ /api/analytics│ /api/filters      │
│              Python + Uvicorn                       │
└────────────────────┬────────────────────────────────┘
                     │ joblib.load()
┌────────────────────┴────────────────────────────────┐
│              ML PIPELINE (XGBoost)                  │
│   Data Cleaning → Feature Engineering → Training    │
│   → Model Selection → Risk Scoring → Alert Gen     │
│        scikit-learn + XGBoost + pandas              │
└─────────────────────────────────────────────────────┘
```

### Data Flow

1. **Raw PDFs** → Extracted via `extract_pdf_data.py` into CSVs (4 months: April–July 2026)
2. **Cleaning** → `clean_data.py` standardizes columns, fixes encoding, handles missing values
3. **ML Pipeline** → `train_pipeline.py` matches projects across months, engineers features, trains models
4. **API Server** → `main.py` loads trained models and serves predictions via REST endpoints
5. **Frontend** → React dashboard fetches data from API and renders interactive visualizations

---

## 4. ML Pipeline — How We Built the AI

### 4.1 Data Source
- **4 monthly snapshots** from PAIMANA (April, May, June, July 2026)
- **2,047 projects** across **17 Ministries** and **22 sectors**
- Each project has CUF fields: cost, expenditure, progress, timelines, location, agency

### 4.2 Feature Engineering (24 Features Total)

We engineered features in two categories:

| Category | Features | Count |
|---|---|---|
| **CUF Base Features** | original_cost, revised_cost, expenditure, physical_progress, expenditure_ratio, financial_progress, PF_gap, planned_duration, project_age, time_elapsed_ratio | 10 |
| **Temporal Features** (multi-month trends) | expenditure_velocity, expenditure_acceleration, progress_velocity, progress_stagnant, cost_revision_trend, cost_overrun_trend, months_tracked, expenditure_ratio_trend | 11 |
| **Categorical (encoded)** | ministry, sector, state | 3 |

### 4.3 Training Approach

We trained models for **three prediction tasks**:

| Task | Type | Best Model | Key Metric |
|---|---|---|---|
| Cost Overrun Prediction | Binary Classification | XGBoost | F1 = 0.865, AUC = 0.954 |
| Time Overrun Prediction | Binary Classification | XGBoost | F1 (best in class) |
| Time Delay Estimation | Regression | XGBoost | R² score |

**Train/Test Split**: 80/20 with stratified sampling, random_state=42

### 4.4 Statistical vs ML Comparison (SIH Requirement b)

This directly answers the SIH question: *"Does AI/ML provide significant gains over conventional statistical methods?"*

| Approach | Model | F1 Score | ROC-AUC |
|---|---|---|---|
| Statistical | Logistic Regression | 0.630 | 0.852 |
| Statistical | Decision Tree | 0.678 | 0.855 |
| ML | Random Forest (CUF-only) | 0.714 | 0.916 |
| ML | Gradient Boosting (CUF-only) | 0.862 | 0.951 |
| **ML** | **XGBoost (CUF+Temporal)** | **0.865** | **0.954** |

**Conclusion**: ML provides a **27.6% improvement** in F1 score over the best statistical baseline.

### 4.5 CUF vs Temporal Features (SIH Requirement c)

| Feature Set | F1 Score |
|---|---|
| CUF-only (base fields) | 0.862 |
| CUF + Temporal (multi-month trends) | 0.865 |

**Conclusion**: Temporal features add a **+0.3% gain** — modest but proves that multi-month tracking provides incremental value for prediction.

### 4.6 Risk Scoring Framework

Every project gets a **composite risk score (0–100)** computed as:

```
Risk Score = Cost_Overrun_Probability × 22
           + Time_Overrun_Probability × 22
           + Expenditure_Lag          × 15
           + PF_Gap_Score             × 13
           + Cost_Magnitude           × 10
           + Stagnation_Score         × 10
           + Progress_Stagnant        ×  8
```

| Risk Score | Category | Count |
|---|---|---|
| 0–30 | 🟢 Low | 654 |
| 30–60 | 🟡 Medium | 950 |
| 60–80 | 🟠 High | 351 |
| 80–100 | 🔴 Critical | 92 |

---

## 5. Technology Stack

### Backend
| Technology | Purpose |
|---|---|
| **Python 3.11** | Core language |
| **FastAPI** | REST API framework |
| **Uvicorn** | ASGI server |
| **pandas / numpy** | Data manipulation |
| **scikit-learn** | ML pipeline, preprocessing, metrics |
| **XGBoost** | Best-performing gradient boosting model |
| **joblib** | Model serialization |

### Frontend
| Technology | Purpose |
|---|---|
| **React 19** | UI framework |
| **Vite 8** | Build tool & dev server |
| **React Router v7** | Client-side routing |
| **Framer Motion** | Page transition animations |
| **Recharts** | Data visualization (charts) |
| **Lucide React** | Icon library |

### Data Processing
| Technology | Purpose |
|---|---|
| **tabula-py / pdfplumber** | PDF data extraction |
| **pandas** | Data cleaning & merging |

---

## 6. Project Structure

```
0SIHNEW/
├── Dataset/
│   ├── raw/                    # Original PDF reports from PAIMANA
│   └── cleaned/                # Cleaned CSVs (April–July 2026)
│
├── backend/
│   ├── main.py                 # FastAPI server (all API endpoints)
│   └── ml/
│       ├── train_pipeline.py   # Full ML pipeline (932 lines)
│       ├── models/             # Trained model files (.joblib)
│       └── results/            # CSVs and JSONs generated by pipeline
│
├── frontend/
│   ├── src/
│   │   ├── App.jsx             # Root component, routing, sidebar
│   │   ├── api.js              # API helper functions
│   │   ├── index.css           # Design system (dark theme, glassmorphism)
│   │   └── pages/
│   │       ├── Dashboard.jsx   # Overview stats & charts
│   │       ├── Predictor.jsx   # AI Input→Prediction simulator
│   │       ├── RiskMonitor.jsx # Project-level risk table
│   │       ├── Alerts.jsx      # Early warning alert feed
│   │       └── Analytics.jsx   # Sector benchmarks, model comparison
│   └── dist/                   # Production build output
│
├── extract_pdf_data.py         # PDF → CSV extraction script
├── extract_remaining.py        # Additional extraction
└── clean_data.py               # Data cleaning pipeline
```

---

## 7. SIH Deliverables Mapping

Here is how our solution maps to every expected outcome listed in the SIH problem statement:

| SIH Expected Outcome | Our Implementation | Location |
|---|---|---|
| a. Cost Overrun Prediction Model | XGBoost classifier (F1=0.865) | `train_pipeline.py`, `cost_overrun_model.joblib` |
| b. Time Overrun Prediction Model | XGBoost classifier + regressor | `time_overrun_classifier.joblib`, `time_overrun_regressor.joblib` |
| c. Project Risk Scoring Framework | Composite 0–100 score with 4 risk categories | `compute_risk_scores()` in pipeline |
| d. Early Warning Alert System | Rule-based alerts (Cost, Time, PF Gap, Stagnation, Expenditure Lag) | `generate_alerts()` in pipeline, `/api/alerts` |
| e. Benchmarking & Comparative Analytics | Sector benchmarks, Statistical vs ML comparison | Analytics page, `model_comparison.json` |
| f. Cost Escalation Driver Analysis | Feature importance + cost driver breakdown by sector | `cost_drivers.csv`, Analytics page |
| g. AI-powered Monitoring Dashboard | Full React dashboard with 5 pages | Frontend (`/`, `/predict`, `/risk`, `/alerts`, `/analytics`) |
| h. LLM-enabled Project Intelligence | AI Predictor (input CUF → real-time prediction) | `/predict` page + `/api/predict` endpoint |
| i. Documentation & Deployment | This document + production build | `documentation.md`, `dist/` |

---

## 8. How to Run

### Backend
```bash
cd 0SIHNEW
pip install fastapi uvicorn pandas numpy scikit-learn xgboost joblib
python backend/main.py
# → Runs on http://localhost:8000
```

### Frontend
```bash
cd 0SIHNEW/frontend
npm install
npm run build
npx serve -s dist -l 5173
# → Runs on http://localhost:5173
```

### Retrain ML Models (if needed)
```bash
cd 0SIHNEW
python backend/ml/train_pipeline.py
# → Retrains all models, regenerates results, saves to backend/ml/models/
```

---

## 9. Key Numbers to Remember

| Metric | Value |
|---|---|
| Projects Monitored | 2,047 |
| Ministries Covered | 17 |
| Infrastructure Sectors | 22 |
| Total Portfolio Cost | ₹42.78 lakh crore |
| ML Features Used | 24 (13 CUF + 11 Temporal) |
| Best Model | XGBoost (F1=0.865, AUC=0.954) |
| ML vs Statistical Improvement | +27.6% F1 |
| Early Warning Alerts Generated | 3,888 |
| Risk Categories | Low (654), Medium (950), High (351), Critical (92) |

---

*Document generated for team reference — PAIMANA AI, SIH 2026*
