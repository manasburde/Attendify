# Attendify 🎓

> A serverless, cloud-native smart attendance system that uses **AWS Facial Recognition** to automate and manage student attendance in universities.

---

## 📌 Table of Contents

- [Overview](#-overview)
- [Features](#-features)
- [Architecture](#-architecture)
- [Tech Stack](#-tech-stack)
- [AWS Services Used](#-aws-services-used)
- [Project Structure](#-project-structure)
- [Setup & Deployment](#-setup--deployment)
  - [Prerequisites](#prerequisites)
  - [1. S3 — Student Photo Storage](#1-s3--student-photo-storage)
  - [2. AWS Rekognition — Face Collection](#2-aws-rekognition--face-collection)
  - [3. DynamoDB — Database Tables](#3-dynamodb--database-tables)
  - [4. AWS Cognito — Authentication](#4-aws-cognito--authentication)
  - [5. Lambda Functions](#5-lambda-functions)
  - [6. API Gateway](#6-api-gateway)
  - [7. Frontend Configuration](#7-frontend-configuration)
- [Lambda Functions Reference](#-lambda-functions-reference)
- [Data Schema](#-data-schema)
- [Authentication Flow](#-authentication-flow)
- [Screenshots](#-screenshots)
- [Known Limitations](#-known-limitations)
- [License](#-license)

---

## 🧠 Overview

**Attendify** eliminates manual attendance processes by letting faculty trigger a real-time face scan directly from their browser. A webcam snapshot is sent to **AWS Rekognition**, which identifies the student, and attendance is automatically recorded in **DynamoDB** — all within seconds. Students can log in independently to view their own attendance records.

---

## ✨ Features

| Feature | Details |
|---|---|
| 🎥 Face Recognition Attendance | Webcam-based snapshot matched against an indexed face collection |
| 🧑‍🏫 Faculty Portal | Faculty can start sessions, view recognized students, and manually override attendance |
| 🎓 Student Portal | Students can view their own attendance records per subject and semester |
| 🔐 Secure Auth | Separate Cognito User Pools for students and faculty, both via Google Sign-In |
| ✅ Email Allowlist | Faculty access is restricted to a configurable list of authorized emails |
| ✏️ Manual Override | Faculty can mark individual students present or absent to correct errors |
| 📋 Attendance History | Historical attendance records stored per-session in DynamoDB |
| ☁️ Fully Serverless | Zero servers to manage — runs entirely on AWS Lambda + API Gateway |

---

## 🏗 Architecture

```
Browser (Faculty)
    │
    ├── Captures webcam image (base64)
    │
    └──► API Gateway (POST /mark-attendance)
              │
              └──► Lambda: FaceRecognitionAttendance
                        │
                        ├──► AWS Rekognition (SearchFacesByImage)
                        │         └── Collection: StudentFaces
                        │
                        └──► DynamoDB: StudentData
                                  └── Updates present/total counts

Browser (Student)
    │
    └──► API Gateway (GET /student-data)   [Cognito Authorizer]
              │
              └──► Lambda: getStudentData
                        │
                        └──► DynamoDB: StudentData (read by Cognito sub)

Faculty: Attendance History
    └──► API Gateway (GET /get-attendance?subject=&semester=)
              └──► Lambda: get-attendance
                        └──► DynamoDB: AttendanceHistory

Faculty: Manual Override
    └──► API Gateway (POST /modify-attendance)
              └──► Lambda: modify-attendance
                        └──► DynamoDB: StudentData (MARK_PRESENT / MARK_ABSENT)
```

---

## 🛠 Tech Stack

- **Frontend:** HTML5, Tailwind CSS, Vanilla JavaScript
- **Backend:** AWS Lambda (Node.js 20.x, ES Modules)
- **SDK:** AWS SDK v3 (`@aws-sdk/client-rekognition`, `@aws-sdk/client-dynamodb`, `@aws-sdk/lib-dynamodb`)
- **Auth:** AWS Cognito (PKCE flow for faculty; implicit token flow for students)
- **Database:** Amazon DynamoDB
- **Face Recognition:** Amazon Rekognition
- **Storage:** Amazon S3
- **API:** Amazon API Gateway (REST)

---

## ☁️ AWS Services Used

| Service | Purpose |
|---|---|
| **Amazon Rekognition** | Index student faces and match webcam snapshots |
| **Amazon S3** | Store student profile photos used for face indexing |
| **Amazon DynamoDB** | Store student records and attendance history |
| **AWS Lambda** | Run all backend business logic (serverless) |
| **Amazon API Gateway** | Expose Lambda functions as REST APIs |
| **AWS Cognito** | Handle Google OAuth2 login for both students and faculty |

---

## 📁 Project Structure

```
Attendify/
│
├── frontend/
│   ├── HomePage.html               # Landing page with portal selection
│   ├── Loginpage.html              # Student login (Cognito implicit flow)
│   ├── FacultyLogin.html           # Faculty login (Cognito PKCE + email allowlist)
│   ├── StudentPortal.html          # Student attendance dashboard
│   └── FacultyPortal.html          # Faculty attendance management panel
│
└── lambda/
    ├── FaceRecognitionAttendance/
    │   └── index.mjs               # Core face scan → attendance marking logic
    ├── get-attendance/
    │   └── index.mjs               # Fetch attendance history by subject & semester
    ├── getStudentData/
    │   └── index.mjs               # Fetch authenticated student's own data
    └── modify-attendance/
        └── index.mjs               # Faculty manual present/absent override
```

---

## 🚀 Setup & Deployment

### Prerequisites

- An **AWS account** with appropriate IAM permissions
- **AWS CLI** installed and configured (`aws configure`)
- A **Google Cloud project** with OAuth 2.0 credentials
- A static file server or hosting for the HTML frontend (e.g., `python -m http.server 8000`)

---

### 1. S3 — Student Photo Storage

Create an S3 bucket to store student profile photos:

```bash
aws s3 mb s3://attendifystudentphotos --region ap-south-1
```

Upload student photos following this naming convention:

```
photos/{studentCognitoSub}-profile.jpg
```

---

### 2. AWS Rekognition — Face Collection

Create the Rekognition face collection:

```bash
aws rekognition create-collection \
    --collection-id "StudentFaces" \
    --region ap-south-1
```

Index each student's photo into the collection. The `external-image-id` must match the student's **Cognito `sub`** (UUID), which is also used as their `StudentID` in DynamoDB:

```bash
aws rekognition index-faces \
    --collection-id "StudentFaces" \
    --image '{"S3Object":{"Bucket":"attendifystudentphotos","Name":"photos/{studentSub}-profile.jpg"}}' \
    --external-image-id "{studentSub}" \
    --region ap-south-1
```

> ⚠️ Repeat this step for every student. The `external-image-id` is the key that links a recognized face back to a DynamoDB record.

---

### 3. DynamoDB — Database Tables

**Table 1: `StudentData`**

| Attribute | Type | Notes |
|---|---|---|
| `StudentID` | String (PK) | Cognito `sub` UUID |
| `firstName` | String | |
| `lastName` | String | |
| `registrationNo` | String | University roll number |
| `attendance` | Map | Nested by semester and subject (see schema below) |

**Table 2: `AttendanceHistory`**

| Attribute | Type | Notes |
|---|---|---|
| `sessionId` | String (PK) | Unique session identifier |
| `subject` | String | |
| `semester` | Number | |
| `timestamp` | Number | Unix epoch |
| `studentsPresent` | List | Array of student IDs recognized |

---

### 4. AWS Cognito — Authentication

Create **two separate User Pools**:

#### Student User Pool
- Enable **Google** as a federated identity provider
- Set **App client** callback URL: `http://localhost:8000/Loginpage.html`
- Use `response_type=token` (implicit flow)
- Note the `CLIENT_ID` and update `Loginpage.html`

#### Faculty User Pool
- Enable **Google** as a federated identity provider
- Set **App client** callback URL: `http://localhost:8000/FacultyLogin.html`
- Use `response_type=code` with **PKCE** (authorization code flow)
- Note the `CLIENT_ID` and update `FacultyLogin.html`
- Update the `authorizedEmails` array in `FacultyLogin.html` to restrict access

---

### 5. Lambda Functions

Deploy each function from its respective directory. All functions use **Node.js 20.x** runtime with ES Module format (`index.mjs`).

```bash
# Example: Package and deploy FaceRecognitionAttendance
cd lambda/FaceRecognitionAttendance
zip -r function.zip index.mjs

aws lambda create-function \
    --function-name FaceRecognitionAttendance \
    --runtime nodejs20.x \
    --handler index.handler \
    --zip-file fileb://function.zip \
    --role arn:aws:iam::{account-id}:role/{your-lambda-execution-role} \
    --region ap-south-1
```

Repeat for `get-attendance`, `getStudentData`, and `modify-attendance`.

**Required IAM permissions for the Lambda execution role:**

```json
{
    "Effect": "Allow",
    "Action": [
        "rekognition:SearchFacesByImage",
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:Scan"
    ],
    "Resource": "*"
}
```

---

### 6. API Gateway

Create a REST API and wire up the following routes to their Lambda functions:

| Method | Path | Lambda | Authorizer |
|---|---|---|---|
| `POST` | `/mark-attendance` | `FaceRecognitionAttendance` | None |
| `GET` | `/get-attendance` | `get-attendance` | None |
| `GET` | `/student-data` | `getStudentData` | Cognito (Student Pool) |
| `POST` | `/modify-attendance` | `modify-attendance` | None |

Enable **CORS** on all routes. Deploy the API and note the **Invoke URL**.

---

### 7. Frontend Configuration

Update the following constants in each HTML file before serving:

**`FacultyLogin.html` & `Loginpage.html`:**
```javascript
const COGNITO_DOMAIN = 'https://{your-cognito-domain}.auth.ap-south-1.amazoncognito.com';
const CLIENT_ID = '{your-app-client-id}';
const REDIRECT_URI = 'http://localhost:8000/{LoginPage}.html';
```

**`FacultyLogin.html` — Email Allowlist:**
```javascript
const authorizedEmails = [
    'admin@youruniversity.edu',
    'faculty@youruniversity.edu'
];
```

**`FacultyPortal.html` & `StudentPortal.html`:**
```javascript
const API_BASE_URL = 'https://{your-api-id}.execute-api.ap-south-1.amazonaws.com/prod';
```

Serve the frontend locally:
```bash
python -m http.server 8000
```

Then open `http://localhost:8000/HomePage.html`.

---

## 📖 Lambda Functions Reference

### `FaceRecognitionAttendance`
**Trigger:** `POST /mark-attendance`

Accepts a base64-encoded webcam image, queries Rekognition's `StudentFaces` collection (threshold: 80% confidence), retrieves the matched student from `StudentData`, and increments their `present` and `total` counts for the given subject and semester using a full record replacement (`PutCommand`).

**Request body:**
```json
{
    "imageBase64": "data:image/jpeg;base64,...",
    "subject": "Chemistry",
    "semester": "3",
    "classDate": "2024-01-15"
}
```

---

### `get-attendance`
**Trigger:** `GET /get-attendance?subject=Chemistry&semester=3`

Scans the `AttendanceHistory` table and returns all session records matching the given subject and semester, sorted newest-first.

---

### `getStudentData`
**Trigger:** `GET /student-data` *(requires Cognito JWT in Authorization header)*

Extracts the student's `sub` claim from the Cognito authorizer context and returns their full `StudentData` record including all semester attendance.

---

### `modify-attendance`
**Trigger:** `POST /modify-attendance`

Allows faculty to batch-update attendance records. Each modification specifies a `studentId` and an `action` of either `MARK_PRESENT` or `MARK_ABSENT`. Counts are clamped between `0` and `total`.

**Request body:**
```json
{
    "subject": "Chemistry",
    "semester": 3,
    "modifications": [
        { "studentId": "uuid-1", "action": "MARK_ABSENT" },
        { "studentId": "uuid-2", "action": "MARK_PRESENT" }
    ]
}
```

---

## 🗃 Data Schema

### `StudentData` — DynamoDB Item Example

```json
{
    "StudentID": "31937dea-4031-70b5-279b-52812cbe1947",
    "firstName": "Rahul",
    "lastName": "Sharma",
    "registrationNo": "2021CS001",
    "attendance": {
        "sem3": [
            { "name": "Chemistry", "present": 18, "total": 22 },
            { "name": "Mathematics", "present": 20, "total": 22 }
        ],
        "sem4": [
            { "name": "Physics", "present": 15, "total": 18 }
        ]
    }
}
```

---

## 🔐 Authentication Flow

### Student Login (Implicit Flow)
1. Student clicks "Sign in with Google" on `Loginpage.html`
2. Redirected to Cognito Hosted UI → Google OAuth
3. Cognito redirects back with `id_token` and `access_token` in the URL fragment (`#`)
4. Tokens stored in `localStorage`, student redirected to `StudentPortal.html`
5. `StudentPortal.html` passes `id_token` as `Authorization` header to API Gateway
6. Cognito Authorizer validates token and injects `claims.sub` into the Lambda event

### Faculty Login (PKCE Flow)
1. Faculty clicks "Sign in with Google" on `FacultyLogin.html`
2. A `code_verifier` and `code_challenge` are generated locally (PKCE)
3. Faculty is redirected to Cognito → Google OAuth
4. Cognito returns an authorization `code` to the redirect URI
5. `FacultyLogin.html` exchanges the code + verifier for tokens via `POST /oauth2/token`
6. The `email` claim from the `id_token` is checked against the `authorizedEmails` allowlist
7. On success, `id_token` is stored and faculty is redirected to `FacultyPortal.html`

---

## ⚠️ Known Limitations

- **Face threshold** is fixed at 80% similarity. Poor lighting or image quality may cause recognition failures.
- The `modify-attendance` and `FaceRecognitionAttendance` lambdas use a full `PutItem` (record replacement) strategy instead of atomic `UpdateItem`. Concurrent writes for the same student could result in a race condition.
- The email allowlist in `FacultyLogin.html` is client-side. For production, this check should be enforced server-side (e.g., in a Lambda authorizer or Cognito group).
- `AttendanceHistory` is currently scanned without an index; adding a GSI on `subject + semester` is recommended for scale.

---

## 📄 License

This project is licensed under the [MIT License](LICENSE).

---

> Built with ❤️ using AWS Rekognition, Lambda, DynamoDB, and Cognito.
