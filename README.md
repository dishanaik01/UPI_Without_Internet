# UPI Offline Mesh — Secure Offline Payment Routing Demo

A Spring Boot backend demonstrating **mesh-routed deferred settlement** through a Bluetooth-style network of simulated devices.

Imagine sending ₹500 to a friend in a basement with no internet connection. Your phone encrypts the payment and passes it to nearby devices. The payment packet travels from device to device until a bridge device reaches an area with internet connectivity and uploads it to the backend.

The backend decrypts the packet, verifies it, prevents duplicate transactions, and processes settlement.

This project demonstrates the server-side architecture and a software-based mesh simulator that allows the entire workflow to be tested on a single laptop.

> **Project status:** Learning and portfolio project based on an existing open-source demonstration. Original implementation and author are credited below.

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Key Features](#key-features)
3. [Technology Stack](#technology-stack)
4. [System Architecture](#system-architecture)
5. [How the Application Works](#how-the-application-works)
6. [Security and Core Concepts](#security-and-core-concepts)
7. [Project Structure](#project-structure)
8. [Getting Started](#getting-started)
9. [API Reference](#api-reference)
10. [Testing](#testing)
11. [Limitations and Future Improvements](#limitations-and-future-improvements)
12. [Acknowledgements and Attribution](#acknowledgements-and-attribution)

---

## 1. Project Overview

Traditional digital payment systems generally require internet connectivity to communicate with payment servers and process transactions.

This project explores an alternative approach in which encrypted payment packets can travel between nearby devices without requiring every device to have an active internet connection.

The system consists of:

- A simulated sender device that creates an encrypted payment.
- Virtual devices that exchange payment packets.
- A bridge device that uploads packets when internet connectivity becomes available.
- A Spring Boot backend that verifies and processes incoming payments.
- A dashboard that displays devices, account balances, transactions, and system activity.

**Important:** This is a software simulation of an offline payment-routing concept, not an implementation of the official UPI infrastructure.

---

## 2. Key Features

- **Hybrid Encryption:** RSA-2048 and AES-256-GCM encryption for protecting payment data.
- **Mesh Network Simulation:** Simulates packet transmission between multiple virtual devices.
- **Deferred Settlement:** Payments are uploaded to the backend when a bridge device obtains internet connectivity.
- **Idempotency:** Prevents duplicate settlement when the same packet is received multiple times.
- **Replay Protection:** Uses timestamps and unique nonces to help reject replayed payments.
- **Transaction Management:** Maintains account balances and a transaction ledger.
- **Interactive Dashboard:** Provides a visual representation of devices, packets, balances, and transaction activity.
- **Concurrency Testing:** Tests simultaneous delivery of the same payment packet.

---

## 3. Technology Stack

| Technology | Purpose |
|---|---|
| Java 17+ | Backend programming language |
| Spring Boot 3.3 | Backend application framework |
| Spring Web | REST API development |
| Spring Data JPA | Database interaction |
| H2 Database | In-memory database |
| Maven | Dependency management and build automation |
| RSA-2048 | Encryption of AES session keys |
| AES-256-GCM | Payload encryption and authentication |
| HTML / CSS / JavaScript | Dashboard interface |
| JUnit | Application testing |

---

## 4. System Architecture

The application follows a layered backend architecture.

### High-level architecture

```text
       SENDER DEVICE
       (Offline)
            |
            |
      Create Payment
            |
            v
     Encrypt Payment
      RSA + AES-GCM
            |
            v
       Mesh Packet
            |
            v
   ---------------------
   |   Virtual Devices |
   |                   |
   |   Phone Alice     |
   |   Stranger 1      |
   |   Stranger 2      |
   |   Bridge Phone    |
   ---------------------
            |
       Gossip Protocol
            |
            v
      Bridge Device
      (Internet Available)
            |
            | HTTPS POST
            v
    SPRING BOOT BACKEND
            |
            v
      Hash Ciphertext
            |
            v
      Idempotency Check
            |
            v
       Decryption
            |
            v
      Freshness Check
            |
            v
        Settlement
            |
            v
     H2 Database
            |
            v
     Transaction Ledger
```

### Backend layers

| Layer | Responsibility |
|---|---|
| Controller | Handles HTTP requests and responses |
| Service | Implements business logic |
| Model | Represents application entities and repositories |
| Crypto | Handles encryption and decryption |
| Config | Application configuration |

---

## 5. How the Application Works

The dashboard demonstrates the following workflow.

### Step 1: Compose a payment

Select the sender, receiver, amount, and PIN.

Click **Inject into Mesh**.

The backend simulates the sender device, creates a payment instruction, encrypts it, and injects the resulting packet into the virtual mesh.

### Step 2: Run gossip rounds

Click **Run Gossip Round**.

Virtual devices exchange payment packets with nearby devices.

The simulator distributes packets between devices and decrements their Time to Live (TTL) values.

### Step 3: Bridge device uploads the payment

Click **Bridges Upload to Backend**.

The bridge device simulates obtaining internet connectivity and sends its held payment packets to the backend through the bridge ingestion endpoint.

### Step 4: Settlement

The backend:

1. Calculates the SHA-256 hash of the ciphertext.
2. Checks whether the packet has already been received.
3. Decrypts the payment.
4. Checks the payment timestamp.
5. Processes the debit and credit operations.
6. Records the transaction.

The dashboard displays the updated account balances and transaction ledger.

---

## 6. Security and Core Concepts

### A. Hybrid Encryption

The project uses two cryptographic algorithms:

- RSA-OAEP with SHA-256.
- AES-256-GCM.

The AES key encrypts the payment payload, while RSA encrypts the AES session key using the server's public key.

This approach allows intermediate devices to carry encrypted payment packets without directly accessing their contents.

### B. Idempotency

A payment packet may reach the backend multiple times through different bridge devices.

Without duplicate protection, the same payment could potentially be processed more than once.

The application uses a SHA-256 ciphertext hash and an atomic `ConcurrentHashMap.putIfAbsent()` operation to identify duplicate packets.

Only the first successful claimant proceeds through the settlement pipeline.

### C. Replay Protection

The encrypted payment payload includes:

- A timestamp (`signedAt`).
- A unique nonce.

The backend checks the timestamp and uses packet deduplication to help prevent previously received payments from being replayed.

### D. Transaction Management

The settlement process uses transactional database operations to debit the sender, credit the receiver, and record the transaction.

The Account entity also uses optimistic locking through JPA's `@Version` mechanism.

---

## 7. Project Structure

```text
UPI_Without_Internet/
│
├── .mvn/
│   └── wrapper/
│
├── src/
│   ├── main/
│   │   ├── java/com/demo/upimesh/
│   │   │
│   │   ├── config/
│   │   │   └── AppConfig.java
│   │   │
│   │   ├── controller/
│   │   │   ├── ApiController.java
│   │   │   └── DashboardController.java
│   │   │
│   │   ├── crypto/
│   │   │   ├── HybridCryptoService.java
│   │   │   └── ServerKeyHolder.java
│   │   │
│   │   ├── model/
│   │   │   ├── Account.java
│   │   │   ├── AccountRepository.java
│   │   │   ├── MeshPacket.java
│   │   │   ├── PaymentInstruction.java
│   │   │   ├── Transaction.java
│   │   │   └── TransactionRepository.java
│   │   │
│   │   ├── service/
│   │   │   ├── BridgeIngestionService.java
│   │   │   ├── DemoService.java
│   │   │   ├── IdempotencyService.java
│   │   │   ├── MeshSimulatorService.java
│   │   │   ├── SettlementService.java
│   │   │   └── VirtualDevice.java
│   │   │
│   │   ├── resources/
│   │   │   ├── application.properties
│   │   │   └── templates/
│   │   │       └── dashboard.html
│   │   │
│   │   └── UpiMeshApplication.java
│   │
│   └── test/
│       └── java/com/demo/upimesh/
│           └── IdempotencyConcurrencyTest.java
│
├── mvnw
├── mvnw.cmd
├── pom.xml
└── README.md
```

---

## 8. Getting Started

### Prerequisites

- Java Development Kit (JDK) 17 or newer.
- Git.
- Internet connection for downloading Maven and project dependencies on the first run.

Maven does not need to be installed separately because the project includes Maven Wrapper scripts.

### Clone the repository

```bash
git clone <YOUR_GITHUB_REPOSITORY_URL>
```

Navigate into the project directory:

```bash
cd UPI_Without_Internet
```

### Run on macOS / Linux

First, ensure the Maven wrapper has executable permission:

```bash
chmod +x mvnw
```

Start the Spring Boot application:

```bash
./mvnw spring-boot:run
```

### Run on Windows

```cmd
mvnw.cmd spring-boot:run
```

### Open the dashboard

After the application starts successfully, open:

http://localhost:8080

The interactive dashboard allows you to simulate payments and observe the complete workflow.

### Stop the application

Press:

```text
Ctrl + C
```

in the terminal.

---

## 9. API Reference

The application exposes the following REST endpoints.

| HTTP Method | Endpoint | Description |
|---|---|---|
| GET | `/api/server-key` | Returns the server's public key |
| POST | `/api/demo/send` | Creates and injects a simulated payment |
| GET | `/api/mesh/state` | Returns the current mesh state |
| POST | `/api/mesh/gossip` | Executes one gossip round |
| POST | `/api/mesh/flush` | Uploads packets held by bridge devices |
| POST | `/api/mesh/reset` | Resets the mesh and idempotency cache |
| POST | `/api/bridge/ingest` | Receives payment packets |
| GET | `/api/accounts` | Retrieves account balances |
| GET | `/api/transactions` | Retrieves the latest 20 transactions |

---

## 10. Testing

The project includes tests covering important application behaviours.

Run the complete test suite on macOS/Linux:

```bash
./mvnw test
```

On Windows:

```cmd
mvnw.cmd test
```

The documented test cases include:

1. **encryptDecryptRoundTrip:** Verifies that encrypted data can be decrypted successfully.
2. **tamperedCiphertextIsRejected:** Verifies that modified ciphertext is rejected.
3. **singlePacketDeliveredByThreeBridgesSettlesExactlyOnce:** Tests simultaneous delivery of the same payment packet by three threads and verifies that only one settlement occurs.

---

## 11. Limitations and Future Improvements

This project is an educational demonstration and is not production-ready.

| Current implementation | Possible future improvement |
|---|---|
| H2 in-memory database | PostgreSQL or MySQL |
| ConcurrentHashMap idempotency | Redis-based distributed idempotency |
| Software-simulated mesh | Real Bluetooth Low Energy or Wi-Fi Direct |
| Server-generated encryption keys | Secure key management using an HSM or KMS |
| Simulated payment accounts | Integration with regulated banking infrastructure |
| No bridge authentication | Mutual TLS or signed bridge certificates |
| Simulated payment processing | Integration with authorised payment infrastructure |

### Important conceptual limitations

- A receiver cannot independently verify the sender's available balance while completely offline.
- Offline senders could attempt to spend the same balance multiple times.
- Real-world Bluetooth communication between background mobile applications has significant technical constraints.
- Carrying encrypted packets between devices still introduces privacy and metadata considerations.

Consequently, the concept is more accurately described as **mesh-routed deferred settlement**, rather than real-time offline UPI.

---

## 12. Acknowledgements and Attribution

This repository is based on the original UPI Offline Mesh demonstration created by **perryvegehan**.

**Original repository:**

https://github.com/perryvegehan/UPI_Without_Internet

The original author is credited for the original implementation and documentation.

This repository is maintained as a learning and portfolio exercise. Any additional modifications, experiments, or independent contributions will be documented separately.

Please refer to the original repository for the source implementation and its accompanying documentation.

---

**Project:** UPI Offline Mesh — Demo  
**Category:** Backend Development | Java | Spring Boot | Cryptography  
**Purpose:** Learning, experimentation, and portfolio development