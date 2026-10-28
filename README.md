# Serverless Heart Rate Streaming And Batch Processing Architecture

This repository contains the final project for the course of Distributed Systems and Cloud Computing of the University of Rome Tor Vergata (faculty Computer Engineering).

---

## 📌 Executive Summary

HeartBeat is a serverless architecture for collecting and processing heart-rate measurements through two ingestion paths: continuous MQTT streaming and S3-based batch uploads. The project combines managed AWS services to ingest simulated BPM data, process it with Lambda functions, expose application functionality through API Gateway, and support user authentication through Amazon Cognito.

The repository includes the front-end assets, AWS deployment templates, an API Gateway Swagger definition, and two Python scripts used to simulate device data:

1. **Streaming ingestion:** `streamingMQTT.py` generates simulated heart-rate values and publishes them to AWS IoT Core over MQTT using mutual TLS certificates.
2. **Batch ingestion:** `batchMQTT.py` generates a batch of timestamped heart-rate records and uploads the resulting JSON payload to an S3 bucket.
3. **Serverless processing:** AWS Lambda functions process incoming data and support application and authorization workflows.
4. **API and identity management:** API Gateway exposes the REST API, while Cognito provides user-pool authentication and group-based authorization checks.
5. **Data storage and presentation:** S3 stores deployment assets and batch objects, DynamoDB contains patient/device records, and the front-end files provide the web interface.

This repository is an academic project and should be treated as a reference implementation rather than a production-ready medical monitoring system.

---

## 🎯 Technical Scope & Key Findings

### 1. Streaming Data Path

- `streamingMQTT.py` connects to AWS IoT Core using a device certificate, a private key, and the Amazon Root CA certificate.
- The simulator uses the client ID `IoTSimulator` and publishes to the `IoTSimulator/bpm` MQTT topic.
- Each payload includes a device identifier, a millisecond timestamp, and five generated BPM values.
- Messages are published with MQTT QoS 1, and the script continues publishing at five-second intervals until interrupted.

### 2. Batch Data Path

- `batchMQTT.py` generates ten timestamped records, each containing five BPM measurements.
- The script serializes the data into JSON and uploads it to the configured S3 bucket.
- The AWS deployment instructions configure an S3 `PUT` trigger for the `splitBatchData` Lambda function so uploaded batch data can initiate processing.
- The processing flow expects a corresponding device entry in the `IoTPatient` DynamoDB table for `deviceId=IoTSimulator`.

### 3. API and Authentication

- `HeartBeatAPI-dev-swagger-apigateway.json` describes the API Gateway REST API.
- Amazon Cognito supplies the user pool and app-client configuration used for sign-in.
- The deployment templates reference authorization functions named `checkPatientGroupAuth`, `checkDoctorGroupAuth`, and `checkAdminGroupAuth`.
- The API, Cognito resources, Lambda configuration, and authorizer identifiers must refer to the resources created in the deployer's own AWS account.

### 4. AWS Service Responsibilities

| Component | Responsibility in the project |
| --- | --- |
| **AWS IoT Core** | Receives device-simulator messages over MQTT using certificate-based authentication. |
| **AWS Lambda** | Runs serverless application, authorization, and data-processing functions. |
| **Amazon S3** | Stores deployment assets, the Swagger definition, front-end files, and uploaded batch payloads. |
| **Amazon API Gateway** | Exposes the REST API used by the front end. |
| **Amazon Cognito** | Manages user-pool authentication and supports group-based authorization. |
| **Amazon DynamoDB** | Stores patient/device records, including the `IoTPatient` table referenced by the scripts and deployment notes. |

---

## 🗂️ Repository Contents

| Path | Description |
| --- | --- |
| `frontend/` | Front-end assets for the web application. |
| `lambdazip/` | Lambda deployment package assets. |
| `template aws/` | AWS CloudFormation templates used to create the application's resources. |
| `HeartBeatAPI-dev-swagger-apigateway.json` | Swagger/API definition for API Gateway. |
| `deploymentAWS.txt` | Additional AWS deployment notes. |
| `streamingMQTT.py` | Simulates live heart-rate data and publishes it to AWS IoT Core. |
| `batchMQTT.py` | Generates batch heart-rate data and uploads it to S3. |
| `ReportHeartBeat.pdf` | Project report. |
| `SlidesHeartBeat.pptx` | Project presentation. |

---

To execute the project follow the next steps:

## Create the AWS Environment
1. Create a bucket and upload the Lambda zip to it -> heart-beat-lambda-deployment (leave all settings at default)

2. In the Swagger file find ID 216698350696 and replace it with your own

3. Create a bucket for the REST API and upload the Swagger to it -> heart-beat-rest-api (leave all settings at default)

4. Create a Cognito user pool ->

    Traditional web application

    Application name -> HeartBeat

    Sign-in identifiers options -> email

    Required sign-in attributes -> email

5. In Cognito -> App client -> Edit -> enable all authentication flows -> save changes

6. Replace the pool ID you created in Cognito in CognitoTemplate (UserPoolId) and replace the RoleArn by putting the one you will use for creating the stack in CloudFormation

7. Create a stack with the new resources from the CognitoTemplate in CloudFormation

    Note: for CloudFormation -> Template source: upload a template file -> submit

8. If you want to test the admin features you need to manually add a user to Cognito and assign them the Admin role -> do this via the app client in Cognito and click “View sign-in page” so that the user is already confirmed

9. Create a stack with the new resources from the BatchBucketTemplate in CloudFormation

10. Create a stack with the new resources from the RestAPITemplate in CloudFormation

11. In HeartBeatTemplate change ->

    ARN of the role with which you want to instantiate the services

    Find ID 216698350696 and replace it with your own

    Put the new pool ID, client ID, and client secret in the Lambda environment variables

    Insert the ARN of the API you created

    For the functions checkPatientGroupAuth, checkDoctorGroupAuth, and checkAdminGroupAuth in SourceArn:
    "arn:aws:execute-api:us-east-1:216698350696:3we4sfdvrh/authorizers/vy1o5f"
    replace 216698350696 with your own ID, 3we4sfdvrh with the API ID, and vy1o5f with the authorizer ID for the specific authorizer

12. Create a stack with the new resources from the HeartBeatTemplate in CloudFormation

13. Manually add an S3 trigger for the splitBatchData Lambda function ->

    On splitBatchData click the “Add trigger” button

    Select S3

    Select bucket batchdataheartbeatcopy

    Select event type PUT

    Accept using the S3 bucket for both input and output

14. Modify the HTTP calls to API Gateway in the HTML files

15. Put the HTML files into the frontend bucket, select them all, and via Actions make all pages public using ACL

## Run the code
1. If you want to test the streamingMQTT.py script

    Generate and download the certificates for the already instantiated thing

    Rename the certificates as follows: device certificate -> device-cert.pem, private key file -> device-key.pem, Amazon Root CA 1 certificate -> root-CA.pem

    Attach the IoTSimulator policy to the certificate

    Modify the endpoint in the script to the one created in IoT Core

    Run the script from a directory containing the certificates as well

2. If you want to test the batchMQTT.py script run it inside CloudShell

Note: for the scripts data to be processed there must be an entry in the IoTPatient DynamoDB table with deviceId=IoTSimulator
