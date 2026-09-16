# CredGuard

## Credential-Based Intrusion Detection System Using Explainable AI for IoT

CredGuard is a credential-based intrusion detection system designed for IoT environments. The system analyzes authentication attempts using Machine Learning (ML), Deep Learning (DL), and Explainable Artificial Intelligence (XAI) techniques to detect potentially malicious access attempts and support secure access control.

The system architecture consists of an IoT device, a backend server, a web application, and a PostgreSQL database. A Raspberry Pi connected to a Camera Module v2.1 is used as the IoT device. Authentication-related information, including IP address, username, password, and timestamp, is collected and transmitted to the backend server for processing and analysis.

The backend performs preprocessing and feature extraction and uses pre-trained ML and DL models to analyze authentication attempts. XAI techniques are used to improve the interpretability of model decisions. Based on the analysis results, an access-control decision is generated and communicated to the IoT device and the web application.

---

## System Architecture

The main components of CredGuard are:

- Raspberry Pi and Camera Module v2.1 as the IoT device.
- Backend server for data processing, feature extraction, model inference, and system logic.
- PostgreSQL database for storing user information, authentication attempts, model results, security actions, and related system data.
- Web application interfaces for the Admin and Device Owner roles.
- Machine Learning and Deep Learning models for authentication analysis.
- XAI techniques, including SHAP and LIME, for interpreting model predictions.

The system supports two primary user roles:

- **Admin:** monitors system activity, users, model performance, and security-related information.
- **Device Owner:** interacts with the system through the web application and uses the protected IoT device.

---

## CredGuard IoT Dataset

CredGuard IoT is the integrated dataset developed for the project.

The dataset was constructed by integrating five authentication- and attack-related datasets:

1. SSH Honeypot Logs
2. Medium-Interaction SSH Honeypot
3. Synthetic Web Authentication Logs
4. RBA Authentication
5. Brute-Force Attempt

The integrated dataset was developed to provide authentication-related data suitable for credential-based intrusion detection.

The dataset contains authentication and contextual features such as:

- Timestamp
- Source IP address
- Username
- Password
- ASN
- Location
- Authentication status
- Label
- Source
- IP type
- Inter-arrival time (IAT)
- Login attempts
- Failed attempts
- Failure ratio
- Mean IAT
- Standard deviation of IAT
- Password length
- Password complexity
- Day of week
- Hour
- Cyclic time features

IP information is enriched using ASN and geographic location information derived from the source IP address.

---

## Feature Engineering

Temporal and behavioral features are generated from authentication activity.

The feature-engineering process includes inter-arrival time, cumulative mean and standard deviation of inter-arrival time, login-attempt progression, and failure ratio.

Expanding-window calculations are used for temporal features to avoid using future information when constructing features for earlier authentication attempts.

---

## Machine Learning and Deep Learning Models

CredGuard evaluates multiple Machine Learning and Deep Learning models for authentication-based intrusion detection.

The evaluated models are:

- XGBoost
- Random Forest
- Support Vector Machine (SVM)
- Convolutional Neural Network (CNN)
- Recurrent Neural Network (RNN)
- Deep Neural Network (DNN)

The dataset is divided into training, validation, and test sets using a 70/15/15 split.

The attack class is treated as the positive class, while normal authentication activity is treated as the negative class.

### Model Performance

| Model | Accuracy | Precision | Recall | F1-Score | Training Time (s) |
|---|---:|---:|---:|---:|---:|
| XGBoost | 82.29% | 85.06% | 75.45% | 79.97% | 82 |
| Random Forest | 82.03% | 86.90% | 76.02% | 81.10% | 585 |
| SVM | 71.21% | 75.10% | 64.62% | 69.47% | 54 |
| CNN | 80.50% | 77.55% | 86.60% | 81.82% | 2640 |
| RNN | 80.94% | 79.04% | 84.90% | 81.87% | 3794 |
| DNN | 80.22% | 78.07% | 84.79% | 81.29% | 2857 |

Based on the reported evaluation, XGBoost was selected as the final model considering the balance between predictive performance and computational efficiency.

---

## Explainable AI

CredGuard incorporates Explainable AI techniques to improve the interpretability of model predictions.

Two XAI methods are used:

- **SHAP (SHapley Additive exPlanations):** used to analyze feature contributions to model predictions and global feature importance.
- **LIME (Local Interpretable Model-Agnostic Explanations):** used to provide local explanations for individual predictions.

These techniques are applied to help interpret why an authentication attempt is classified as normal or potentially malicious.

---

## Post-Login User Behaviour Analysis

In addition to authentication-based detection, CredGuard includes post-login user behaviour analysis.

The system monitors user behaviour after authentication and uses historical behaviour when sufficient data is available. Rule-based heuristics are also used when historical data is not sufficient.

The system calculates a risk score from 0 to 100 and categorizes the resulting risk as:

- Low Risk
- Medium Risk
- High Risk

The post-login analysis considers behavioural indicators such as session duration, live-stream watching activity, camera usage, camera interaction frequency, and full-screen usage.

High-risk behaviour may result in the session being terminated or blocked according to the system's decision process.

---

## IoT Integration

The IoT component consists of a Raspberry Pi connected to a Camera Module v2.1.

The Raspberry Pi collects authentication-related information and communicates with the backend server. The backend processes the received information and generates an access-control decision.

The resulting decision is transmitted to the Raspberry Pi and the web application, allowing the decision to be enforced on the IoT device and reflected in the web interface.

---

## Database

PostgreSQL is used as the project's database.

The database stores information related to:

- User accounts
- Login attempts
- Extracted features
- Model results
- Security actions
- IoT data
- Post-login behaviour data
- Security logs

Flask is used for the backend and application logic, with Flask-SQLAlchemy used for database interaction.

---

## Technologies

The project uses the following technologies and tools:

- Python
- Flask
- PostgreSQL
- Flask-SQLAlchemy
- Scikit-learn
- TensorFlow / Keras
- XGBoost
- SHAP
- LIME
- Raspberry Pi
- Raspberry Pi Camera Module v2.1
- Jupyter
- Google Colab
- Kaggle

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

## License

This project is distributed under the MIT License.
