# Sanketak (संकेतक)

**Turning safety reports into explainable early warnings.**

An AI/NLP engine that detects Serious Injury & Fatality (SIF) precursors in unsafe-act, unsafe-condition and near-miss reports for Oil India Limited (OIL). It scores each report for SIF risk, extracts the failed safety barrier and the relevant life-saving rule, finds repeat patterns across reports, and matches them to historical disasters.

Built by **Team Rakshak** for **Smart India Hackathon 2026** (Problem Statement ID: 26165, theme: Smart Automation).

---

## Table of Contents

- [Overview](#overview)
- [Screenshots](#screenshots)
- [Key Features](#key-features)
- [How It Works](#how-it-works)
- [Tech Stack](#tech-stack)
- [Architecture and Data Privacy](#architecture-and-data-privacy)
- [Project Structure](#project-structure)
- [ML Pipeline](#ml-pipeline)
- [Getting Started](#getting-started)
- [Configuration](#configuration)
- [Planned Work](#planned-work)
- [References and Datasets](#references-and-datasets)
- [Contributing](#contributing)
- [Acknowledgements](#acknowledgements)

---

## Overview

OIL receives large volumes of safety reports that contain early warning signs of serious injuries and fatalities. Manual, periodic review makes it hard to spot and prioritise the critical ones. Repeated barrier failures across sites and equipment stay disconnected. Conventional classification tells you *what happened*, but not *why the situation could become dangerous*.

Sanketak addresses this by being:

| | |
|---|---|
| **Explainable** | Shows the evidence behind every SIF flag. |
| **Barrier-aware** | Identifies the failed or missing safety control. |
| **Pattern-aware** | Detects repeated failures across reports, sites and equipment. |
| **Precedent-aware** | Connects patterns with historical incidents. |
| **Worker-first** | Anonymous reporting with no personal identity fields, and report-status tracking with a token. |

The AI supports HSE reviewers and does not replace them: **evidence first, HSE decides.**

## Screenshots

### HSE Command Centre Dashboard

![HSE Dashboard](https://drive.google.com/uc?export=view&id=1l2pPbyNTnv639j5q1nYQ7t_9PE7ozxhh)

Reports received, reports needing review, high-SIF-potential reports, open corrective actions and critical reports, with a priority table showing site, SIF potential, barrier failure, life-saving rule, received time and status.

### Report Intelligence

![Report Intelligence](https://drive.google.com/uc?export=view&id=15bBkW4__lXbOJ_dieLYtwNeidYlCBlZ2)

A single-report view with tracking ID, priority, SIF potential, model confidence, the original worker report and the relevant life-saving rule.

## Key Features

| Feature | Description |
|---|---|
| **Anonymous reporting** | Reports are stored with an anonymous token. No login and no PII for workers. |
| **Report status tracking** | Workers can check the status of a report using only their token. |
| **SIF risk scoring** | A locally run ML classifier (TF-IDF + Logistic Regression) outputs a risk probability and level. |
| **NLP fingerprint extraction** | An LLM extracts activity, hazard and barrier failures from the report text. |
| **Barrier and life-saving rule mapping** | Each report is mapped to a failed safety barrier and a relevant life-saving rule (taxonomy labels in `dashboard/taxonomy-labels.js`). |
| **Pattern detection** | Finds recurring hazards and failures across reports. |
| **Precedent matching** | pgvector similarity search links reports to historical disasters. |
| **HSE dashboard** | Dashboard, Risk Radar, Patterns, Corrective Actions, Report view and Settings pages. |
| **Corrective actions** | Assign actions and track whether HSE has verified them. |
| **PDF report generation** | Printable incident reports from the dashboard. |
| **Secure HSE login** | JWT-authenticated accounts with hashed passwords. |

## How It Works

```
Worker submits a report (text, any language)
        |
        v
Stored in PostgreSQL with an anonymous token (no PII)
        |
        +--> SIF classifier (local, scikit-learn) --> risk probability + level
        |
        +--> NLP fingerprint extraction (LLM API) --> activity / hazard / barrier failures
        |
        +--> Pattern detection + precedent matching (pgvector similarity)
        |
        v
HSE staff log in (JWT) --> triage, review evidence, assign corrective actions
Worker checks status anytime with the anonymous token
```

## Tech Stack

| Layer | Technologies |
|---|---|
| **Backend** | Python, FastAPI, Uvicorn, Pydantic, SQLAlchemy, Alembic |
| **Database** | PostgreSQL 16 with pgvector; Supabase connection for the dashboard |
| **ML / NLP** | scikit-learn (TF-IDF + Logistic Regression), sentence-transformers, pandas, numpy, joblib |
| **LLM** | OpenAI API (the Groq and Google Generative AI client libraries are also listed in the dependencies) |
| **Auth** | JWT (python-jose, passlib, bcrypt) |
| **HSE dashboard** | HTML, CSS, vanilla JavaScript |
| **Deployment** | Docker, Docker Compose |

## Architecture and Data Privacy

| Component | Runs on | External calls |
|---|---|---|
| FastAPI backend | Organisation's server (Docker) | None |
| PostgreSQL + pgvector | Organisation's server (Docker) | None |
| JWT authentication | Organisation's server, self-signed tokens | None |
| SIF classifier | Local, via scikit-learn | None |
| NLP fingerprint extraction | Server calls an LLM provider | **Yes: needs internet and an OpenAI API key** |
| Dashboard connection | Supabase | **Yes** |

The core scoring model, database and API run on infrastructure the organisation controls. The two external dependencies above are disclosed exceptions. See [ARCHITECTURE.md](ARCHITECTURE.md) for details.

## Project Structure

```
Sanketak/
├── app/                      # FastAPI backend
│   ├── main.py               # Application entry point
│   ├── core/                 # Configuration and core utilities
│   ├── models/               # SQLAlchemy models
│   ├── schemas/              # Pydantic schemas
│   ├── routers/              # API routes
│   ├── services/             # Business logic (analysis, scoring)
│   └── ml_models/            # Model loading
├── alembic/                  # Database migrations
├── dashboard/                # HSE web dashboard
│   ├── index.html, login.html, dash.html
│   ├── risk-radar.html, patterns.html, corrective-actions.html
│   ├── report.html, settings.html, pdf-generator.html
│   ├── hse-app.js, hse-services.js, login.js, hse.css, login.css
│   ├── taxonomy-labels.js    # Barrier / life-saving rule labels
│   └── supabase-config.js    # Supabase configuration
├── nlp/                      # NLP fingerprint extraction
├── intelligence/             # Pattern detection and precedent matching
├── src/                      # Dataset and model-training pipeline
│   ├── create_master_dataset.py
│   ├── clean_master_dataset.py
│   ├── generate_sif_labels.py
│   ├── create_training_dataset.py
│   ├── train_sif_classifier.py
│   ├── test_sif_model.py
│   ├── inspect_datasets.py
│   └── api.py
├── output/                   # Datasets and trained model
│   ├── sanketak_master_40k.csv
│   ├── sanketak_master_clean.csv
│   ├── sanketak_master_labeled.csv
│   ├── sanketak_training.csv
│   └── sif_classifier.joblib
├── seed_demo_data.py         # Seeds demo reports
├── seed_precedents.py        # Seeds historical disaster precedents
├── seed_reports_from_predictions.py
├── check_model.py            # Sanity check for the SIF model
├── Dockerfile
├── docker-compose.yml
├── entrypoint.sh             # Waits for DB, runs migrations, starts the API
├── alembic.ini
├── requirements.txt
└── ARCHITECTURE.md
```

## ML Pipeline

The SIF classifier is trained from a labelled master dataset built by the scripts in `src/`:

| Script | Purpose |
|---|---|
| `create_master_dataset.py` | Combines source datasets into one master dataset (`sanketak_master_40k.csv`). |
| `clean_master_dataset.py` | Cleans the master dataset (`sanketak_master_clean.csv`). |
| `generate_sif_labels.py` | Generates SIF labels (`sanketak_master_labeled.csv`). |
| `create_training_dataset.py` | Builds the training set (`sanketak_training.csv`). |
| `train_sif_classifier.py` | Trains the classifier and saves `output/sif_classifier.joblib`. |
| `test_sif_model.py` | Tests the trained model. |
| `inspect_datasets.py` | Helper for inspecting the datasets. |

The trained model (`output/sif_classifier.joblib`) is included, so retraining is optional. To retrain:

```bash
python src/train_sif_classifier.py
```

## Getting Started

### Prerequisites

- [Docker](https://docs.docker.com/get-docker/) and Docker Compose
- An OpenAI API key (for NLP fingerprint extraction)
- A Supabase project (for the dashboard connection)
- Git

### 1. Clone the repository

```bash
git clone https://github.com/Codejourn/Sanketak.git
cd Sanketak
```

### 2. Create the environment file

Create a `.env` file in the project root (see [Configuration](#configuration)):

```env
OPENAI_API_KEY=your_openai_api_key_here
```

### 3. Start the backend

```bash
docker-compose up --build
```

This starts PostgreSQL + pgvector on port `5432` and the FastAPI backend on port `8000`. On startup, `entrypoint.sh` waits for the database, runs `alembic upgrade head`, and starts the API.

### 4. Open the API docs

```
http://localhost:8000/docs
```

### 5. (Optional) Seed demo data

```bash
docker-compose exec api python seed_precedents.py
docker-compose exec api python seed_demo_data.py
```

### 6. Open the HSE dashboard

Add your Supabase URL and public key to `dashboard/supabase-config.js`, then serve the dashboard folder:

```bash
cd dashboard
python -m http.server 5500
```

Open `http://localhost:5500/login.html`.

### Running without Docker

```bash
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt
alembic upgrade head
uvicorn app.main:app --reload --port 8000
```

This needs a PostgreSQL instance with the `pgvector` extension and a `DATABASE_URL` in `.env`.

## Configuration

| Variable / file | Purpose |
|---|---|
| `DATABASE_URL` | PostgreSQL connection string. Docker Compose sets it to `postgresql://sanketak:sanketak@db:5432/sanketak`. |
| `OPENAI_API_KEY` | Needed for NLP fingerprint extraction. |
| `dashboard/supabase-config.js` | Supabase URL and public (anon) key. |

> **Security note:** the default database credentials in `docker-compose.yml` are for local development only. Change them and set a strong JWT secret before any real deployment. Never commit `.env`, API keys or a Supabase service-role key.

## Planned Work

From `ARCHITECTURE.md` and the SIH proposal. These are **not implemented yet**:

- [ ] `/rules` CRUD endpoint so HSE admins can edit detection categories (barrier categories, equipment tags) without redeploying code.
- [ ] Replace the external LLM with a locally hosted model for a fully on-premise deployment.
- [ ] Worker-facing mobile app with voice and photo reporting.
- [ ] Multimodal analysis combining text, voice and images.
- [ ] Emerging-risk intelligence across hazards, equipment and barriers.
- [ ] Integration of validated IoT, sensor and operational data.
- [ ] Continuous model improvement from HSE-validated feedback.

## References and Datasets

**Datasets**
- [OSHA construction accident cases (4,470)](https://github.com/safetyhub/OSHA_Acc/blob/master/OSHA%20construction%204470%20cases%20original.xlsx)
- [OSHA Severe Injury Reports](https://www.osha.gov/severe-injury-reports)
- [MSHA Accident/Injury Narratives](https://arlweb.msha.gov/fatals/fabmc-1995-2007.htm)

**Research and standards**
- [Automated text classification of near-misses from safety reports: an improved deep learning approach (ScienceDirect)](https://www.sciencedirect.com/science/article/abs/pii/S147403462030029X)
- [Predictive Risk Modeling via Natural Language Processing of Industrial Safety Reports (ResearchGate)](https://www.researchgate.net/publication/401310321_Predictive_Risk_Modeling_via_Natural_Language_Processing_of_Industrial_Safety_Reports)
- [Campbell Institute: classifying a near miss by its potential for serious injury or fatality (Wolters Kluwer)](https://www.wolterskluwer.com/en/expert-insights/classify-a-near-miss-on-its-potential-for-serious-injury-or-fatality)
- [IOGP Life-Saving Rules](https://www.iogp.org/workstreams/safety/safety/life-saving-rules/)

**Demo video:** https://youtu.be/io1ZMRXvZbA

## Contributing

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/your-feature`
3. Commit your changes: `git commit -m "Add your feature"`
4. Push the branch: `git push origin feature/your-feature`
5. Open a Pull Request

## Acknowledgements

- **Team Rakshak**, Smart India Hackathon 2026, built for **Oil India Limited (OIL)**.
- Forked from [16701012024rash/Sanketak-](https://github.com/16701012024rash/Sanketak-). Thanks to all team members and contributors.

---

*Sanketak (संकेतक) means "indicator" or "signal". Every report is a signal worth listening to.*
