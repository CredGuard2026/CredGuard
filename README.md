# CredGuard

## Credential-Based Intrusion Detection System Using Explainable AI (XAI) for IoT Environments

CredGuard is a credential-based intrusion detection system designed for IoT environments. The system analyzes authentication attempts using Machine Learning (ML), Deep Learning (DL), and Explainable Artificial Intelligence (XAI) techniques to detect potentially malicious access attempts and support secure access control.

The system consists of an IoT device, a backend server, a PostgreSQL database, and a web-based interface. A Raspberry Pi connected to a Camera Module v2.1 represents the protected IoT resource. When a user attempts to access the camera stream, the system collects the username, password, source IP address, and timestamp and transmits them to the backend server for processing and analysis.

<img src="Figures/CredGuard_Architecture.JPG" alt="CredGuard Architecture" width="800">

---

## System Architecture

The main components of CredGuard include:

- **IoT Device:** A Raspberry Pi connected to a Camera Module v2.1, representing the protected IoT resource.
- **Backend Server:** Receives each login attempt, extracts the required authentication attributes, applies the trained detection model, generates explanation outputs, stores the results, and returns the corresponding access-control decision.
- **Machine Learning and Deep Learning Models:** Used to classify authentication attempts as Normal or Attack.
- **Explainable AI:** SHAP and LIME are used to interpret model predictions.
- **PostgreSQL Database:** Stores user information, login attempts, model results, security actions, IoT-related data, and post-login behaviour records.
- **Web Application:** Provides the web-based authentication interface for users and the administrative dashboard for the Administrator.

The system supports two primary user roles:

- **Admin:** Monitors system activity, users, model results, and security-related information.
- **Device Owner:** Initiates authentication and interacts with the protected IoT resource through the web application.

---

## CredGuard-IoT Dataset

CredGuard-IoT is the integrated dataset developed for the project.

The dataset was constructed by integrating five authentication- and intrusion-related data sources:

1. SSH Honeypot Logs
2. Medium-Interaction SSH Honeypot (Cowrie)
3. Web-Based Authentication Intrusion Data
4. Risk-Based Authentication (RBA)
5. SSH Brute-Force Attempt Dataset

The integrated dataset provides authentication-related information used for credential-based intrusion detection.

The dataset includes features such as:

- Timestamp
- Source IP address
- Username
- Password
- ASN
- Location
- Authentication status
- Source
- Label
- Inter-arrival time
- Login attempts
- Failed attempts
- Failure ratio
- Mean inter-arrival time
- Standard deviation of inter-arrival time
- Password length
- Password complexity
- Day of week
- Hour
- Cyclic time features

ASN and geographic location information are obtained from the source IP address using the IPinfo service.

---

## Data Processing and Feature Engineering

The data processing pipeline includes schema alignment, integration of heterogeneous sources, status normalization, duplicate removal, critical-field filtering, credential completion, network enrichment, and temporal standardization.

Temporal and behavioural features are generated from ordered authentication attempts.

The feature-engineering process includes:

- Inter-arrival time (IAT)
- Mean and standard deviation of inter-arrival time
- Login-attempt progression
- Failure ratio
- Password length
- Password complexity
- Cyclic hour encoding using sine and cosine transformations
- ASN and geographic location

Temporal features are calculated using historical information associated with the source IP to avoid using future information during feature construction.

---

## Implementation Environment

The implementation was carried out in Python 3.14.

- **Machine Learning models:** implemented using scikit-learn and XGBoost.
- **Deep Learning models:** implemented using TensorFlow and Keras.
- **Explainability:** implemented using SHAP and LIME.
- **Data handling and numerical processing:** pandas and NumPy.

---

## Machine Learning and Deep Learning Models

CredGuard evaluates multiple Machine Learning and Deep Learning models for authentication-based intrusion detection.

The evaluated models are:

- Random Forest (RF)
- XGBoost
- Support Vector Machine (SVM)
- Convolutional Neural Network (CNN)
- Recurrent Neural Network (RNN)
- Deep Neural Network (DNN)

### Data Splitting

To prevent information leakage caused by the same IP address appearing in both the training and testing partitions, the dataset was divided into training, validation, and testing subsets (intended ratio **70% / 15% / 15%**) using `GroupShuffleSplit` with the **IP address as the grouping variable**. All records of a given IP address are therefore assigned to only one subset.

- **Training set:** used to fit the selected ML and DL models.
- **Validation set:** used for model tuning and early stopping where applicable.
- **Testing set:** reserved for the final performance evaluation.

This grouped split provides a more realistic estimate of the models' generalization to unseen sources. Because a small number of IP addresses account for a large share of the records, the resulting row proportions differ from the intended ratio.

`Attack` is treated as the positive class and `Normal` as the negative class.

### Model Performance

| Model | Accuracy | Precision | Recall | F1-Score | Training Time (s) |
|---|---:|---:|---:|---:|---:|
| Random Forest | 82.00% | 79.79% | 78.44% | 79.11% | 868 |
| XGBoost | 81.31% | 79.16% | 77.34% | 78.24% | 287 |
| SVM | 71.53% | 67.05% | 67.77% | 67.41% | 326 |
| CNN | 77.85% | 74.91% | 73.72% | 74.31% | 3591 |
| RNN | 76.35% | 73.80% | 70.64% | 72.19% | 3203 |
| DNN | 77.40% | 73.77% | 74.44% | 74.11% | 533 |

Random Forest achieved the highest accuracy, precision, recall, and F1-score among all evaluated models, and was therefore selected as the final model. XGBoost required substantially less training time (287 s compared with 868 s).

An LSTM model was also implemented but excluded from the reported model comparison because of its sequential processing requirements and longer training time.

---

## Explainable AI

CredGuard incorporates Explainable AI techniques to improve the interpretability of model predictions.

Two XAI techniques are used:

- **SHAP (SHapley Additive exPlanations):** Used to analyze feature contributions and global feature importance.
- **LIME (Local Interpretable Model-Agnostic Explanations):** Used to provide local explanations for individual predictions.

Both techniques were integrated to interpret the predictions of the selected Random Forest model. They provide interpretable information about model classification results and support analysis of why an authentication attempt is classified as Normal or Attack.

**Key findings:**

- The Random Forest feature-importance analysis emphasized failure rate, inter-arrival time, and failed attempts.
- The global SHAP analysis highlighted username frequency, failure rate, and location frequency as the most influential features.
- Network-access patterns (location and ASN frequency) contributed in both analyses, while password-related features had limited influence.
- Local LIME explanations clarify feature contributions for individual authentication decisions.

### SHAP Analysis

<img src="Figures/Global_SHAP.jpg" alt="Global SHAP Feature Importance" width="600">

<img src="Figures/SHAP_Attack.png" alt="SHAP Attack Class" width="600">

<img src="Figures/SHAP_Normal.png" alt="SHAP Normal Class" width="600">

### LIME Analysis

<img src="Figures/LIME_Attack.jpg" alt="LIME Attack Explanation" width="600">

<img src="Figures/LIME_Normal.jpg" alt="LIME Normal Explanation" width="600">

---

## Post-Login User Behaviour Analysis

In addition to authentication-based detection, CredGuard performs post-login user behaviour analysis.

After successful authentication, the system continuously monitors user behaviour during the session. The analysis considers activities such as session duration, live-stream viewing, camera interactions, and full-screen usage.

When sufficient historical data are available, the system constructs a behavioural baseline for the user and compares the current session with the user's previous behaviour. When historical data are insufficient, rule-based heuristics are used to estimate the risk level.

The system computes a Risk Score ranging from 0 to 100 and classifies the session into:

- **Low Risk:** Normal behaviour
- **Medium Risk:** Suspicious behaviour
- **High Risk:** Abnormal and high-risk behaviour

High-risk sessions may be terminated or blocked according to the system's decision process.

---

## IoT Integration

The IoT component consists of a Raspberry Pi 3 Model B+ connected to a Raspberry Pi Camera Module v2.1.

The prototype testbed consists of a Raspberry Pi device connected to a camera module, a web-based authentication interface, a Flask backend server, a PostgreSQL database, and an administrative dashboard. The Raspberry Pi camera represents the protected IoT resource, while the web interface allows users to submit authentication requests.

The backend server receives each login attempt, extracts the required authentication attributes, applies the trained detection model, generates explanation outputs, stores the results, and returns the corresponding access-control decision.

When a user attempts to access the camera stream, the system collects the username, password, source IP address, and timestamp. These attributes are transmitted to the backend server, where they are transformed into the selected feature representation and classified as either normal or suspicious. If the login attempt is valid and classified as normal, access to the camera stream is granted. Otherwise, access is denied and the event is recorded for administrator review.

<img src="Figures/CredGuard_Testbed.png" alt="CredGuard IoT Testbed" width="800">

The IoT implementation also supports live video streaming, image snapshot capture, and video recording through the Raspberry Pi Camera Module.

---

## Database

PostgreSQL is used as the project's database and is integrated with the Flask backend through Flask-SQLAlchemy.

The database stores information related to:

- User accounts
- Login attempts
- Model predictions
- Security actions
- IoT device data
- Post-login behaviour
- User activity records
- Behaviour-related explanations

Passwords are not stored in plaintext. Password hashing and verification are used for user authentication.

---

## Technologies

The project uses the following technologies and tools:

- Python 3.14
- pandas
- NumPy
- Scikit-learn
- TensorFlow / Keras
- XGBoost
- SHAP
- LIME
- Flask
- Flask-SQLAlchemy
- PostgreSQL
- Jupyter
- Google Colab
- Kaggle
- Raspberry Pi
- Raspberry Pi Camera Module v2.1

---

## Project Team

CredGuard was developed by:

- Renad Sultan Alamri
- Shaima Shedaid Alharbi
- Kadi Nawaf Alotaibi
- Aisha Abdulrahman Alranini
- Alaa Hani Alraddadi
- Rahaf Ahmed Alkhulaifi

**Supervisor:** Prof. Fatemah Mordhi Alharbi

---

## Repository Contents

The repository contains the project documentation, dataset, model development notebooks, figures, and supporting files.

- `Data/` — CredGuard-IoT dataset and related data files
- `Figures/` — Project figures and visual materials
- `Scripts/` — Model development and training notebooks
- `README.md` — Project overview and documentation
- `REPRODUCIBILITY.md` — Reproducibility details
- `requirements.txt` — Python package dependencies
- `CITATION.cff` — Citation metadata
- `LICENSE` — Project license

---

## License

This project is distributed under the MIT License.
