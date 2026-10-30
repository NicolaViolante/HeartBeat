# Serverless Heart Rate Stream & Batch Processing

<p align="left">
  <img src="https://img.shields.io/badge/AWS-232F3E?style=for-the-badge&logo=amazon-aws&logoColor=white" />
  <img src="https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white" />
  <img src="https://img.shields.io/badge/JavaScript-F7DF1E?style=for-the-badge&logo=javascript&logoColor=black" />
  <img src="https://img.shields.io/badge/HTML5-E34F26?style=for-the-badge&logo=html5&logoColor=white" />
</p>

## 📝 Overview
This repository contains a fully **serverless telemedicine and health-monitoring platform** built on Amazon Web Services (AWS). The system is designed to ingest, process, and analyze heart rate (BPM) telemetry acquired from wearable IoT devices, while providing a web application for managing Electronic Health Records (EHR), medical staff, and patients.

The primary objective of the architecture is twofold:
1. **Near Real-Time Stream Processing:** Continuously ingest and analyze live BPM streams from connected wearables to detect cardiac anomalies and immediately alert the assigned physician.
2. **Offline Batch Processing:** Handle intermittent connectivity by allowing wearable devices to buffer telemetry locally in JSON files and automatically process them in batch mode once network access is restored.

## ⚙️ System Architecture & Data Pipelines
The platform orchestrates multiple managed AWS services across three main operational flows:

- **Role-Based Access Control (RBAC) & Web Portal:** 
  - Hosted as a static HTML/JS web application on **Amazon S3** (`frontendheartbeat`) and exposed via **Amazon API Gateway** using Lambda Proxy and CORS integrations.
  - Authentication is managed by **Amazon Cognito** using OpenID Connect (OIDC) and JSON Web Tokens (JWT). Three user groups (**Admin**, **Doctor**, **Patient**) are enforced via custom **Lambda Authorizers** (`checkAdminGroupAuth`, `checkDoctorGroupAuth`, `checkPatientGroupAuth`) that validate bearer tokens and return dynamic IAM policies:
    - **Admins** onboard new physicians (`addDoctor`).
    - **Doctors** register patients (`addPatient`), inspect automated BPM analysis results (`readAnalysisResults`), and read/write clinical annotations (`readClinicalRecords`, `writeClinicalRecords`).
    - **Patients** bind wearable devices to their profile (`addDevice`) and view their own medical history (`patientReadClinicalRecords`).
- **Streaming Pipeline (Near Real-Time):**
  - Wearable devices (registered as IoT Things with X.509 certificates and custom IoT Policies) publish BPM data over **MQTT** to **AWS IoT Core** on the `IoTSimulator/bpm` topic.
  - An IoT Core routing rule invokes `sendToSQS` to forward incoming payloads into an **Amazon SQS FIFO** queue (`bpmValues.fifo`).
  - The queue triggers `sendToStepFunction`, which fetches up to 10 messages per device, enriches the payload with patient demographics (age and gender) from **Amazon DynamoDB**, and starts an execution of the `AnalyzeStateMachine` workflow.
- **Batch Pipeline (Offline Recovery):**
  - When a device goes offline, it buffers BPM readings locally into a JSON file and publishes it via MQTT upon reconnection to upload it into the `batchdataheartbeat` **Amazon S3** bucket.
  - The S3 object-creation event triggers `splitBatchData`, which parses the JSON batch into individual records, resolves patient demographics by `deviceId`, and triggers an `AnalyzeStateMachine` execution for each record.

## 🔬 Technical Highlights & Design Choices
- **Step Functions Workflow Orchestration (`AnalyzeStateMachine`):**
  - **Feature Extraction & Clinical Evaluation:** The workflow first aggregates incoming readings to compute the average BPM (`featuresExtraction`), then evaluates it against age- and gender-specific clinical thresholds (`analyze`) to classify the state into *Severe Bradycardia*, *Moderate Bradycardia*, *No Anomaly*, *Moderate Tachycardia*, or *Severe Tachycardia*.
  - **Conditional Branching & Parallel Execution:** A `Choice` state routes normal results directly to DynamoDB. If an anomaly is detected, a `Parallel` state simultaneously persists the analysis in `AnalysisResults` and invokes `patientToDoctor` to look up the assigned physician's email and dispatch an alert.
- **Filtered Medical Notifications (Amazon SNS):** Physicians are subscribed to the `AnomalyNotification` SNS topic upon registration. Each subscription is configured with a **Filter Policy** matching the `doctor_id` message attribute, ensuring doctors only receive real-time email alerts for their own patients.
- **High-Throughput FIFO & Partial Batch Failure Handling (Amazon SQS):** 
  - The `bpmValues.fifo` queue guarantees chronological ordering and 5-minute deduplication via `MessageDeduplicationId`, with **High-Throughput FIFO** enabled to maximize ingestion rates.
  - The SQS trigger on `sendToStepFunction` enables **`ReportBatchItemFailures`**, ensuring that if an error occurs while processing a batch, only the failed messages become visible again in the queue—preventing duplicate processing and preserving Lambda polling concurrency.
- **Single-Digit Millisecond NoSQL Persistence (Amazon DynamoDB):** Uses 5 on-demand DynamoDB tables designed around access patterns: `AnalysisResults` (`patientID` + `timestamp`), `ClinicalRecords` (`patient` + `timestamp`), `DoctorPatient` (`patientId` + `doctorId`), `IoTPatient` (`deviceID`), and `Patient` (`patientId`).
- **Modular Infrastructure as Code (AWS CloudFormation):** The infrastructure is partitioned into 4 YAML templates (`CognitoTemplate`, `RestAPITemplate`, `BatchBucketTemplate`, `HeartBeatTemplate`) to prevent circular dependencies and automate end-to-end deployment, while **Amazon CloudWatch** centralizes logs for observability and debugging.

## 🚀 Create the AWS Environment
1. Launch the CloudFormation `CognitoTemplate.yaml` and wait until the resources are created
2. Create an S3 Bucket and upload inside it the file `HeartBeatAPI-dev-swagger-apigateway.json`
3. Launch the CloudFormation `RestAPITemplate.yaml` (give the S3 bucket's name of the previous step as parameter)
4. Take the invoke URL of the API Gateway just created and copy it inside the `apigateway.js` file (inside the `frontend` folder)
5. Zip the modified `frontend` folder
6. Launch the CloudFormation `BatchBucketTemplate.yaml`
7. Launch the CloudFormation `HeartBeatTemplate.yaml`

## 💻 Run the Code
1. Start the application by opening the static website endpoint of the S3 bucket created for the frontend
2. Create an admin user from the Cognito console
3. Login as Admin and add a Doctor
4. Login as Doctor and add a Patient
5. Take the auto-generated patient's deviceID from DynamoDB and insert it in the `streamingMQTT.py` and `batchMQTT.py` files
6. To test the stream processing, run the `streamingMQTT.py` file
7. To test the batch processing, run the `batchMQTT.py` file
8. Login as Patient and read your clinical records
9. Login as Doctor to see the analysis results and to write a clinical record for the Patient

---
*Developed by Greta Luna Ancora & Nicola Violante as an academic project at the Department of Computer Engineering, University of Rome Tor Vergata.*
