# CBDC eKYC Onboarding Architecture Specification

## Overview
This document specifies the architecture for the identity verification (eKYC) module, specifically designed for the **initial user onboarding and sign-up phase** within the CBDC ecosystem. (Subsequent authentication will rely on device-native biometrics). The pipeline enforces a strict separation of concerns between client-side capture and server-side processing, utilizing a centralized orchestrator to manage asynchronous, multi-phase inference tasks.

## 1. Client Side (Mobile / Thin Client)
The client handles hardware interfacing, edge-level validation, and secure payload transmission. 

### 1.1 Biometric Liveness Capture (Front Camera)
Captures the user's face to verify physical presence via randomized active checks (e.g., gaze shifts, smiles). Uses a staggered recording mechanism: prompts the user, waits 1 second, and records a burst of frames. Outputs a labeled dictionary mapping the requested action to the specific temporal sequence (e.g., `"smile": [frame1, frame2, ...]`).

### 1.2 Document Capture Module (Back Camera)
Captures the Egyptian National ID. Uses an overlay bounding box to enforce physical card alignment.

### 1.3 Edge Preprocessing Module
Performs local image quality checks (blur via Laplacian variance, lighting validation). Evaluates images in a loop until quality thresholds are met or local retry limits are triggered, minimizing unnecessary server load.

### 1.4 Secure Transmission Module
Packages encrypted ID images, the labeled liveness frame sequences, and a timestamped nonce into the transmission payload over TLS 1.3.

---

## 2. Server Side (Backend & Inference)
Engineered for asynchronous execution, high concurrency, and minimal network latency between ML nodes. Employs a bipartite "fail-fast" execution pipeline.

### 2.1 API Gateway & Orchestrator (The Controller)
The central nervous system of the server. Receives payloads, validates tokens, and manages the temporal execution pipeline.
* **Phase 1 (Parallel Extraction & Liveness):** Asynchronously dispatches ID images to the Vision Pipeline **AND** the labeled liveness frames to the Action-Conditioned Liveness Module. If Liveness fails, the transaction is immediately terminated.
* **Phase 2 (Parallel Resolution):** Upon receiving successful outputs from Phase 1 (extracted text, cropped ID face, and anchor selfie), it asynchronously spawns threads for Database Verification and Facial Matching.
* **Tools:** `Go` (Golang) via `Fiber` or `Gin`, utilizing `gRPC` for internal microservice communication.

### 2.2 Vision Pipeline (Unified Localization & OCR)
A merged inference context to reduce network hops. The Orchestrator sends the raw ID image here, and this pipeline handles internal routing between models.
* **Step A (Field Localization):** Isolates bounding boxes of required text fields and the ID face crop using `YOLOv8` (C++ / TensorRT).
* **Step B (OCR):** Processes cropped text fields into serialized JSON using `PaddleOCR` (C++).
* **Outputs:** Returns a unified payload containing the extracted text (Name, National ID, etc.) and the cropped ID face back to the Orchestrator.

### 2.3 Action-Conditioned Liveness Module
Operates in parallel to the Vision Pipeline. Analyzes the labeled temporal sequences provided by the client.
* **Function:** Runs 3D volume analysis and sequence tracking (via 3D CNN or CNN+LSTM) to verify depth, texture, and strict compliance with the requested action (verifying the user actually smiled during the "smile" frame burst).
* **Outputs:** Returns a `liveness_score`, a boolean `liveness_verified`, and isolates the highest-quality frame from the bursts to serve as the `anchor_selfie` for downstream matching.

### 2.4 Facial Extraction & Matching Module
Processes the `anchor_selfie` (from the Liveness Module) and the ID face crop (from the Vision Pipeline). Generates vector embeddings and computes cosine similarity to determine a match.
* **Tools:** `ArcFace` or `DeepFace` models via C++ ONNX Runtime.

### 2.5 Database Verification Module
Triggered asynchronously by the Go orchestrator during Phase 2.
* **Function:** Queries a dummy national registry to confirm the National ID exists and is valid, and checks the CBDC user database to ensure the user is not attempting to create a duplicate account.
* **Tools:** `Go`, `pgx` (async PostgreSQL drivers).

---

## 3. Threat Modeling & Countermeasures

| Attack Vector | Description | Architectural Solution |
| :--- | :--- | :--- |
| **Static Presentation** | Holding a printed photo or screen. | **Action-Conditioned Liveness:** Server strictly correlates the physical action in the temporal frames to the requested challenge, combined with 3D depth and texture analysis. |
| **Video Injection** | Bypassing hardware via virtual camera. | **Hardware Verification:** Query OS-level camera IDs. Monitor frame-drop anomalies. Temporal sequence validation prevents generic pre-recorded video injection. |
| **Replay Attacks** | Intercepting and resending a payload. | **Cryptographic Nonce:** Server-generated session token. Payload signed with nonce and strict TTL. |
| **Document Forgery** | Modifying text on an ID image. | **Anomaly Detection:** EXIF data checks. ML models detecting font aliasing mismatch. |
| **Man-in-the-Middle** | Intercepting network requests. | **Certificate Pinning:** Hardcoded SSL certificate hash in the mobile app. |