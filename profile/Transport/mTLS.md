# Egypt CBDC Intermediary System 
## mTLS Architecture 
### From Onboarding to Secure Channel 

**Version 1.0 | June 2026** 
*Confidential — Graduation Project Documentation* 

---

## 1. Overview 
This document describes the complete mutual TLS (mTLS) architecture of the Egypt CBDC Intermediary system, covering the full lifecycle from the moment onboarding produces a signed certificate to every subsequent authenticated API call made over the secure channel. 

The system uses a two-phase security model: 
* **Phase 1 — Onboarding:** The bank verifies the user's identity (KYC + liveness), then issues an X.509 client certificate signed by the bank's RSA CA.  This certificate embeds the user's EC public key and is stored on the device. 
* **Phase 2 — Authenticated Requests:** Every subsequent API call to a protected endpoint uses mTLS.  The client presents the bank-signed certificate during the TLS handshake.  The server verifies it was signed by the bank CA, extracting the user's identity directly from the certificate fields — no separate authentication header is needed. 

> **Design principle:** Identity is established once, cryptographically, at onboarding. All subsequent trust derives from that certificate. 

---

## 2. Key Architecture Decisions 

### 2.1 Two Separate Key Pairs on the Device 
The Android app maintains two distinct keypairs, each with a different purpose and security profile: 

| Key | Description |
| :--- | :--- |
| **Onboarding Key** | RSA-2048, stored in AndroidKeyStore/TEE, biometric-gated. Used to decrypt the bank's encrypted response and sign the CSR during onboarding only.  |
| **TLS Key** | EC P-256 (secp256r1), stored in AndroidKeyStore/TEE, no biometric. Used exclusively for the mTLS handshake on every API call. Never touches application-level crypto.  |

> **Why EC for TLS?** RSA keys in AndroidKeyStore cannot be used for mTLS because Conscrypt (Android's TLS library) requires RSA/ECB/NoPadding during the handshake — a raw RSA operation that AndroidKeyStore hard-blocks for RSA keys. EC keys avoid this: `DIGEST_NONE` is a valid and accepted spec option for EC, allowing Conscrypt's `NONEwithECDSA` upcall to succeed. 

### 2.2 One Certificate Per Bank, Not Per Server 
The bank operates a single CA (`CIB_CA.pem`) that signs all client certificates.  
The mTLS server uses this same CA cert as both its server identity and its client CA pool. This means: 
* No separate server leaf certificate is needed — the bank IS the server identity. 
* Client certificates are issued and verified by the same CA. 
* Adding new server endpoints requires zero certificate changes. 

### 2.3 Identity Extracted from the Certificate, Not from a Token 
The middleware in `https_middleware.go` reads identity fields directly from the client certificate presented during the TLS handshake: 

```go
cert := c.Request.TLS.PeerCertificates[0]
userID   := cert.Subject.CommonName          // e.g. national ID
bankID   := cert.Subject.Organization[0]     // e.g. "CIB"
deviceID := cert.Subject.OrganizationalUnit[0]  // Android device ID
```
No JWT, no session token, no Authorization header. The certificate is the authentication. 

---

## 3. Onboarding Phase — Establishing Identity 
Onboarding is the one-time process that creates the device's mTLS credential.  It runs over plain HTTPS on port 8080 (no client cert required). 

### 3.1 Step-by-Step Flow 

#### Step 1 — Device Generates Two Key Pairs 
On first onboarding call, `SecureOnboardingBridge` generates: 
* **Onboarding Key** (`KEY_ALIAS`): RSA-2048 in AndroidKeyStore, `PURPOSE_SIGN | PURPOSE_DECRYPT | PURPOSE_ENCRYPT`, biometric-gated. 
* **TLS Key** (`TLS_KEY_ALIAS`): EC P-256 in AndroidKeyStore, `PURPOSE_SIGN | PURPOSE_VERIFY`, `DIGEST_SHA256 | DIGEST_NONE`, no biometric. 

```kotlin
// TLS key spec — DIGEST_NONE is critical for Conscrypt compatibility
KeyGenParameterSpec.Builder(TLS_KEY_ALIAS, PURPOSE_SIGN or PURPOSE_VERIFY)
    .setAlgorithmParameterSpec(ECGenParameterSpec("secp256r1"))
    .setDigests(DIGEST_SHA256, DIGEST_NONE)   // DIGEST_NONE allows NONEwithECDSA
    .setUserAuthenticationRequired(false)     // TLS handshake is silent
    .build()
```

#### Step 2 — Build and Encrypt the Payload 
The app builds a JSON payload containing KYC data (national ID image, liveness frames) and the CSR fields (device ID + TLS public key PEM). 
The payload is then: 
* Hashed (SHA-256) and signed with the onboarding RSA private key → `SignedEnvelope` 
* Hybrid-encrypted for the bank's RSA public key: AES-256-GCM for the data, RSA-OAEP (SHA-1/MGF1-SHA-1) for the AES key wrapping 

> **Why SHA-1 for OAEP?** AndroidKeyStore hard-limits MGF1 to SHA-1. The outer OAEP hash can be SHA-256, but the MGF1 parameter must be SHA-1. The Go server's `HybridEncryptForClient()` uses `sha1.New()` to match. 

#### Step 3 — Server Processes the Request 
The Go server (`POST /onboarding/create-account`): 
* Decrypts the hybrid-encrypted payload using the bank's RSA private key 
* Verifies the `SignedEnvelope` signature using the sender's public key (embedded in the envelope) 
* Extracts the TLS public key PEM from the CSR field 
* Parses it as an EC public key via `security.ParseECPublicKey()` 
* Runs the KYC orchestrator (vision, liveness, face matching) in parallel 
* Creates the user and wallet records in Postgres 

#### Step 4 — Server Issues the Client Certificate 
The server calls `generateECCertificate()`, producing an X.509 certificate with: 

| Field | Value |
| :--- | :--- |
| **Subject CN** | User ID (national ID)  |
| **Subject O** | Bank name (e.g. CIB)  |
| **Subject OU** | Android device ID  |
| **Public Key** | Client's EC P-256 public key  |
| **Key Usage** | Digital Signature  |
| **Extended Key Usage** | TLS Web Client Authentication  |
| **Signed By** | Bank's RSA private key (CIB)  |
| **Issuer** | Bank CA (`CIB_CA.pem`)  |
| **Validity** | 1 year from issuance  |

#### Step 5 — Server Encrypts and Returns the Certificate 
The server encrypts the certificate (base64 PEM) and wallet ID back to the client using `HybridEncryptForClient()`, which uses the sender's RSA public key extracted from the original `SignedEnvelope`: 

```go
// Go — HybridEncryptForClient uses SHA-1 for AndroidKeyStore compatibility
encryptedAESKey, err := rsa.EncryptOAEP(sha1.New(), rand.Reader,
    senderPubKey, aesKey, nil)
```

#### Step 6 — Client Stores the Certificate 
The app decrypts the response using the onboarding RSA private key (biometric not required for decrypt in this flow), then calls `storeCertificateInKeyStore()`: 

```kotlin
// Link the bank-signed cert to the TLS private key under the same alias
keyStore.setKeyEntry(TLS_KEY_ALIAS, privateKey, null, arrayOf(cert))
// Now AndroidKeyStore has: EC private key + bank-signed EC cert under one alias
```

> **Result:** AndroidKeyStore alias `secure_tls_client_key` now contains an EC private key paired with a bank-signed X.509 certificate. The device is ready for mTLS. 

---

## 4. The mTLS Secure Channel 

### 4.1 Server Configuration 
The Go server runs two HTTP servers on separate ports: 

| Port | Purpose |
| :--- | :--- |
| **:8080** | Public — onboarding only. Plain HTTPS, no client cert required.  |
| **:8443** | mTLS — all authenticated endpoints. Requires and verifies client cert.  |

The mTLS server TLS config: 

```go
mtlsServer := &http.Server{
    Addr:    ":8443",
    Handler: mtlsRouter,
    TLSConfig: &tls.Config{
        ClientAuth: tls.RequireAndVerifyClientCert,
        ClientCAs:  clientCAs,   // pool containing CIB_CA.pem
        MinVersion: tls.VersionTLS12,
    },
}
// Server cert and key: the bank CA cert IS the server identity
mtlsServer.ListenAndServeTLS(
    ".../security/CIB_CA.pem",  // server certificate
    ".../security/CIB",          // server private key
)
```

### 4.2 Client Configuration (SecureHttpClient) 
`SecureHttpClient` builds two things for every HTTPS connection: 

#### Trust Manager — Verifying the Server 
Loads `CIB_CA.pem` from app assets into a custom `TrustStore`. This means the client only trusts the bank's CA, not the system CA bundle — preventing MITM via a rogue certificate authority. 

```kotlin
val bankCACert = CertificateFactory.getInstance("X.509")
    .generateCertificate(context.assets.open("bank_keys/CIB_CA.pem"))

val trustStore = KeyStore.getInstance(KeyStore.getDefaultType()).apply {
    load(null)
    setCertificateEntry("bank_ca", bankCACert)
}
// Only the bank CA is trusted — no system CAs
```

#### Key Manager — Presenting the Client Certificate 
Loads the TLS keypair from AndroidKeyStore by alias. `KeyManagerFactory` finds both the EC private key and the bank-signed EC certificate automatically: 

```kotlin
val androidKeyStore = KeyStore.getInstance("AndroidKeyStore").apply { load(null) }
// Alias holds: EC private key + bank-signed EC cert (linked by storeCertificateInKeyStore)
val kmf = KeyManagerFactory.getInstance("X509")
kmf.init(androidKeyStore, null)  // null password — AndroidKeyStore handles auth

val sslContext = SSLContext.getInstance("TLS")
sslContext.init(kmf.keyManagers, arrayOf(buildTrustManager()), null)
```

### 4.3 The TLS Handshake — What Happens on Every Request 
When `SecureHttpClient.post()` is called, the following sequence occurs automatically before any application data is exchanged: 

| Step | Action |
| :--- | :--- |
| **1** | Client → Server: ClientHello (supported cipher suites, TLS version)  |
| **2** | Server → Client: ServerHello + server certificate (`CIB_CA.pem`) + CertificateRequest  |
| **3** | Client verifies server cert against its trust store (`CIB_CA.pem`) ✓  |
| **4** | Client → Server: Client certificate (bank-signed EC cert)  |
| **5** | Client → Server: CertificateVerify — EC signature over handshake transcript using the AndroidKeyStore EC private key (`NONEwithECDSA` via Conscrypt upcall)  |
| **6** | Server verifies client cert was signed by CIB_CA ✓  |
| **7** | Both sides derive session keys — encrypted channel established  |
| **8** | Application data (HTTP request/response) flows over the encrypted channel  |

> **DIGEST_NONE explained:** At step 5, Conscrypt calls the AndroidKeyStore key using `NONEwithECDSA` (raw digest, no pre-hashing). The key spec must include `DIGEST_NONE` to permit this. Without it, AndroidKeyStore returns 'Incompatible digest' and the handshake fails. 

### 4.4 Identity Extraction in the Middleware 
Once the TLS handshake completes, every request to `:8443` passes through `MTLSIdentityMiddleware` before reaching any handler: 

```go
func MTLSIdentityMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        if c.Request.TLS == nil || len(c.Request.TLS.PeerCertificates) == 0 {
            c.AbortWithStatusJSON(401, gin.H{"error": "client certificate required"})
            return
        }
        cert     := c.Request.TLS.PeerCertificates[0]
        userID   := cert.Subject.CommonName
        bankID   := cert.Subject.Organization[0]
        deviceID := cert.Subject.OrganizationalUnit[0]

        c.Set("user_id",   userID)
        c.Set("bank_id",   bankID)
        c.Set("device_id", deviceID)
        c.Next()
    }
}
```

Any handler can then retrieve the authenticated identity with no additional work: 

```go
userID := c.GetString("user_id")   // the national ID from the cert CN
bankID := c.GetString("bank_id")   // "CIB" from the cert O field
```

---

## 5. Adding New Authenticated Endpoints 
Because authentication is handled entirely by the middleware and the TLS layer, adding a new protected endpoint is minimal: 

### 5.1 Go Backend 
Register the route on the `mtlsRouter` — the middleware is already applied at the router level: 

```go
// server.go — mtlsRouter already has MTLSIdentityMiddleware()
mtlsRouter.POST("/online/send",    handleSend(svcs.SendSvc))
mtlsRouter.GET("/online/balance",  handleBalance(svcs.BalanceSvc))
// No per-route auth code needed
```

### 5.2 Android — SecureRequestBridge 
Add one case to the `when` block and one private function: 

```kotlin
override fun onMethodCall(call: MethodCall, result: Result) {
    when (call.method) {
        "test"    -> launchRequest(result) { doTest() }
        "send"    -> launchRequest(result) { doSend(call.argument("payload")!!) }
        "balance" -> launchRequest(result) { doBalance() }
        else -> result.notImplemented()
    }
}

private fun doSend(payload: Map<String, Any>): Map<String, Any> {
    val (code, body) = httpClient.post(   // httpClient already handles mTLS
        url     = "$MTLS_BASE_URL/online/send",
        body    = JSONObject(payload).toString().toByteArray(Charsets.UTF_8),
        headers = mapOf("Content-Type" to "application/json"),
    )
    return mapOf("success" to (code in 200..299),
                 "status_code" to code, "body" to body)
}
```

> **Key point:** `SecureHttpClient.post()` is reused as-is for every endpoint. The mTLS certificate is presented automatically on every call — no per-request auth code anywhere. 

---

## 6. Security Properties 

| Property | How it is achieved |
| :--- | :--- |
| **Mutual authentication** | Both server and client prove their identity via X.509 certificates before any data is exchanged.  |
| **Certificate pinning** | The client only trusts `CIB_CA.pem` — rogue CAs cannot issue trusted server certs.  |
| **Hardware-backed keys** | Both the onboarding key and TLS key live in the device TEE via AndroidKeyStore. Keys cannot be exported.  |
| **Biometric gate** | The onboarding key (used to authorize certificate issuance) requires biometric authentication. The TLS key deliberately does not — the handshake is silent.  |
| **Device binding** | The device ID (Android `ANDROID_ID`) is embedded in the certificate OU. A certificate is valid only from the device it was issued on.  |
| **User binding** | The user ID (national ID) is the certificate CN. The server reads identity from the cert, not from a user-supplied token.  |
| **Forward secrecy** | TLS 1.2+ with ECDHE cipher suites ensures session keys are ephemeral.  |
| **No token replay** | There are no bearer tokens to steal. The private key never leaves the TEE.  |
| **Certificate expiry** | Client certificates expire after 1 year, requiring re-onboarding.  |

---

## 7. Component Summary 

| Component | Responsibility |
| :--- | :--- |
| **SecureOnboardingBridge.kt** | Generates key pairs, builds and encrypts the onboarding payload, triggers biometric auth, stores the bank-signed certificate into AndroidKeyStore after onboarding.  |
| **SecureHttpClient.kt** | Builds the SSLSocketFactory from AndroidKeyStore for every mTLS request. Loads the bank CA for server verification. Reused for all protected endpoints.  |
| **SecureRequestBridge.kt** | Flutter/Dart bridge. Routes method calls to HTTP functions. Uses SecureHttpClient — one function per endpoint.  |
| **https_middleware.go** | Gin middleware that verifies client cert presence and extracts userID/bankID/deviceID into the request context.  |
| **server.go** | Configures two servers: `:8080` (public/onboarding) and `:8443` (mTLS). Applies middleware at router level.  |
| **create_account.go** | Runs the onboarding orchestrator. Calls `generateECCertificate()` to issue the client cert signed by the bank CA.  |
| **security/keys.go** | `ParseECPublicKey()` parses the EC public key PEM sent in the CSR. `ParsePublicKey()` handles RSA keys for the onboarding flow.  |
| **security/cipher.go** | `HybridEncryptForClient()` encrypts the certificate response using SHA-1/MGF1-SHA-1 OAEP for AndroidKeyStore compatibility.  |

---

## 8. Known Constraints and Rationale 

| Constraint | Rationale |
| :--- | :--- |
| **EC only for TLS key** | AndroidKeyStore RSA keys cannot be used with Conscrypt's mTLS upcall path (RSA/ECB/NoPadding is hard-blocked). EC keys work with `DIGEST_NONE`.  |
| **SHA-1 in OAEP for client-bound encryption** | AndroidKeyStore hard-limits MGF1 to SHA-1. `HybridEncryptForClient()` uses `sha1.New()` on the Go side; decryptResponse() uses SHA-1/MGF1-SHA-1 on the Android side.  |
| **Hostname verification relaxed** | The server uses an IP address (192.168.x.x) during development. HostnameVerifier accepts the IP directly. In production, a proper hostname with a matching CN/SAN in the server cert should be used.  |
| **One bank CA for both server and client certs** | Simplifies the PKI for a single-bank deployment. Multi-bank deployments would require per-bank CA certs in the client trust store.  |

---
*Egypt CBDC Intermediary — mTLS Architecture Documentation — June 2026* 