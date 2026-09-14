# Reproducibility Appendix

This appendix documents the parameters, equations, and protocols required to reproduce every quantitative result reported for CredGuard. It is intended to be read alongside [README.md](README.md).

---

## 1. Data Sources and Provenance

CredGuardV1 integrates five heterogeneous authentication sources. Each was obtained from a public repository and processed independently before integration.

| # | Source | Format | Raw records | Role in the corpus |
| --- | --- | --- | --- | --- |
| 1 | SSH honeypot logs | Unstructured `.txt` | 84,467 | Malicious interaction attempts against administrative ports |
| 2 | Cowrie medium-interaction SSH honeypot | Multi-file, session-keyed | 40,128 | Bot-supplied credentials with session-level metadata |
| 3 | Risk-Based Authentication (RBA) logs | Tabular | 31,269,264 | Behavioural scale; legitimate users and large-scale campaigns |
| 4 | SSH brute-force logs | JSON (nested) | — | Dense targeted brute-force against specific usernames |
| 5 | Web-based intrusion logs | Tabular | — | Ground-truth labels in a local-network setting |

Source 4 is stored with multiple password variants nested under a single username record. These are **expanded** so that each individual attempt becomes a distinct row prior to integration; scoring nested records as single events would systematically understate attempt counts and failure ratios.

---

## 2. Common Schema

All sources are mapped to a nine-field schema:

| Field | Type | Description |
| --- | --- | --- |
| `timestamp` | `datetime64[us]` | Event time, microsecond resolution |
| `src_ip` | `string` | Source IP address of the attempt |
| `username` | `string` | Submitted username |
| `password` | `string` | Submitted password |
| `asn` | `int` | Autonomous System Number, resolved from `src_ip` |
| `location` | `string` | Country of origin, resolved from `src_ip` |
| `status` | `int` | Outcome: `1` = success, `0` = failure |
| `source` | `string` | Originating dataset identifier |
| `label` | `int` | Ground truth: `1` = Attack, `0` = Normal |

Sources lacking a given field receive an initialised placeholder column so that the feature matrix remains structurally consistent across the integration.

---

## 3. Cleaning Protocol

Applied in the following order. Order matters: deduplication precedes enrichment so that enrichment cost is not paid on records that will be discarded.

1. **Relational merging.** Multi-file sources (Cowrie) are joined on session identifiers to link authentication attempts to their corresponding network metadata.
2. **Status normalisation.** Heterogeneous success indicators (`success`, `Login Status`, and equivalents) are mapped to the binary `status` field.
3. **Deduplication.** Records identical across *all* fields, including microsecond-resolution timestamps, are removed. Retaining them would inflate apparent accuracy by permitting near-identical records to appear in both training and test partitions.
4. **Critical-field filtering.** Records missing `src_ip` are discarded; the field is irrecoverable and is the grouping key for every temporal feature.
5. **Credential imputation.** Missing `username` / `password` values are populated from the Mirai credentials list — genuinely compromised credentials from real breaches — so that simulated brute-force and credential-stuffing attempts retain realistic structural properties. Synthetic random strings were rejected for this purpose, as they would distort the `p_len` and `p_comp` distributions.
6. **Network enrichment.** `asn` and `location` are resolved from `src_ip` via the IPinfo service. Records whose enrichment cannot be resolved are **dropped, not imputed**, since an imputed ASN is a fabricated network identity.
7. **Temporal standardisation.** All timestamps are synchronised and converted to `datetime64[us]`, a precondition for accurate inter-arrival computation.

---

## 4. Class Balancing

The integrated corpus is dominated by the 31.2M RBA records, producing severe imbalance.

**Group-based stratified sampling** is applied at the source-IP level:

- The corpus is stratified into `Normal` and `Attack` classes.
- Sampling selects **source IPs**, not individual records.
- For every selected IP, the **entire historical sequence is retained**.

The resulting corpus contains approximately **1,003,030 records**.

Sequence preservation is essential and non-negotiable. Random record-level sampling would fragment per-IP histories, corrupting `iat`, `mean_iat`, `std_iat`, `login_attempt`, and `fail_rate` — all of which are defined over an IP's ordered attempt sequence. A fragmented sequence yields features that describe a campaign that never occurred.

---

## 5. Feature Specification

Let $S_i$ denote the ordered sequence of attempts originating from source IP $i$, and let $t_k$ denote the timestamp of the $k$-th attempt in that sequence. All statistics are computed over **expanding windows containing past events only**.

### 5.1 Temporal features

**Inter-arrival time.** Seconds between consecutive attempts from the same source:

$$\text{iat}_k = t_k - t_{k-1}, \qquad \text{iat}_1 = 0$$

**Cumulative mean inter-arrival time:**

$$\text{meanIat}_k = \frac{1}{k-1}\sum_{j=2}^{k}\text{iat}_j$$

**Cumulative standard deviation of inter-arrival time:**

$$\text{stdIat}_k = \sqrt{\frac{1}{k-1}\sum_{j=2}^{k}\left(\text{iat}_j - \text{meanIat}_k\right)^2}$$

Low `std_iat` indicates machine-regular timing; higher values are characteristic of human interaction.

**Attempt progression.** A sequential counter over the source's attempts:

$$\text{loginAttempt}_k = k$$

**Failure ratio.** Failed attempts as a proportion of attempts to date:

$$\text{failRate}_k = \frac{\sum_{j=1}^{k}\mathbb{1}[\text{status}_j = 0]}{k}$$

### 5.2 Credential features

**Password length:** $\text{pLen} = |p|$

**Password complexity.** A score over character-class composition, incrementing for the presence of lowercase letters, uppercase letters, digits, and special characters, weighted by length:

$$\text{pComp} = \sum_{c \in C}\mathbb{1}[c \text{ present in } p] \cdot \log(1 + |p|)$$

where $C = \{\text{lower}, \text{upper}, \text{digit}, \text{special}\}$.

### 5.3 Cyclical time encoding

Hour-of-day is projected onto a two-dimensional plane so that hour boundaries are represented as continuous:

$$\text{hourSin} = \sin\left(\frac{2\pi h}{24}\right), \qquad \text{hourCos} = \cos\left(\frac{2\pi h}{24}\right)$$

Under a linear encoding, 23:59 and 00:01 are maximally distant despite being two minutes apart. The sine/cosine projection removes this artefact.

### 5.4 Contextual features

- `asn` — source network identity, retained as a categorical feature.
- `location` — country of origin, retained as a categorical feature.

---

## 6. Leakage Prevention Protocol

Three properties are enforced and asserted in `Scripts/feature_engineering.py`, which fails rather than emitting a leaking feature:

1. **No look-ahead.** Every temporal statistic at index $k$ is computed exclusively from indices $1 \ldots k$. Vectorised operations that would silently consume future rows (centred rolling windows, full-column aggregates, backward fills) are prohibited.
2. **Grouped computation.** All sequence features are computed within `src_ip` groups. Cross-IP contamination would leak one source's behaviour into another's feature vector.
3. **Deduplication before partitioning.** Duplicate removal precedes the train/validation/test split, so near-identical records cannot straddle the partition boundary.

Without these constraints, offline accuracy is inflated and does not transfer to online deployment, where future events are by definition unavailable.

---

## 7. Partitioning

| Partition | Proportion | Purpose |
| --- | --- | --- |
| Training | 70% | Model fitting |
| Validation | 15% | Early stopping and hyperparameter selection |
| Test | 15% | Reported metrics only; used once |

`random_state = 42` throughout. `Attack` is the positive class for all Precision, Recall, and F1 figures.

---

## 8. Model Configurations

### 8.1 XGBoost (deployed)

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

`subsample` and `colsample_bytree` control variance; `gamma` and `min_child_weight` constrain tree growth; early stopping terminates training once validation `logloss` ceases to improve.

### 8.2 Random Forest and SVM

Implemented via scikit-learn. Random Forest uses parallel tree construction with majority voting. SVM uses an RBF kernel. Both are fitted on the identical feature matrix and partitions.

### 8.3 Deep architectures

- **CNN** — convolutional feature extraction followed by dense layers with dropout regularisation.
- **RNN** — recurrent layers maintaining hidden state across the input representation.
- **DNN** — fully connected layers learning hierarchical feature interactions.

All three converge with declining training loss and a narrow train/validation gap, indicating effective generalisation without significant overfitting. Loss and accuracy curves are reproduced in `Figures/`.

### 8.4 Excluded architecture

An LSTM was implemented but excluded from the reported comparison. Its sequential processing imposed a prohibitive training cost without a corresponding accuracy gain: the engineered feature set already encodes temporal structure explicitly through inter-arrival statistics, attempt progression, and cyclical hour encoding, leaving little residual sequential signal for the recurrent architecture to recover.

---

## 9. Evaluation Metrics

With TP, TN, FP, FN denoting true positives, true negatives, false positives, and false negatives, and `Attack` as the positive class:

$$\text{Accuracy} = \frac{TP + TN}{TP + FP + TN + FN}$$

$$\text{Precision} = \frac{TP}{TP + FP}$$

$$\text{Recall} = \frac{TP}{TP + FN}$$

$$F_1 = \frac{2 \cdot \text{Precision} \cdot \text{Recall}}{\text{Precision} + \text{Recall}}$$

In this application the two error types carry asymmetric cost. A false negative admits an attacker; a false positive locks out a legitimate device owner. F1 is reported as the balanced summary, but model selection also weighed training and inference latency, which is what led to XGBoost over the marginally more precise Random Forest.

---

## 10. Post-Login Risk Scoring

For a user with sufficient session history, a behavioural baseline is constructed and the current session is scored against it.

| Indicator | Deviation condition |
| --- | --- |
| Mean previous session duration | Current duration < 25% of historical mean |
| Live-stream watching rate | Baseline ≥ 70% of sessions, but not watched in current |
| Camera usage rate | Frequent historically, but no camera or alternative interaction in current |
| Mean camera interaction count | Baseline ≥ 2 per session, but none in current |
| Full-screen usage rate | Baseline ≥ 70% of sessions, but unused in current |
| Mean full-screen usage | Baseline ≥ 1 per session, but none in current |

**Session thresholds.** A live-stream view is considered meaningful at ≥ 10 seconds. Sessions under 3 seconds are treated as highly abnormal, as they permit no meaningful interaction.

**Risk bands.** Indicator deviations are aggregated into a composite score on a 0–100 scale:

| Band | Interpretation |
| --- | --- |
| Low | Consistent with the user's established behaviour |
| Medium | Suspicious; flagged for analyst review |
| High | Abnormal; session terminated or blocked |

**Cold start.** Where history is insufficient to support a baseline, a rule-based heuristic applies: short sessions without interaction raise the score, normal interaction lowers it. Missing `session_duration_sec` values are treated as zero so that scoring proceeds without runtime failure.

---

## 11. Runtime Environment

**Training.** Google Colab and Kaggle notebook environments; Python 3, scikit-learn for ML models, TensorFlow/Keras for DL models. Training times reported in the benchmark table are wall-clock seconds in these environments and will vary with allocated hardware. Their **relative ordering** — which is the basis for the deployment decision — is stable across runs.

**Backend.** Flask with Flask-SQLAlchemy over PostgreSQL. Passwords are stored as hashes via `generate_password_hash` and verified with `check_password_hash`; plaintext credentials are never persisted.

**IoT node.** Raspberry Pi 3 Model B+ with Camera Module v2.1 over CSI, configured at 640 × 480. This resolution balances stream stability against the processing headroom available on the device. The most recent frame is held in a shared memory structure guarded by synchronisation primitives, permitting concurrent streaming and recording without frame-tearing or data races.

---

## 12. Known Limitations

- **Single-deployment validation.** The system was validated on one Raspberry Pi node. Multi-node and heterogeneous-device replication remains future work.
- **Honeypot bias.** Honeypot-sourced traffic is disproportionately automated. Human-driven targeted intrusions are correspondingly underrepresented, which may affect generalisation to low-and-slow manual attacks.
- **Geographic concentration.** The behavioural baselines were constructed from a limited user population; broader deployment would require baseline recalibration.
- **Encrypted-tunnel visibility.** Traffic traversing VPNs is not currently inspected. Metadata-level analysis of encrypted tunnels is identified as future work.
