<p align="center">
  <img src="Figures/CredGuard.jpg" alt="CredGuard" width="700">
</p>

# CredGuard

CredGuard is a credential-based Intrusion Detection System (IDS) for Internet of Things (IoT) environments that couples supervised learning with Explainable Artificial Intelligence (XAI). Rather than inspecting packet payloads or network flows, CredGuard operates at the **authentication layer**: it models the behavioural signature of every login attempt — inter-arrival timing, attempt progression, failure ratio, credential structure, and network/geographic origin — and classifies it as `Normal` or `Attack` in real time.

CredGuard is an **end-to-end framework**, not a single model. It spans five stages: construction of a unified authentication dataset (**CredGuardV1**) from five heterogeneous sources, leakage-free behavioural feature engineering, a controlled benchmark of three Machine Learning (ML) and three Deep Learning (DL) architectures, an explainability layer built on SHAP and LIME, and a deployed Flask/PostgreSQL service that executes autonomous mitigation on a physical Raspberry Pi IoT node. Detection does not stop at the login boundary: a **post-login behavioural engine** continues to score the session against the user's own historical baseline, so a stolen credential that survives authentication is still caught in-session.

---

## Table of Contents

- [How CredGuard Works](#how-credguard-works)
- [How CredGuard Compares](#how-credguard-compares)
- [Features](#features)
- [Installation](#installation)
- [Usage](#usage)
  - [1. Build the CredGuardV1 Dataset](#1-build-the-credguardv1-dataset)
  - [2. Behavioural Feature Engineering](#2-behavioural-feature-engineering)
  - [3. Train the Machine Learning Models](#3-train-the-machine-learning-models)
  - [4. Train the Deep Learning Models](#4-train-the-deep-learning-models)
  - [5. Generate XAI Explanations (SHAP / LIME)](#5-generate-xai-explanations-shap--lime)
  - [6. Run the Detection Backend](#6-run-the-detection-backend)
  - [7. Deploy the IoT Node (Raspberry Pi)](#7-deploy-the-iot-node-raspberry-pi)
  - [8. Post-Login Behavioural Risk Scoring](#8-post-login-behavioural-risk-scoring)
- [Reproducibility](#reproducibility)
- [Data Availability](#data-availability)
- [Directory Structure](#directory-structure)
- [Citation](#citation)
- [Contributing](#contributing)
- [License](#license)

---

## How CredGuard Works

Conventional IoT intrusion detection systems are predominantly **flow-based**: they consume packet headers, traffic volumes, or protocol statistics and are trained on general-purpose network datasets such as BoT-IoT, TON\_IoT, or CICIDS. This design is effective against volumetric and protocol-level attacks but is largely blind to the credential layer, where brute-force, credential-stuffing, and account-takeover campaigns occur. A valid username and password pair transmitted over a well-formed session is, from the perspective of a flow-based detector, indistinguishable from legitimate traffic.

CredGuard closes this gap by treating the **authentication event itself** as the unit of analysis. Each login attempt is converted into a behavioural feature vector that captures *how* the credential was presented rather than *what* was transmitted, and the resulting vector is classified, explained, and acted upon within a single pipeline.

<p align="center">
  <img src="Figures/architecture_overview.svg" alt="CredGuard architecture and data flow" width="820">
</p>

The pipeline comprises four operational layers:

**1. Acquisition layer.** Authentication events are captured from the IoT node and the web application, normalised to a common schema, and enriched with network context. Autonomous System Number (ASN) and geographic origin are resolved from the source IP address through the IPinfo service.

**2. Feature layer.** The raw event is transformed into behavioural features using **expanding-window statistics only**, so that no feature at time *t* is computed using information from *t+1* or later. This constraint is what allows the offline benchmark to be a faithful proxy for online deployment; it is described in full in [REPRODUCIBILITY.md](REPRODUCIBILITY.md).

**3. Detection and explanation layer.** The feature vector is scored by the deployed XGBoost classifier. Every decision is accompanied by an explanation: SHAP supplies the global attribution structure of the model, while LIME produces a local, per-decision rationale that is surfaced directly in the analyst dashboard.

**4. Response layer.** The backend translates the classification into an enforcement action — grant, challenge, or block — and dispatches it to the Raspberry Pi node, which executes the decision on the device. Analytical processing remains entirely on the server, keeping the constrained IoT device responsible only for execution.

### Post-login behavioural monitoring

Authentication alone is an insufficient security boundary, since a compromised credential is by definition valid. CredGuard therefore continues to monitor the session after access is granted, comparing the observed interaction against a per-user behavioural baseline derived from that user's own history.

<p align="center">
  <img src="Figures/behaviour_engine.svg" alt="Post-login behavioural risk scoring" width="760">
</p>

Six indicators are evaluated — mean previous session duration, live-stream watching rate, camera usage rate, mean camera interaction count, full-screen usage rate, and mean full-screen usage — and combined into a composite **Risk Score** on a 0–100 scale, mapped to `Low`, `Medium`, and `High` risk bands. Sessions reaching the high band are terminated or blocked. Where a user has insufficient history to support a baseline, the engine falls back to a rule-based heuristic, so cold-start accounts are never left unscored.

### Why this matters

The combination yields a detector that is **accurate, fast, interpretable, and enforcing** at once. Detection latency remains low enough for real-time use on constrained hardware; each alert carries a human-readable justification rather than an opaque score; and the system acts on its own conclusions instead of merely reporting them.

---

## How CredGuard Compares

Existing work in this space tends to occupy one of three positions. **Signature-based systems** (Snort, Suricata) are precise but cannot generalise to unseen credential-attack variants. **ML- and DL-based IoT IDSs** generalise well but are trained on flow-level datasets and remain opaque at the decision level. **XAI-enhanced IDSs** restore interpretability but are typically evaluated offline, on general network traffic, without a deployed response path.

CredGuard is, to our knowledge, the only framework that combines all six capabilities below within a single deployed system.

| Capability | Signature IDS | Flow-based ML/DL IDS | XAI-enhanced IDS | **CredGuard** |
| --- | :---: | :---: | :---: | :---: |
| Credential-layer modelling | ✗ | ✗ | ✗ | **✓** |
| Purpose-built authentication dataset | ✗ | ✗ | ✗ | **✓** (CredGuardV1) |
| Controlled ML vs. DL benchmark | ✗ | partial | partial | **✓** (6 architectures) |
| Global **and** local explainability | ✗ | ✗ | partial | **✓** (SHAP + LIME) |
| Post-login behavioural monitoring | ✗ | ✗ | ✗ | **✓** |
| Autonomous response on physical IoT hardware | partial | ✗ | ✗ | **✓** (Raspberry Pi) |

### Numeric benchmark

All six architectures were trained and evaluated on the same CredGuardV1 partitions (70% train / 15% validation / 15% test), under identical preprocessing and an identical leakage-free feature set. `Attack` is the positive class. Training times were measured on the Google Colab and Kaggle environments described in [REPRODUCIBILITY.md](REPRODUCIBILITY.md).

| Model | Accuracy | Precision | Recall | F1-Score | Training time (s) |
| --- | --- | --- | --- | --- | --- |
| **XGBoost** | **82.29%** | 85.06% | 75.45% | 79.97% | **82** |
| Random Forest | 82.03% | **86.90%** | 76.02% | 81.10% | 585 |
| SVM | 71.21% | 75.10% | 64.62% | 69.47% | 54 |
| CNN | 80.50% | 77.55% | **86.60%** | 81.82% | 2,640 |
| RNN | 80.94% | 79.04% | 84.90% | **81.87%** | 3,794 |
| DNN | 80.22% | 78.07% | 84.79% | 81.29% | 2,857 |

<p align="center">
  <img src="Figures/model_benchmark.svg" alt="Comparative model performance on CredGuardV1" width="780">
</p>

The deep architectures achieve the strongest recall — CNN reaches 86.60%, consistent with their capacity to extract higher-order feature interactions — but they do so at a training cost between 32× and 46× that of XGBoost, and none exceeds it on accuracy. SVM is the weakest configuration on every quality metric. **XGBoost was therefore selected for deployment**, on the basis of the accuracy-to-latency trade-off rather than accuracy alone: it attains the highest accuracy in the study while training in 82 seconds and scoring fast enough for real-time use on constrained hardware. Random Forest attains marginally higher precision and F1, but at roughly seven times the training cost and with heavier inference, which is decisive in a real-time setting.

> An LSTM architecture was implemented but excluded from the reported comparison. Its sequential processing imposed a prohibitive training cost without a corresponding accuracy gain, since the engineered feature set already encodes temporal structure explicitly (inter-arrival statistics, attempt progression, cyclical hour encoding) rather than leaving it to be recovered from raw sequences.

The table and figure regenerate from [`Data/model_benchmark.csv`](Data/model_benchmark.csv) with `python3 Scripts/plot_model_benchmark.py`.

---

## Features

### 1. CredGuardV1 — a purpose-built authentication dataset

Public IoT intrusion datasets are overwhelmingly flow-oriented and contain few labelled credential events. CredGuardV1 was constructed to fill that gap, by integrating five heterogeneous authentication sources into a single labelled corpus.

| # | Source | Raw records | Contribution |
| --- | --- | --- | --- |
| 1 | SSH honeypot logs (unstructured `.txt`) | 84,467 | Raw malicious traffic against administrative ports |
| 2 | Cowrie medium-interaction SSH honeypot | 40,128 | Bot credentials and session-level metadata |
| 3 | Risk-Based Authentication (RBA) logs | 31,269,264 | Behavioural scale; legitimate vs. large-scale campaigns |
| 4 | SSH brute-force logs (JSON) | — | High-density targeted brute-force patterns |
| 5 | Web-based intrusion logs | — | Ground-truth labels in a local-network setting |

After schema alignment, cleaning, and class balancing, the released corpus contains approximately **1,003,030 records** over a nine-field common schema: `timestamp`, `src_ip`, `username`, `password`, `asn`, `location`, `status`, `source`, `label`.

#### What we do here

- **Schema alignment.** Heterogeneous column names are mapped to the common schema; heterogeneous success indicators are normalised to a binary `status` field. Multi-file sources such as Cowrie are joined relationally on session identifiers to link authentication attempts to their network metadata. Sources lacking a field receive an initialised placeholder column, preserving a consistent feature matrix.
- **Cleaning and quality assurance.** Records duplicated across all fields, including microsecond-resolution timestamps, are removed to prevent optimistic evaluation. Records missing a critical identifier such as `src_ip` are discarded. Missing credential fields are populated from the Mirai credentials list, a set of genuinely compromised credentials, so that simulated brute-force and credential-stuffing attempts remain realistic. ASN and geographic origin are resolved from `src_ip`; records whose enrichment cannot be resolved are dropped rather than imputed.
- **Temporal standardisation.** All timestamps are synchronised to a high-precision `datetime64[us]` representation, which is a prerequisite for accurate inter-arrival computation.
- **Class balancing.** The corpus is initially dominated by the 31.2M RBA records. Rather than random undersampling, a **group-based stratified sampling** strategy is applied: sampling is performed at the source-IP level, so that the *entire* historical sequence of every selected IP is retained. This preserves the temporal evolution of each campaign, which random sampling destroys.

### 2. Leakage-free behavioural feature engineering

Every temporal feature is computed with **expanding windows over the past only**, eliminating look-ahead bias and ensuring that offline results transfer to online deployment.

- **Inter-arrival time (`iat`)** — seconds between the current request and the previous request from the same source IP.
- **Cumulative timing statistics (`mean_iat`, `std_iat`)** — separate automated activity, which is highly regular, from human activity, which is not.
- **Attempt progression (`login_attempt`)** — a sequential counter per source, exposing persistence.
- **Failure ratio (`fail_rate`)** — failed attempts over attempts to date; a real-time brute-force indicator.
- **Credential structure (`p_len`, `p_comp`)** — password length and structural complexity, scored over character-class composition.
- **Network context (`asn`)** — source network identity, supporting reputation and volume analysis per provider.
- **Geographic context** — country of origin, exposing anomalous access geographies.
- **Cyclical hour encoding** — hour-of-day projected onto a sine/cosine plane, so that 23:59 and 00:01 are correctly represented as adjacent rather than maximally distant.

### 3. Controlled ML/DL benchmark

Six architectures — XGBoost, Random Forest, SVM, CNN, RNN, DNN — are trained under identical conditions and reported over Accuracy, Precision, Recall, F1-Score, and training time, with per-model confusion matrices and DL convergence curves. The benchmark is an experimental result of the study, not a preliminary step: it is what establishes that gradient boosting dominates deep architectures on this task once latency is treated as a first-class constraint.

### 4. Explainability (SHAP + LIME)

<p align="center">
  <img src="Figures/shap_global_importance.svg" alt="Global SHAP feature importance" width="720">
</p>

**Global interpretability — SHAP.** Shapley values quantify each feature's mean contribution across the corpus. `fail_rate` and `p_len` emerge as the dominant attributions. The beeswarm decomposition further shows that high `p_len` and `p_comp` values push strongly toward the `Attack` class — long, structurally complex passwords being characteristic of automated credential-stuffing dictionaries rather than of human-chosen credentials — and that the model relies more heavily on credential structure than on timing.

**Local interpretability — LIME.** For any individual decision, LIME produces a locally faithful approximation identifying which features supported the prediction and which opposed it, with magnitudes. These explanations are rendered directly in the analyst dashboard, so an alert arrives with its justification attached.

### 5. Post-login behavioural risk engine

Session-level monitoring against a per-user baseline, combining six behavioural indicators into a 0–100 risk score with `Low` / `Medium` / `High` bands, and a rule-based fallback for users without sufficient history. This provides a second detection boundary beyond authentication, targeting exactly the case where the credential itself is valid but the actor is not.

### 6. Deployed IoT integration

A Raspberry Pi 3 Model B+ with a Camera Module v2.1 (CSI, 640 × 480) serves as the physical IoT node, providing live streaming, on-demand snapshots, and timestamped recording. The node communicates with the Flask backend over HTTP and acts as the **execution layer** for access-control decisions, maintaining a clear separation between analysis on the server and enforcement on the device.

---

## Installation

### Prerequisites

- Python 3.9 or later
- PostgreSQL 13 or later
- Required Python packages:

```bash
pip install -r requirements.txt
# (scikit-learn, xgboost, tensorflow, shap, lime, pandas, numpy,
#  flask, flask-sqlalchemy, psycopg2-binary, matplotlib, seaborn, tqdm)
```

- For the IoT node (Raspberry Pi OS): `picamera2`, `opencv-python`, `flask`
- Optional: an [IPinfo](https://ipinfo.io) API token for ASN and geolocation enrichment

### Setup

1. Clone the repository:

```bash
git clone https://github.com/CredGuard2026/CredGuard.git
cd CredGuard
```

2. Install dependencies:

```bash
pip install -r requirements.txt
```

3. Configure the environment. Credentials are read from environment variables only and are **never stored in the repository**:

```bash
export DATABASE_URL=postgresql://user:password@localhost:5432/credguard
export IPINFO_TOKEN=...        # optional, enables ASN/geo enrichment
export RASPBERRY_PI_HOST=...   # optional, IoT node network address
```

4. Initialise the database schema:

```bash
python3 Scripts/init_db.py
```

---

## Usage

### 1. Build the CredGuardV1 Dataset

Aligns the five raw sources to the common schema, cleans and enriches them, and applies group-based stratified sampling.

#### Command:

```bash
python3 Scripts/build_dataset.py -i Data/raw/ -o Data/CredGuardV1.csv [--sample-size 1003030]
```

#### Arguments:

- `-i/--input`: directory containing the raw source files.
- `-o/--output`: path for the unified master dataset.
- `--sample-size`: target record count after balancing (default: `1003030`).
- `--ipinfo-token`: optional; overrides `IPINFO_TOKEN`.

### 2. Behavioural Feature Engineering

Transforms the standardised dataset into the leakage-free behavioural feature matrix.

#### Command:

```bash
python3 Scripts/feature_engineering.py -i Data/CredGuardV1.csv -o Data/CredGuardV1_features.csv
```

All temporal statistics are computed with expanding windows over past events only. The script asserts this property and will fail rather than emit a leaking feature.

### 3. Train the Machine Learning Models

#### Command:

```bash
python3 Scripts/train_ml.py --model xgboost --data Data/CredGuardV1_features.csv --out Models/
python3 Scripts/train_ml.py --model rf      --data Data/CredGuardV1_features.csv --out Models/
python3 Scripts/train_ml.py --model svm     --data Data/CredGuardV1_features.csv --out Models/
```

#### Deployed XGBoost configuration:

| Hyperparameter | Value |
| --- | --- |
| `objective` | `binary:logistic` |
| `n_estimators` | 400 |
| `learning_rate` | 0.05 |
| `max_depth` | 8 |
| `subsample` | 0.9 |
| `colsample_bytree` | 0.9 |
| `gamma` | 0.1 |
| `min_child_weight` | 3 |
| `early_stopping_rounds` | 20 |
| `eval_metric` | `logloss` |
| `random_state` | 42 |

### 4. Train the Deep Learning Models

#### Command:

```bash
python3 Scripts/train_dl.py --model cnn --data Data/CredGuardV1_features.csv --out Models/
python3 Scripts/train_dl.py --model rnn --data Data/CredGuardV1_features.csv --out Models/
python3 Scripts/train_dl.py --model dnn --data Data/CredGuardV1_features.csv --out Models/
```

Loss and accuracy curves are written to `Figures/` for convergence inspection. Reference notebooks for all six architectures are available in [`Notebooks/`](Notebooks/).

### 5. Generate XAI Explanations (SHAP / LIME)

#### Command:

```bash
# Global attribution (bar + beeswarm)
python3 Scripts/explain_shap.py --model Models/xgboost.json \
    --data Data/CredGuardV1_features.csv --out Figures/

# Local explanation for a single decision
python3 Scripts/explain_lime.py --model Models/xgboost.json \
    --data Data/CredGuardV1_features.csv --instance 42 --out Figures/
```

### 6. Run the Detection Backend

Starts the Flask service: authentication endpoints, real-time scoring, the analyst dashboard, and the mitigation dispatcher.

#### Command:

```bash
python3 app.py --host 0.0.0.0 --port 5000
```

The dashboard exposes live login telemetry, the alert queue, blocked-account management, and the SHAP/LIME analytics panel.

### 7. Deploy the IoT Node (Raspberry Pi)

Run on the Raspberry Pi itself:

#### Command:

```bash
python3 IoT/camera_server.py --resolution 640x480 --port 8000
```

Endpoints: `/stream` (live MJPEG), `/snapshot` (still frame), `/record/start`, `/record/stop`. Recordings are stored locally with timestamp-based identifiers. Set `RASPBERRY_PI_HOST` on the backend so it can reach the node.

### 8. Post-Login Behavioural Risk Scoring

Scores an active session against the user's historical baseline.

#### Command:

```bash
python3 Scripts/behaviour_risk.py --session-id <SESSION_ID>
```

#### Output:

```json
{
  "session_id": "3f9c1a24",
  "risk_score": 78,
  "risk_level": "High",
  "triggered_indicators": [
    "session_duration_below_25pct_of_baseline",
    "live_stream_not_watched",
    "no_camera_interaction"
  ],
  "action": "terminate_session"
}
```

---

## Reproducibility

The full reproducibility appendix — source provenance, exact cleaning and sampling rules, the complete feature specification with equations, the leakage-prevention protocol, model hyperparameters, hardware and runtime environments, and the evaluation-metric definitions — is documented in:

➡️ **[REPRODUCIBILITY.md](REPRODUCIBILITY.md)**

Highlights:

- Partitions: **70% / 15% / 15%** train / validation / test, `random_state=42` throughout.
- All temporal features use **expanding windows over past events only**; no look-ahead bias.
- Deduplication at microsecond timestamp resolution, applied before partitioning.
- Class balancing via **group-based stratified sampling at the source-IP level**, preserving full per-IP sequences.
- Training environments: Google Colab and Kaggle; runtimes reported per model.
- `Attack` is the positive class for all Precision, Recall, and F1 figures.

## Data Availability

In line with open-science principles, this repository releases the artefacts required to reproduce every quantitative result reported in the paper:

- [`Data/model_benchmark.csv`](Data/model_benchmark.csv) — per-model performance metrics and training times (paper Table 5.14).
- [`Data/feature_schema.md`](Data/feature_schema.md) — the complete feature specification with definitions and dtypes.
- [`Data/shap_global_importance.csv`](Data/shap_global_importance.csv) — mean absolute SHAP values per feature.
- [`Notebooks/`](Notebooks/) — training notebooks for all six architectures.

The CredGuardV1 corpus is derived from honeypot and authentication logs containing source IP addresses and credential material. It is therefore released in **aggregated and anonymised** form, with IP addresses truncated and credential strings hashed. The full corpus is available from the corresponding author on reasonable request, subject to responsible-disclosure and data-protection constraints.

---

## Directory Structure

```
CredGuard/
├── Scripts/
│   ├── build_dataset.py              # Schema alignment, cleaning, enrichment, balancing
│   ├── feature_engineering.py        # Leakage-free behavioural feature construction
│   ├── train_ml.py                   # XGBoost / Random Forest / SVM training
│   ├── train_dl.py                   # CNN / RNN / DNN training
│   ├── explain_shap.py               # Global attribution (bar + beeswarm)
│   ├── explain_lime.py               # Local, per-decision explanations
│   ├── behaviour_risk.py             # Post-login risk scoring engine
│   ├── plot_model_benchmark.py       # Renders the comparative benchmark figure
│   ├── plot_confusion_matrices.py    # Renders per-model confusion matrices
│   └── init_db.py                    # PostgreSQL schema initialisation
├── Backend/
│   ├── app.py                        # Flask application entry point
│   ├── db.py                         # SQLAlchemy database object
│   ├── models.py                     # ORM table definitions
│   ├── detection.py                  # Real-time inference and mitigation dispatch
│   └── routes/                       # Authentication, dashboard, analytics endpoints
├── Frontend/
│   ├── templates/                    # Admin dashboard and device-owner interface
│   └── static/                       # Stylesheets, scripts, assets
├── IoT/
│   ├── camera_server.py              # Raspberry Pi camera service
│   └── decision_executor.py          # Executes backend access-control decisions
├── Notebooks/
│   ├── XGB_Model.ipynb               # XGBoost training and tuning
│   ├── RF_Model.ipynb                # Random Forest training
│   ├── SVM_Model.ipynb               # SVM training
│   ├── CNN_Model.ipynb               # CNN training
│   ├── RNN_Model.ipynb               # RNN training
│   └── DNN_Model.ipynb               # DNN training
├── Data/
│   ├── model_benchmark.csv           # Per-model metrics (paper Table 5.14)
│   ├── feature_schema.md             # Feature specification
│   └── shap_global_importance.csv    # Mean absolute SHAP values
├── Models/                           # Serialised trained models (gitignored)
├── Figures/
│   ├── CredGuard.jpg                 # Project banner
│   ├── architecture_overview.svg     # End-to-end pipeline
│   ├── behaviour_engine.svg          # Post-login risk scoring
│   ├── model_benchmark.svg           # Comparative model performance
│   └── shap_global_importance.svg    # Global SHAP attribution
├── requirements.txt                  # Python dependencies
├── CITATION.cff                      # Machine-readable citation metadata
├── REPRODUCIBILITY.md                # Reproducibility appendix
├── LICENSE                           # MIT License
└── README.md                         # This file
```

---

## Citation

If you use CredGuard or the CredGuardV1 feature specification in your research, please cite:

> Alamri, R., Alharbi, S., Alotaibi, K., Alranini, A., Alraddadi, A., Alkhulaifi, R., and Alharbi, F. "CredGuard: A Credential-Based Intrusion Detection System for IoT Environments Using Explainable Artificial Intelligence." (2026).

Machine-readable metadata is provided in [`CITATION.cff`](CITATION.cff).

---

## Contributing

Contributions are welcome. To contribute:

1. Fork this repository.
2. Create a new branch (`git checkout -b feature/YourFeatureName`).
3. Commit your changes (`git commit -m "Add some feature"`).
4. Push to the branch (`git push origin feature/YourFeatureName`).
5. Open a pull request.

## License

Released under the MIT License. See [LICENSE](LICENSE) for details.
