# MeshPay

**Offline UPI payments routed through a Bluetooth-style mesh network — settled exactly once, even when delivered multiple times, through untrusted intermediaries.**

You're in a basement with zero connectivity. You send your friend ₹500. Your phone encrypts the payment, broadcasts it to nearby phones, and the packet hops device-to-device until *some* phone walks outside, gets 4G, and silently uploads it to this backend. The backend decrypts, deduplicates, and settles.

This repo is the **server side** of that system, plus a software simulator of the mesh so the whole flow can be demoed on a single laptop — no real Bluetooth hardware needed.
 
---

## Why I built this

I wanted a project that forced me to deal with real distributed-systems problems instead of CRUD boilerplate: untrusted message carriers, at-least-once delivery, replay attacks, and concurrent duplicate writes. This demo proves three things end to end:

- **A payment can travel through untrusted intermediaries** without any of them reading or tampering with it (hybrid RSA + AES-GCM).
- **The same payment can arrive at the backend multiple times simultaneously and still settle exactly once** (atomic idempotency on the ciphertext hash).
- **A tampered or replayed packet is rejected** before it touches the ledger.
  The dashboard demos all three live, and there's a concurrency test that proves #2 under real thread contention.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         SENDER PHONE (offline)                          │
│  PaymentInstruction { sender, receiver, amount, pinHash, nonce, time }  │
│              │                                                          │
│              ▼ encrypt with server's RSA public key                     │
│   MeshPacket { packetId, ttl, createdAt, ciphertext }                   │
└──────────────────────────────────────┬──────────────────────────────────┘
                                       │ Bluetooth gossip
                                       ▼
        ┌─────────┐  hop   ┌─────────┐  hop   ┌─────────┐
        │stranger1│ ─────▶ │stranger2│ ─────▶ │ bridge  │ ◀── walks outside
        └─────────┘        └─────────┘        └────┬────┘     gets 4G
                                                   │
                                                   ▼ HTTPS POST
┌─────────────────────────────────────────────────────────────────────────┐
│                       MESHPAY BACKEND (this project)                    │
│                                                                         │
│  /api/bridge/ingest                                                     │
│       │                                                                 │
│       ▼                                                                 │
│  [1] hash ciphertext (SHA-256)                                          │
│       │                                                                 │
│       ▼                                                                 │
│  [2] IdempotencyService.claim(hash)  ◀── atomic putIfAbsent (≈ Redis    │
│       │                                  SETNX). Duplicates rejected    │
│       │                                  here, before any work.         │
│       ▼                                                                 │
│  [3] HybridCryptoService.decrypt(ciphertext)                            │
│       │       (RSA-OAEP unwraps AES key, AES-GCM decrypts payload       │
│       │        AND verifies the auth tag — tampering = exception)       │
│       ▼                                                                 │
│  [4] Freshness check: signedAt within last 24h                          │
│       │                                                                 │
│       ▼                                                                 │
│  [5] SettlementService.settle()                                         │
│       @Transactional: debit sender, credit receiver, write ledger       │
│       @Version on Account = optimistic locking (defense in depth)       │
└─────────────────────────────────────────────────────────────────────────┘
```
 
---

## The three hard problems and how they're solved

### Problem 1: Untrusted intermediates

A random stranger's phone carries your transaction. How do you stop them from reading the amount or changing it?

**Solution: Hybrid encryption (RSA-OAEP + AES-GCM).** The sender encrypts with the server's public key, so intermediates see only opaque ciphertext. RSA alone can't fit the payload, so the standard hybrid pattern is used:

1. Generate a fresh AES-256 key for *this packet*.
2. Encrypt the JSON with **AES-256-GCM** (fast + authenticated).
3. Encrypt just the AES key with **RSA-OAEP**.
4. Concatenate: `[256 bytes RSA-encrypted AES key][12 bytes IV][AES ciphertext + 16-byte GCM tag]`.
   GCM is authenticated encryption — flip one bit anywhere and decryption throws instead of silently corrupting. Same scheme TLS uses. See `HybridCryptoService.java`.

### Problem 2: The duplicate-storm

Three bridge nodes hold the same packet and all upload within milliseconds of each other. Naively processing all three would debit the sender 3x.

**Solution: Atomic compare-and-set on the ciphertext hash**, before any decryption work happens:

```java
// IdempotencyService.java
Instant prev = seen.putIfAbsent(packetHash, now);
return prev == null;  // true = first claimer, false = duplicate
```

`ConcurrentHashMap.putIfAbsent` is atomic — even 100 simultaneous threads produce exactly one claimer. The ciphertext (not the packetId, which a malicious intermediate could rewrite, and not the cleartext, which requires decrypting first) is the dedupe key. In production this becomes Redis `SET key NX EX 86400`, with a unique DB index on `packet_hash` as defense in depth.

### Problem 3: Replay attacks

An attacker who captured a ciphertext weeks ago could replay it later.

**Solution, two layers:** the encrypted payload carries a `signedAt` timestamp (packets older than 24h are rejected) and a UUID nonce, both inside the GCM-authenticated payload so neither can be altered without breaking the auth tag. A genuine replay is byte-identical to the original, so the idempotency cache catches it. See `BridgeIngestionService.java`.
 
---

## Demo flow

The dashboard walks through the full pipeline in four clicks:

1. **Inject into Mesh** — server simulates the sender's phone: builds a `PaymentInstruction`, encrypts it, hands the packet to a virtual device.
2. **Run Gossip Round** (click 1–2x) — every device broadcasts its packets to every other device; TTL decrements per hop.
3. **Bridges Upload to Backend** — the one device with internet POSTs every packet it holds to `/api/bridge/ingest`, which runs hash → claim → decrypt → freshness check → settle.
4. **Watch balances and the ledger update** in real time.
   To see the duplicate-delivery case proven under real concurrency rather than the UI:
```
mvnw.cmd test -Dtest=IdempotencyConcurrencyTest#singlePacketDeliveredByThreeBridgesSettlesExactlyOnce
```
Three threads, one packet, simultaneous delivery — asserts exactly one `SETTLED`, two `DUPLICATE_DROPPED`, sender debited exactly once.
 
---

## Honest limitations of the concept

These aren't bugs — they're inherent to "no internet, anywhere in the chain," and I'd rather be upfront about them than have someone find them first:

1. **The receiver can't verify the sender has funds.** A phone showing "₹500 sent" is an IOU, not a settled payment — if the sender's account is empty when the packet lands, settlement is `REJECTED` and the receiver has no recourse. This is why real offline UPI (UPI Lite) uses a pre-funded hardware-backed wallet.
2. **A malicious sender can double-spend offline** — send the same balance to two people in two basements; whichever packet reaches the backend first wins.
3. **Real Bluetooth mesh is hard.** Background BLE is heavily throttled on modern Android, and iOS peripheral mode is locked down. This demo simulates the mesh in software rather than solving that problem.
4. **Privacy/liability** — a stranger's phone carries your encrypted packet. They can't read it, but its existence is metadata worth thinking about for a real deployment.
   I'd describe this honestly as **"mesh-routed deferred settlement,"** not "real-time offline UPI" — the cryptography and idempotency engineering are real; the funds-guarantee problem is open.

---

## What's NOT real (and what would change for production)

| In this demo | In production |
|---|---|
| H2 in-memory DB | PostgreSQL/MySQL with replicas |
| `ConcurrentHashMap` for idempotency | Redis with `SET NX EX` |
| RSA keypair regenerated on startup | Private key in an HSM (AWS KMS, Vault) |
| Software-simulated mesh | Real BLE GATT or Wi-Fi Direct |
| One settlement service | Integration with NPCI / a real bank core |
| No auth on `/api/bridge/ingest` | Mutual TLS or signed bridge-node certs |
| In-memory seeded accounts | Real KYC'd users, real PIN verification |
| No rate limiting | Per-node and per-sender velocity limits |

The cryptography and idempotency code is essentially production-shaped; the infrastructure around it is what changes.
 
---

## File-by-file walkthrough

```
meshpay/
├── pom.xml                                  Maven build, Spring Boot 3.3, Java 17
├── mvnw, mvnw.cmd                           Maven wrapper
├── README.md
└── src/main/
    ├── resources/
    │   ├── application.properties           H2 in-memory DB, port 8080, TTLs
    │   └── templates/dashboard.html         The interactive demo UI
    └── java/com/demo/meshpay/
        ├── MeshPayApplication.java          Spring Boot main class
        ├── model/                           Account, Transaction, MeshPacket, PaymentInstruction (+ repos)
        ├── crypto/                          ServerKeyHolder, HybridCryptoService
        ├── service/                         DemoService, VirtualDevice, MeshSimulatorService,
        │                                    IdempotencyService, SettlementService, BridgeIngestionService
        ├── controller/                      ApiController, DashboardController
        └── config/AppConfig.java            @EnableScheduling for cache eviction
 
src/test/java/com/demo/meshpay/
└── IdempotencyConcurrencyTest.java          3-bridges-at-once test + tamper test
```
 
---

## API reference

| Method | Path | What it does |
|---|---|---|
| GET | `/` | Dashboard HTML |
| GET | `/api/server-key` | Server's RSA public key (base64) |
| GET | `/api/accounts` | All accounts and balances |
| GET | `/api/transactions` | Last 20 transactions |
| GET | `/api/mesh/state` | Current state of every virtual device |
| POST | `/api/demo/send` | Simulate sender phone — encrypt + inject packet |
| POST | `/api/mesh/gossip` | Run one round of gossip across the mesh |
| POST | `/api/mesh/flush` | Bridges with internet upload to backend (parallel) |
| POST | `/api/mesh/reset` | Clear mesh + idempotency cache |
| POST | `/api/bridge/ingest` | **The production endpoint.** Real bridges POST here |
| GET | `/h2-console` | Browse the in-memory database (`jdbc:h2:mem:meshpay`, user `sa`, no password) |

### Request format for `/api/bridge/ingest`

```http
POST /api/bridge/ingest
Content-Type: application/json
X-Bridge-Node-Id: phone-bridge-42
X-Hop-Count: 3
 
{
  "packetId": "550e8400-e29b-41d4-a716-446655440000",
  "ttl": 2,
  "createdAt": 1730000000000,
  "ciphertext": "base64-encoded-RSA-and-AES-blob"
}
```

Response:
```json
{
  "outcome": "SETTLED",          // or "DUPLICATE_DROPPED" or "INVALID"
  "packetHash": "a3f8c9...",
  "reason": null,                 // populated on INVALID
  "transactionId": 42             // populated on SETTLED
}
```
 
---

## Tests

```
mvnw.cmd test
```

- **`encryptDecryptRoundTrip`** — hybrid encryption is symmetric.
- **`tamperedCiphertextIsRejected`** — flipping a byte returns `INVALID`, never crashes or settles.
- **`singlePacketDeliveredByThreeBridgesSettlesExactlyOnce`** — the headline test (see Demo flow above).
---

## How to run

Requires **JDK 17+** (`java -version` to check). No database or Redis install needed — the Maven wrapper handles everything.

```bash
./mvnw spring-boot:run      # Mac/Linux
mvnw.cmd spring-boot:run    # Windows (PowerShell: .\mvnw.cmd spring-boot:run)
```

First run downloads Maven + dependencies (~90 MB, a couple of minutes). Then open **http://localhost:8080**. `Ctrl+C` to stop.

If port 8080 is taken, change `server.port` in `application.properties`.