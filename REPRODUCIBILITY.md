# Reproducibility Appendix

This appendix documents the data processing, feature engineering, model configurations, evaluation protocol, and post-login behaviour analysis used in the CredGuard project. It is intended to be read alongside [README.md](README.md).

---

## 1. Data Sources and Provenance

CredGuard IoT integrates five heterogeneous authentication and intrusion-related data sources. Each source was processed independently before integration.

| # | Source | Raw Records | Role in the Dataset |
|---|---|---:|---|
| 1 | SSH Honeypot Logs | 84,467 | Malicious authentication and interaction attempts |
| 2 | Medium-Interaction SSH Honeypot (Cowrie) | 40,128 | Authentication attempts with usernames, passwords, and session-level metadata |
| 3 | Risk-Based Authentication (RBA) | 31,269,264 | Large-scale authentication and behavioural data |
| 4 | SSH Brute-Force Dataset | — | Targeted brute-force authentication activity |
| 5 | Synthetic Web Authentication Logs | — | Authentication data containing normal and malicious activity |

The five sources were integrated to construct the CredGuard IoT dataset.

The SSH brute-force source contains multiple password attempts associated with usernames and was processed so that individual attempts could be represented during dataset construction.

---

## 2. Common Schema

The integrated data were aligned to a common schema containing the following fields:

| Field | Description |
|---|---|
| `timestamp` | Timestamp of the authentication attempt |
| `src_ip` | Source IP address |
| `username` | Submitted username |
| `password` | Submitted password |
| `asn` | Autonomous System Number associated with the source IP |
| `location` | Geographic location associated with the source IP |
| `status` | Authentication outcome: `1` = success, `0` = failure |
| `source` | Original dataset identifier |
| `label` | Ground-truth class: `1` = Attack, `0` = Normal |

Datasets that did not originally contain some fields were aligned to the common structure during preprocessing.

---

## 3. Data Cleaning and Preprocessing

The preprocessing pipeline included the following steps:

1. **Schema alignment and integration.**  
   Source-specific fields were mapped to a common structure. Multi-file sources such as Cowrie were joined using session identifiers.

2. **Status normalization.**  
   Different authentication outcome representations were converted to the binary `status` field.

3. **Duplicate removal.**  
   Identical records were removed from the integrated data.

4. **Critical-field filtering.**  
   Records without a valid `src_ip` were removed because the source IP is required for the temporal feature calculations.

5. **Credential completion.**  
   Missing username and password values were addressed using credentials from the Mirai Credentials List.

6. **Network enrichment.**  
   ASN and geographic location information were obtained from the source IP using the IPinfo service.

7. **Temporal standardization.**  
   Timestamps were standardized to support temporal feature extraction and inter-arrival time calculations.

---

## 4. Dataset Balancing and Sampling

The integrated dataset contains a large proportion of records originating from the RBA dataset.

To construct the final corpus, group-based sampling was performed at the source-IP level.

- The data were separated into `Normal` and `Attack` classes.
- Sampling was performed using source IPs rather than individual records.
- The complete historical sequence associated with each selected source IP was retained.

This approach preserves the temporal structure required for features such as inter-arrival time, cumulative statistics, login-attempt progression, and failure ratio.

The resulting CredGuard IoT dataset contains approximately **1,003,030 records**.

---

## 5. Feature Specification

Feature engineering was performed using authentication, temporal, credential, and network-context information.

### 5.1 Temporal Features

Temporal features were calculated using the ordered authentication attempts associated with each source IP.

#### Inter-Arrival Time

The time difference between consecutive authentication attempts was calculated as:

$$
\text{iat}_k = t_k - t_{k-1}
$$

where $t_k$ represents the timestamp of the current attempt.

#### Login Attempt Progression

A sequential counter was used to represent the progression of login attempts for a source:

$$
\text{loginAttempt}_k = k
$$

#### Failure Ratio

The failure ratio was calculated as the proportion of failed attempts observed up to the current attempt:

$$
\text{failRate}_k =
\frac{\sum_{j=1}^{k}\mathbb{1}[\text{status}_j=0]}{k}
$$

Cumulative temporal statistics, including mean and standard deviation of inter-arrival time, were also derived from the ordered authentication sequence.

---

### 5.2 Credential Features

Credential-related features included:

- Password length
- Password complexity

Password complexity was evaluated using the structural characteristics of the password, including lowercase letters, uppercase letters, digits, and special characters.

---

### 5.3 Cyclical Time Encoding

The hour of the day was represented using sine and cosine transformations:

$$
\text{hourSin} =
\sin\left(\frac{2\pi h}{24}\right)
$$

$$
\text{hourCos} =
\cos\left(\frac{2\pi h}{24}\right)
$$

This encoding represents the cyclical relationship between hours of the day.

---

### 5.4 Contextual Features

Network and geographic context were included through:

- `asn` — Autonomous System Number associated with the source IP.
- `location` — Geographic location associated with the source IP.

---

## 6. Leakage Prevention

Temporal feature engineering was designed to avoid using future information during feature construction.

The project uses expanding historical information when calculating temporal statistics so that the features available for an authentication attempt are based on the relevant preceding activity.

The temporal features are calculated within individual `src_ip` groups to preserve the behavioural history of each source.

Duplicate removal is also performed before model partitioning to prevent duplicate records from appearing across different data partitions.

---

## 7. Dataset Partitioning

The integrated dataset was divided into training, validation, and test sets using the following proportions:

| Partition | Proportion | Purpose |
|---|---:|---|
| Training | 70% | Model training |
| Validation | 15% | Model validation and model configuration |
| Test | 15% | Final performance evaluation |

For the evaluation metrics, `Attack` is treated as the positive class and `Normal` as the negative class.

---

## 8. Model Configurations

CredGuard evaluated Machine Learning and Deep Learning models for authentication-based intrusion detection.

The evaluated models were:

- XGBoost
- Random Forest
- Support Vector Machine (SVM)
- Convolutional Neural Network (CNN)
- Recurrent Neural Network (RNN)
- Deep Neural Network (DNN)

### 8.1 XGBoost

The reported XGBoost configuration used the following parameters:

| Hyperparameter | Value |
|---|---:|
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

XGBoost was selected as the final model based on the reported balance between predictive performance and computational efficiency.

### 8.2 Random Forest and SVM

Random Forest and SVM were implemented using scikit-learn and evaluated using the same dataset partitioning and feature representation.

### 8.3 Deep Learning Models

The evaluated Deep Learning models were:

- **CNN**
- **RNN**
- **DNN**

These models were trained and evaluated as part of the comparative model analysis.

### 8.4 LSTM

An LSTM model was implemented but excluded from the reported model comparison. The report states that its sequential processing required substantially longer training time and was less suitable for the dataset characteristics used in the study.

---

## 9. Evaluation Metrics

The following metrics were used to evaluate model performance:

### Accuracy

$$
\text{Accuracy} =
\frac{TP + TN}{TP + FP + TN + FN}
$$

### Precision

$$
\text{Precision} =
\frac{TP}{TP + FP}
$$

### Recall

$$
\text{Recall} =
\frac{TP}{TP + FN}
$$

### F1-Score

$$
F_1 =
\frac{2 \cdot \text{Precision} \cdot \text{Recall}}
{\text{Precision} + \text{Recall}}
$$

where:

- `TP` = True Positive
- `TN` = True Negative
- `FP` = False Positive
- `FN` = False Negative

`Attack` represents the positive class and `Normal` represents the negative class.

---

## 10. Model Performance

The reported performance of the evaluated models on the CredGuard IoT dataset is shown below.

| Model | Accuracy | Precision | Recall | F1-Score | Training Time (s) |
|---|---:|---:|---:|---:|---:|
| XGBoost | 82.29% | 85.06% | 75.45% | 79.97% | 82 |
| Random Forest | 82.03% | 86.90% | 76.02% | 81.10% | 585 |
| SVM | 71.21% | 75.10% | 64.62% | 69.47% | 54.43 |
| CNN | 80.50% | 77.55% | 86.60% | 81.82% | 2640.02 |
| RNN | 80.94% | 79.04% | 84.90% | 81.87% | 3794 |
| DNN | 80.22% | 78.07% | 84.79% | 81.29% | 2857 |

---

## 11. Explainable AI

CredGuard uses Explainable AI techniques to interpret model predictions.

The project applies:

- **SHAP (SHapley Additive exPlanations)** for feature contribution and global feature-importance analysis.
- **LIME (Local Interpretable Model-agnostic Explanations)** for local interpretation of individual predictions.

These techniques are used to provide interpretable information about the model's classification results.

---

## 12. Post-Login Risk Scoring

CredGuard also performs behavioural analysis after successful authentication.

The system monitors user activity during the session and evaluates the current behaviour against historical user behaviour when sufficient historical data are available.

The behavioural analysis includes indicators such as:

| Indicator | Deviation Condition |
|---|---|
| Mean previous session duration | Current duration is below 25% of the historical mean |
| Live-stream watching rate | Historical rate is at least 70%, but the current session does not meet the expected behaviour |
| Camera usage rate | Camera interaction is frequent historically but absent in the current session |
| Mean camera interaction count | Historical baseline is at least 2 interactions per session, but no interaction occurs in the current session |
| Full-screen usage rate | Historical rate is at least 70%, but full-screen is not used in the current session |
| Mean full-screen usage | Historical average is at least 1 interaction per session, but no full-screen usage occurs in the current session |

A live-stream interaction of at least 10 seconds is considered meaningful, while sessions shorter than 3 seconds are treated as highly abnormal.

A composite **Risk Score from 0 to 100** is calculated and classified into:

| Risk Level | Interpretation |
|---|---|
| Low Risk | Normal behaviour |
| Medium Risk | Suspicious behaviour |
| High Risk | Abnormal and high-risk behaviour |

When the risk level is high, the session may be terminated or blocked.

When insufficient historical data are available, rule-based heuristics are used to estimate the risk level.

---

## 13. Runtime Environment

The project uses the following development and implementation technologies:

- Python
- scikit-learn
- TensorFlow / Keras
- XGBoost
- SHAP
- LIME
- Jupyter
- Google Colab
- Kaggle
- Flask
- PostgreSQL
- Flask-SQLAlchemy
- Raspberry Pi 3 Model B+
- Raspberry Pi Camera Module v2.1

The Raspberry Pi Camera Module v2.1 is connected to the Raspberry Pi through the Camera Serial Interface (CSI), and the reported video configuration uses a resolution of **640 × 480 pixels**.

The backend uses Flask with PostgreSQL through Flask-SQLAlchemy.

---

## 14. Database and Security Data

The PostgreSQL database stores core system information including:

- User accounts
- Login attempts
- Model results
- Security actions
- IoT device data
- Post-login behaviour records

The project also stores records related to user activity and post-login behaviour analysis.

Passwords are not stored in plaintext. The implementation uses password hashing and verification functions for user authentication.

---

## 15. IoT Decision Execution

The Raspberry Pi functions as the IoT component of the system.

Authentication-related information is transmitted to the backend for analysis. After the backend generates an access-control decision, the Raspberry Pi receives the result and executes the corresponding action on the IoT device.

The Raspberry Pi Camera Module v2.1 also supports live video streaming, image snapshots, and video recording as described in the system implementation.

---
