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
1. Create a bucket and upload the Lambda zip to it -> `heart-beat-lambda-deployment` (leave all settings at default)
2. In the Swagger file find ID `216698350696` and replace it with your own
3. Create a bucket for the REST API and upload the Swagger to it -> `heart-beat-rest-api` (leave all settings at default)
4. Create a Cognito user pool ->
   - Traditional web application
   - Application name -> `HeartBeat`
   - Sign-in identifiers options -> `email`
   - Required sign-in attributes -> `email`
   - In Cognito -> App client -> Edit -> enable all authentication flows -> save changes
5. Replace the pool ID you created in Cognito in `CognitoTemplate` (`UserPoolId`) and replace the `RoleArn` by putting the one you will use for creating the stack in CloudFormation
6. Create a stack with the new resources from the `CognitoTemplate` in CloudFormation
   > **Note:** for CloudFormation -> Template source: upload a template file -> submit
7. If you want to test the admin features you need to manually add a user to Cognito and assign them the Admin role -> do this via the app client in Cognito and click “View sign-in page” so that the user is already confirmed
8. Create a stack with the new resources from the `BatchBucketTemplate` in CloudFormation
9. Create a stack with the new resources from the `RestAPITemplate` in CloudFormation
10. In `HeartBeatTemplate` change ->
    - ARN of the role with which you want to instantiate the services
    - Find ID `216698350696` and replace it with your own
    - Put the new pool ID, client ID, and client secret in the Lambda environment variables
    - Insert the ARN of the API you created
    - For the functions `checkPatientGroupAuth`, `checkDoctorGroupAuth`, and `checkAdminGroupAuth` in `SourceArn: "arn:aws:execute-api:us-east-1:216698350696:3we4sfdvrh/authorizers/vy1o5f"` replace `216698350696` with your own ID, `3we4sfdvrh` with the API ID, and `vy1o5f` with the authorizer ID for the specific authorizer
11. Create a stack with the new resources from the `HeartBeatTemplate` in CloudFormation
12. Manually add an S3 trigger for the `splitBatchData` Lambda function ->
    - On `splitBatchData` click the “Add trigger” button
    - Select `S3`
    - Select bucket `batchdataheartbeatcopy`
    - Select event type `PUT`
    - Accept using the S3 bucket for both input and output
13. Modify the HTTP calls to API Gateway in the HTML files
14. Put the HTML files into the frontend bucket, select them all, and via Actions make all pages public using ACL

## 💻 Run the Code
1. If you want to test the `streamingMQTT.py` script:
   - Generate and download the certificates for the already instantiated thing
   - Rename the certificates as follows:
     - device certificate -> `device-cert.pem`
     - private key file -> `device-key.pem`
     - Amazon Root CA 1 certificate -> `root-CA.pem`
   - Attach the `IoTSimulator` policy to the certificate
   - Modify the endpoint in the script to the one created in IoT Core
   - Run the script from a directory containing the certificates as well
2. If you want to test the `batchMQTT.py` script, run it inside CloudShell

> **Note:** for the scripts data to be processed there must be an entry in the `IoTPatient` DynamoDB table with `deviceId=IoTSimulator`

---
*Developed by Greta Luna Ancora & Nicola Violante as an academic project at the Department of Computer Engineering, University of Rome Tor Vergata.*
