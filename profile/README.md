# Egypt CBDC Intermediary System

---
## Welcome
We are CMP 26 graduation group that made the first CBDC system prototype in egypt, second in africa completely from scratch 

---

## Table of Contents
1. [System Architecture](#system-architecture)
2. [Core Services & Features](#core-services--features)
3. [Security & Cryptography](#security--cryptography)
4. [Project Structure](#project-structure)
5. [Prerequisites](#prerequisites)
6. [Quick Start & Deployment](#quick-start--deployment)
7. [API & Communication protocols](#api--communication-protocols)
8. [Testing](#testing)
9. [Contributors](#contributors)

---

## System Architecture

### Component Diagram
---

## Core Services & Features

### 1. Identity & Onboarding (KYC)
### 2. Anti Money Laundering (AML)
### 3. Online Payments 
### 3. Offline Payments 
### 3. Merchant Payment Gateaway
### 3. Point-of-Sale 
---

## Security & Cryptography

### TLS
### mTLS & Secure Channel
### Hardware-Backed Keys
### Rotational Keys
### Rachet Chain-of-Trust
### Digital Signatures
---

## Project Structure

```text
├── proto/                  # Protobuf definitions
├── proto_gen/              # Generated proto files (Go/Python)
├── backend/                # Go server code, middleware, handlers
├── ml-nodes/               # C++ gRPC servers (Vision, Liveness, Matching)
│   ├── vision_node/
│   ├── liveness_node/
│   └── matching_node/
├── mobile/                 # Flutter/Android client code
└── docs/                   # Architecture notes, manuals
```

---

## Prerequisites

* **Languages/Compilers:** Go 1.x, Python 3.x, C++ (CMake, Make), Dart/Flutter, Kotlin
* **Tools:** `protoc` (Protocol Buffers)
* **Databases:** PostgreSQL, Redis
* **Environment Variables:** `.env` file requirements

---

## Quick Start & Deployment

### 1. Generating Proto Files
### 2. Building and Running ML Nodes
### 3. Running the Go Backend
---

## API & Communication Protocols

* **Public Endpoints (Plain HTTPS):** e.g., `/onboarding/create-account`
* **Authenticated Endpoints (mTLS):** e.g., `/online/send`, `/online/balance`
* **Internal Microservices (gRPC):** * Vision Node (Port 50051)
  * Liveness Node (Port 50052)
  * Matching Node (Port 50053)

---

## Testing

---
# How to run
## Requirements
1. Python installed
2. Open cv installed
3. uv installed
4. go installed
5. Flutter installed

## Ledger DB
### starting step
bash
```
cd LedgerDB
```
bash
```
make run-all
```
## Intermediary 
### starting step
bash
```
cd Intermediary
```
### run the server
bash
```
make run-server
```

### Onboarding Nodes
```bash
make build-all BASE_DIR=<your path to Intermediary>
make run-all BASE_DIR=<your path to Intermediary>
```
example: 
```bash
make build-all BASE_DIR=/home/zizo/Documents/GP/Intermediary
make run-all BASE_DIR=/home/zizo/Documents/GP/Intermediary
```

### Windows Users 
```
cd Intermediary
"online_wallets.sql","offline_wallets.sql","tamper.sql","pos.sql" | ForEach-Object { Get-Content ".\db\migrations\$_" | docker exec -i intermediary-postgres psql -U test -d intermediary } 
```

# mobile app 

notes: must run on an actual android phone not an emulator 

```
cd mobile-app
make run
```
---

## Contributors

* **Ziad Samer** - [Onboarding and LedgerDB]
* **Zeyad Mohamed** - [AML and LedgerDB]
* **Fatma Zenhom** - [Online and Offline Wallets]
* **Hana Mostafa** - [Online and Offline Wallets]

---
*Confidential — Graduation Project Documentation*
