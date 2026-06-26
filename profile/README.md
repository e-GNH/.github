# Egypt CBDC Intermediary System

---
## Welcome
Whoever you are, we are CMP 26 graduation group that made the first CBDC system in egypt, second in africa completely from scratch 

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
### 2. Transaction Engine
### 3. ...
---

## Security & Cryptography

### mTLS & Secure Channel
### Hardware-Backed Keys
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

* **Languages/Compilers:** Go 1.x, Python 3.x, C++ (CMake, Make), Dart/Flutter
* **Tools:** `protoc` (Protocol Buffers)
* **Databases:** PostgreSQL
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
## How to run
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

## Intermediary 
```
go mod tidy 


docker run --name intermediary-redis -p 6379:6379 -d redis:7


docker run --name intermediary-postgres -e POSTGRES_USER=test -e POSTGRES_PASSWORD=test -e POSTGRES_DB=intermediary -p 5432:5432 -d postgres:16 
```

### Windows Users 
```
"online_wallets.sql","offline_wallets.sql","tamper.sql","pos.sql" | ForEach-Object { Get-Content ".\db\migrations\$_" | docker exec -i intermediary-postgres psql -U test -d intermediary } 
```

### Unix Users 
```
for f in online_wallets.sql offline_wallets.sql tamper.sql pos.sql; do docker exec -i intermediary-postgres psql -U test -d intermediary < ./db/migrations/$f done 

sudo allow ufc <PORT> 
```

## to run server 
```
go run main.go 
```

# mobile app 

notes: must run on an actual android phone not an emulator 

```
flutter pub get 
flutter doctor 
flutter run
```
---

## Contributors

* **[Name]** - [Role/Focus]
* **[Name]** - [Role/Focus]

---
*Confidential — Graduation Project Documentation*
