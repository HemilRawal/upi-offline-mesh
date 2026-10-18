# Offline Payment System (Mesh Network Simulation)

A Spring Boot backend that simulates offline UPI-style payments routed through a Bluetooth mesh network. The mesh is software-simulated — no real Bluetooth hardware is needed. The cryptography, idempotency, and settlement logic are production-grade.

---

## What This Project Does

The core problem: two people are in a basement with no internet. One wants to pay the other. How does the payment reach the bank?

**The answer:** encrypt the payment on the sender's phone, let it hop through nearby strangers' phones via Bluetooth gossip, and wait for any one of those phones to walk outside and get 4G. That phone uploads the encrypted packet to the backend, which decrypts, deduplicates, and settles it.

This project is the **server-side backend** plus a **software simulation of the mesh**, so the entire flow can be demonstrated on a single laptop.

---

## How to Run

### Prerequisites
- JDK 17 or newer — check with `java -version`

### Option 1 — Maven Wrapper (no install needed)
```cmd
mvnw.cmd spring-boot:run
```

### Option 2 — Docker (recommended)
```cmd
docker-compose up --build
```

Once started, open: **http://localhost:8080**

### Run Tests
```cmd
mvnw.cmd test
```

### Stop
- Maven: `Ctrl+C`
- Docker: `docker-compose down`

---

## Tech Stack

| Layer | Technology |
|---|---|
| Backend | Spring Boot 3.3, Java 17 |
| Database | H2 in-memory (zero setup) |
| Cryptography | RSA-OAEP + AES-256-GCM (JDK built-in) |
| Containerization | Docker (multi-stage build) |
| ORM | Spring Data JPA |
| Template | Thymeleaf (dashboard UI) |

---

## Architecture

```
SENDER PHONE (simulated)
  PaymentInstruction { sender, receiver, amount, nonce, timestamp }
       │
       ▼  hybrid encrypt with server's RSA public key
  MeshPacket { packetId, ttl, ciphertext }
       │
       ▼  gossip protocol (phone → phone via Bluetooth)
  [phone-alice] → [stranger1] → [stranger2] → [bridge]
                                                  │
                                                  ▼  walks outside, gets 4G
                                           HTTPS POST /api/bridge/ingest
                                                  │
SPRING BOOT BACKEND                               │
  [1] SHA-256 hash the ciphertext                 │
  [2] Idempotency check (ConcurrentHashMap)  ◄────┘
  [3] RSA-OAEP decrypt AES key
  [4] AES-256-GCM decrypt payload + verify tag
  [5] Freshness check (reject if > 24h old)
  [6] @Transactional debit + credit + ledger
```

---

## Three Problems Solved

### 1. Untrusted relay nodes
Strangers' phones carry your payment. They must not be able to read or tamper with it.

**Solution:** Hybrid encryption — RSA-OAEP wraps a one-time AES-256 key, AES-GCM encrypts the payload with authentication. Any single-bit tamper breaks the GCM tag and decryption throws. See `HybridCryptoService.java`.

### 2. Duplicate delivery
Multiple bridge nodes may upload the same packet simultaneously. Without deduplication the sender gets debited multiple times.

**Solution:** `SHA-256(ciphertext)` is the idempotency key. `ConcurrentHashMap.putIfAbsent` is atomic — exactly one thread claims the hash, the rest are dropped as `DUPLICATE_DROPPED` before any DB work happens. See `IdempotencyService.java`.

### 3. Replay attacks
An attacker captures a ciphertext and replays it later.

**Solution:** The encrypted payload contains a `signedAt` timestamp and a UUID nonce. The server rejects packets older than 24 hours. The nonce ensures two legitimate payments of the same amount have different ciphertexts. See `BridgeIngestionService.java`.

---

## How the Simulation Works

Five virtual devices are seeded on startup:

| Device | Has Internet |
|---|---|
| phone-alice | No |
| phone-stranger1 | No |
| phone-stranger2 | Yes |
| phone-stranger3 | No |
| phone-bridge | Yes |

**Gossip round:** every device that holds a packet shares it with every other device, decrementing TTL by 1 per hop. TTL = 0 means the packet stops forwarding.

**Bridge flush:** devices with `hasInternet=true` POST their packets to `/api/bridge/ingest`, simulating the moment a phone walks outside.

In a real deployment, this simulation is replaced by Android/iOS apps using Bluetooth LE or Wi-Fi Direct.

---

## Demo Flow (Dashboard at http://localhost:8080)

1. **Inject into Mesh** — choose sender, receiver, amount. Backend encrypts and hands the packet to `phone-alice`.
2. **Run Gossip Round** — click once or twice. The packet spreads to all devices.
3. **Bridges Upload** — bridge devices POST their packets to the backend. Watch balances update.
4. **Reset** — clears the mesh and idempotency cache for the next demo.

---

## API Reference

| Method | Path | Description |
|---|---|---|
| GET | `/` | Dashboard UI |
| GET | `/api/accounts` | All accounts and balances |
| GET | `/api/transactions` | Last 20 settled transactions |
| GET | `/api/mesh/state` | Packet count per virtual device |
| GET | `/api/server-key` | Server RSA public key (base64) |
| POST | `/api/demo/send` | Simulate sender phone — encrypt + inject |
| POST | `/api/mesh/gossip` | Run one gossip round |
| POST | `/api/mesh/flush` | Bridge nodes upload to backend |
| POST | `/api/mesh/reset` | Clear mesh and idempotency cache |
| POST | `/api/bridge/ingest` | Main ingest endpoint for bridge nodes |
| GET | `/h2-console` | H2 database browser |

H2 console: JDBC URL `jdbc:h2:mem:upimesh`, username `sa`, no password.

---

## Project Structure

```
src/main/java/com/demo/upimesh/
├── UpiMeshApplication.java          Spring Boot entry point
├── config/
│   └── AppConfig.java               Enables scheduled tasks
├── controller/
│   ├── ApiController.java           All REST endpoints
│   └── DashboardController.java     Serves dashboard at /
├── crypto/
│   ├── ServerKeyHolder.java         Generates RSA-2048 keypair on startup
│   └── HybridCryptoService.java     RSA-OAEP + AES-256-GCM + SHA-256 hash
├── model/
│   ├── Account.java                 JPA entity with @Version optimistic lock
│   ├── Transaction.java             Ledger entry, unique index on packetHash
│   ├── MeshPacket.java              Wire format for packets in the mesh
│   └── PaymentInstruction.java      Decrypted payment payload
└── service/
    ├── DemoService.java             Seeds accounts, simulates sender phone
    ├── VirtualDevice.java           One simulated phone
    ├── MeshSimulatorService.java    Gossip protocol across virtual devices
    ├── IdempotencyService.java      Atomic hash claim (ConcurrentHashMap)
    ├── BridgeIngestionService.java  Full ingest pipeline
    └── SettlementService.java       @Transactional debit + credit + ledger

src/test/java/com/demo/upimesh/
└── IdempotencyConcurrencyTest.java  3-thread simultaneous delivery test
```

---

## Tests

Three tests are included:

- **`encryptDecryptRoundTrip`** — verifies hybrid encryption is symmetric
- **`tamperedCiphertextIsRejected`** — flips a byte in the ciphertext, asserts `INVALID` is returned
- **`singlePacketDeliveredByThreeBridgesSettlesExactlyOnce`** — three threads deliver the same packet simultaneously, asserts exactly one `SETTLED` and two `DUPLICATE_DROPPED`

---

## What Is Real vs Simulated

| Real (production-grade) | Simulated (demo only) |
|---|---|
| RSA-OAEP + AES-256-GCM encryption | Bluetooth / Wi-Fi Direct transport |
| SHA-256 idempotency with atomic claim | Device discovery and range constraints |
| @Transactional settlement with optimistic locking | Mobile app (Android/iOS) |
| Replay and tamper protection | Multiple physical devices |
| REST API for bridge ingestion | |

---

## Honest Limitations

1. **No proof of funds offline** — the receiver has no way to verify the sender has the money until the packet settles. If the account is empty at settlement time, the payment is rejected and the receiver has no recourse. Real offline UPI (UPI Lite) solves this with a pre-funded hardware-backed wallet.
2. **Double-spend risk** — a sender with ₹500 could send ₹500 to two different people before either packet reaches the backend. Whichever settles first wins; the second is rejected.
3. **Metadata exposure** — strangers carrying your packet can see its existence even if they can't read the content.
