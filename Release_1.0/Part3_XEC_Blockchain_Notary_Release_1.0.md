# LMWPool XEC Platform
## Volume 3 — XEC Blockchain Notary

| Document metadata | Value |
|---|---|
| Document | Technical Architecture and Operations Manual |
| Release | 1.0 |
| Author | Bruno Stefanutti |
| Platform | eCash (XEC) |
| Environment | Production |
| Status | Definitive release |

> **Document scope.** This volume documents the deployed XEC Blockchain Notary, including its Node.js application, SQLite metadata store, REST API, notarization workflow, blockchain integration, certificate generation, notifications, authentication, verification, production operations, security and disaster recovery.

---

## Document Control

> **Release note:** This document is Volume 3 of the LMWPool Technical Documentation Release 1.0 set. Read it together with Volumes 1, 2 and 4 and Appendices A–F for the complete platform reference.

| Field | Value |
|---|---|
| Release | 1.0 |
| Source baseline | Volume 3 RC2 |
| Intended audience | Platform owner, system administrators, technical maintainers, integration developers |
| Change policy | Production changes must be reflected in the relevant manual volume |
| Architectural authority | Running production configuration and verified operational evidence |

## Table of Contents

1. [Platform Overview](#1-platform-overview)
2. [`server.js` — Core Application](#2-serverjs--core-application)
3. [SQLite Database](#3-sqlite-database)
4. [REST API](#4-rest-api)
5. [Document Notarization Workflow](#5-document-notarization-workflow)
6. [Blockchain Integration](#6-blockchain-integration)
7. [PDF Certificate Generation](#7-pdf-certificate-generation)
8. [Email Notification System](#8-email-notification-system)
9. [User Management and Authentication](#9-user-management-and-authentication)
10. [Verification Services](#10-verification-services)
11. [System Deployment and Production Operations](#11-system-deployment-and-production-operations)
12. [Administration and Maintenance](#12-administration-and-maintenance)
13. [Security, Privacy and Compliance](#13-security-privacy-and-compliance)
14. [Disaster Recovery and Business Continuity](#14-disaster-recovery-and-business-continuity)

---

## 1. Platform Overview

### 1.1 Purpose

The XEC Blockchain Notary is an independent production service within the LMWPool platform.

Its purpose is to provide **cryptographic proof of existence** by permanently anchoring the SHA-256 hash of digital documents into the eCash blockchain.

Unlike conventional cloud notarization platforms, the service operates on infrastructure fully controlled by the platform owner.

Its principal components are self-hosted:

- Node.js application;
- SQLite database;
- eCash full node;
- PDF certificate generation;
- email subsystem;
- REST API; and
- blockchain verification.

This architecture keeps the notarization workflow under the platform owner's control without relying on external blockchain providers.

---

### 1.2 Design Objectives

The project was designed around five primary objectives.

#### Simplicity

A user should be able to notarize a document within a few seconds.

The workflow intentionally minimizes user interaction.

---

#### Blockchain Independence

Every blockchain operation is executed against the local `bitcoind-xec` node, the Docker deployment of the official eCash Core full node.

No third-party blockchain APIs participate in the notarization process.

---

#### Privacy

Uploaded documents are never written to the blockchain.

Only the SHA-256 digest is permanently recorded.

Consequently:

- document contents remain private;
- document ownership remains private;
- blockchain storage requirements remain minimal.

---

#### Deterministic Verification

Every notarization can be verified using:

- original document
- SHA-256 hash
- blockchain transaction
- generated certificate

The verification procedure is fully reproducible.

---

#### Production Reliability

The service is designed for continuous production operation.

Failure of auxiliary services must not compromise already completed notarizations.

---

### 1.3 Platform Overview

The Blockchain Notary is composed of six principal components.

| Component | Technology | Responsibility |
|---|---|---|
| `server.js` | Node.js | REST API and application orchestration |
| SQLite | SQLite | Metadata persistence |
| `bitcoind-xec` | eCash Core | Blockchain interaction |
| PDF Engine | Node.js | Certificate generation |
| SMTP | Mail | User notifications |
| Caddy | Reverse Proxy | HTTPS publication |

Every component performs one clearly defined responsibility.

---

### 1.4 Production Architecture

```
                     Internet

                         │

                    HTTPS

                         │

                    Caddy

                         │

                         ▼

                 server.js (Node.js)

          ┌──────────┴──────────┐

          ▼                     ▼

      SQLite DB           bitcoind-xec

          │                     │

          ▼                     ▼

    Certificates          eCash Blockchain

          │

          ▼

      SMTP Server

          │

          ▼

         User
```

Unlike the mining platform, the Notary is implemented primarily as a Node.js application running directly on the host operating system.

It is **not deployed inside Docker**.

---

### 1.5 Production Directory

Production installation:

```
/root/xec_notary
```

Typical structure:

```
xec_notary

│

├── server.js

├── package.json

├── data/

├── certificates/

├── templates/

├── public/

├── logs/

└── utilities/
```

The application is intentionally self-contained.

---

### 1.6 Main Responsibilities

The service is responsible for:

- document registration;
- SHA-256 verification;
- blockchain submission;
- transaction monitoring;
- PDF certificate generation;
- email notification;
- verification API;
- notarization history.

Mining functionality is completely outside its scope.

---

### 1.7 Operational Philosophy

The Notary follows the same architectural philosophy as the mining platform:

- small components;
- single responsibility;
- deterministic behavior;
- clear operational boundaries;
- independent verification; and
- no unnecessary dependencies.

Every blockchain interaction is explicit and fully traceable.

---

### 1.8 Service Lifecycle

The normal lifecycle of a notarization is:

```
Upload

↓

SHA-256

↓

REST API

↓

SQLite

↓

Blockchain

↓

Confirmation

↓

Certificate

↓

Email

↓

Verification
```

Every stage can be independently monitored and audited.

---

### 1.9 Relationship with the Mining Platform

Although hosted on the same VPS, the Notary remains logically separated from mining.

The two systems share only the:

- Linux host;
- `bitcoind-xec` node;
- Caddy reverse proxy; and
- Internet connection.

No mining component depends on the Notary.

No Notary component depends on CKPool.

This separation prevents application-level failures in one environment from interrupting the core functionality of the other.

---

### 1.10 Operational Characteristics

The Notary is intended to operate continuously.

Primary characteristics include:

- lightweight resource usage;
- local SQLite storage;
- direct blockchain interaction;
- REST-oriented architecture;
- asynchronous confirmation workflow;
- deterministic certificate generation.

These characteristics make the service suitable for both public access and future integration into external applications.

---

### 1.11 Summary

The XEC Blockchain Notary represents the second major production subsystem of the LMWPool platform.

While the mining infrastructure focuses on proof of work, the Notary provides blockchain-based trust services built upon the same authoritative eCash full node.

Its architecture emphasizes simplicity, privacy, deterministic verification and long-term maintainability.

The following chapters describe each subsystem in detail, beginning with the Node.js application (`server.js`), which acts as the operational core of the service.

## 2. `server.js` — Core Application

### 2.1 Purpose

`server.js` is the operational core of the XEC Blockchain Notary.

Every notarization request passes through this application.

It coordinates the:

- REST API;
- user validation;
- SQLite metadata store;
- blockchain interaction;
- certificate generation;
- email delivery; and
- confirmation workflow.

No other component contains business logic.

For this reason, `server.js` is the central orchestration layer of the Notary platform.

---

### 2.2 Architectural Role

```
                    Browser

                        │

                    HTTPS

                        │

                     Caddy

                        │

                        ▼

                  server.js

     ┌─────────────┼──────────────┐

     ▼             ▼              ▼

 SQLite        Blockchain        SMTP

     │             │              │

     ▼             ▼              ▼

 Metadata      TX Broadcast     Emails

     │

     ▼

 PDF Certificates
```

Every external request eventually reaches this application.

---

### 2.3 Responsibilities

`server.js` performs the following responsibilities.

#### REST Server

Receives HTTPS requests.

---

#### User Management

Identifies users.

Validates API access.

Handles User Keys.

---

#### SHA-256 Registration

Registers document hashes.

Checks duplicates.

Creates notarization requests.

---

#### Blockchain Submission

Communicates with

```
bitcoind-xec
```

through RPC.

Creates blockchain transactions.

Stores TXIDs.

---

#### Database Management

Reads

Writes

Updates

SQLite records.

---

#### PDF Certificates

Generates notarization certificates.

Provides downloadable PDF documents.

---

#### Email Notifications

Sends

confirmation emails.

completion emails.

certificate notifications.

administrative notifications.

---

#### History

Provides user history.

Supports historical searches.

---

### 2.4 Runtime Environment

Unlike CKPool, `server.js` executes directly on Linux.

It is not containerized.

Typical runtime

```
Node.js

↓

Express

↓

SQLite

↓

bitcoind-xec RPC
```

This architecture simplifies interaction with the local filesystem.

---

### 2.5 REST Architecture

The application follows a REST-oriented architecture.

```
HTTP Request

↓

Validation

↓

Business Logic

↓

SQLite

↓

Blockchain

↓

Response
```

Every endpoint follows the same processing model.

---

### 2.6 Main REST Endpoints

Production endpoints include:

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/notarize` | Create a notarization |
| `GET` | `/verify/:hash` | Verify a document hash |
| `GET` | `/certificate/:hash.pdf` | Download a certificate |
| `GET` | `/history` | Return user history |

These endpoints represent the public interface of the service.

---

### 2.7 Request Lifecycle

Typical request.

```
Browser

↓

POST

/notarize

↓

Validation

↓

SHA-256

↓

Duplicate Check

↓

SQLite

↓

Blockchain

↓

Response
```

The request is completed only after every mandatory validation succeeds.

---

### 2.8 Validation Layer

Before any blockchain operation is performed, `server.js` validates:

- document hash;
- filename;
- customer;
- email;
- User Key;
- request consistency; and
- duplicate requests.

Invalid requests are rejected immediately.

---

### 2.9 Duplicate Detection

One of the first operations performed by `server.js` is duplicate detection.


**Workflow:**

```
Incoming SHA-256

↓

SQLite Lookup

↓

Already Exists ?

↓

YES

↓

Return Existing Information

↓

NO

↓

Continue Workflow
```

This prevents unnecessary blockchain transactions.

---

### 2.10 Database Layer

Every notarization generates metadata.

Examples include:

- document hash;
- filename;
- email;
- customer ID;
- submission timestamp;
- blockchain status;
- transaction ID; and
- certificate status.

Only metadata are stored.

The original document is **never persisted**.

---

### 2.11 Blockchain Layer

Once validation succeeds, `server.js` communicates with `bitcoind-xec` through RPC.

Typical operations include:

Submit notarization.

Retrieve transaction.

Check confirmations.

Verify blockchain state.

The blockchain remains the authoritative record.

SQLite stores operational metadata only.

---

### 2.12 Certificate Generation

Following successful notarization, `server.js` generates a:

```
PDF Certificate
```

containing the hash, filename, timestamp, blockchain, transaction ID and verification URL.

Certificate generation is deterministic.

Regenerating a certificate always produces identical blockchain information.

---

### 2.13 Email Workflow

Typical sequence

```
Submission

↓

Email

↓

Blockchain

↓

Confirmation

↓

Certificate

↓

Final Email
```

Emails therefore reflect the real state of the notarization.

---

### 2.14 Error Handling

Typical failures include:

Invalid request.

↓

Duplicate.

↓

RPC unavailable.

↓

SQLite unavailable.

↓

SMTP unavailable.

↓

Certificate generation failure.

Each condition generates an explicit response.

Silent failures are intentionally avoided.

---

### 2.15 Logging

Every important operation should be logged.

Recommended events include:

API requests.

Validation failures.

Blockchain submission.

Transaction confirmation.

Certificate generation.

Email delivery.

Unexpected exceptions.

These logs are essential for production troubleshooting.

---

### 2.16 Security

`server.js` validates:

User Key.

Email.

Document hash.

Customer identifier.

Only authenticated requests requiring privileged operations should proceed.

The application never trusts client-side validation alone.

---

### 2.17 Relationship with Other Components

```
Browser

↓

Caddy

↓

server.js

↓

SQLite

↓

bitcoind-xec

↓

SMTP

↓

PDF
```

No component bypasses `server.js`.

It remains the single orchestration layer for the entire Notary platform.

---

### 2.18 Architectural Importance

Among the software components that compose the Blockchain Notary, `server.js` is the central application component.

It concentrates the business logic while delegating persistence, blockchain interaction, email delivery and certificate generation to specialized subsystems.

This architecture keeps responsibilities clearly separated while allowing the entire notarization workflow to remain deterministic, auditable and maintainable over time.

## 3. SQLite Database

### 3.1 Purpose

The SQLite database is the persistent metadata repository of the XEC Blockchain Notary.

Unlike the blockchain, which provides immutable public evidence, SQLite stores the operational information required to manage the service.

Examples include:

- users;
- document hashes;
- email addresses;
- customer identifiers;
- blockchain status;
- transaction identifiers;
- certificate status; and
- notification history.

The database therefore complements the blockchain without replacing it.

---

### 3.2 Design Philosophy

A fundamental architectural principle of the Notary is:

> **The blockchain stores proof. SQLite stores metadata.**

This separation provides several advantages.

The blockchain remains immutable.

SQLite remains searchable.

The blockchain guarantees integrity.

SQLite guarantees operational efficiency.

---

### 3.3 Position inside the Architecture

```
                 Browser

                     │

                     ▼

                 server.js

              ┌──────┴───────┐

              ▼              ▼

          SQLite         bitcoind-xec

              │              │

              ▼              ▼

      Operational      Blockchain

       Metadata          Evidence
```

Neither component replaces the other.

Each stores different information.

---

### 3.4 Responsibilities

SQLite stores:

- registered users;
- document metadata;
- SHA-256 hashes;
- customer IDs;
- submission timestamps;
- blockchain transaction IDs;
- confirmation status;
- certificate information; and
- operational history.

The original uploaded document is never stored.

---

### 3.5 Why SQLite?

SQLite was selected because the platform requirements include:

- a single production server;
- lightweight deployment;
- no separate database administration;
- high reliability; and
- low maintenance.

The expected transaction volume is well within SQLite capabilities.

Consequently, introducing PostgreSQL or MySQL would unnecessarily increase operational complexity.

---

### 3.6 Database Location

Production database directory:

```
/root/xec_notary/data/
```

Primary file:

```
notary.sqlite
```

This file represents one of the most valuable assets of the platform.

Regular backup is mandatory.

---

### 3.7 Logical Data Model

Although implementation details may evolve, the logical entities include:

Users

↓

Documents

↓

Notarizations

↓

Blockchain Transactions

↓

Certificates

↓

History

Each entity has a clearly defined responsibility.

Business logic remains inside

```
server.js
```

not inside the database.

---

### 3.8 User Records

Each user record typically contains a customer identifier, email address, registration date, User Key, status and preferences.

The User Key is used to authenticate subsequent notarization requests.

---

### 3.9 Document Records

Each notarized document stores metadata such as the SHA-256 hash, filename, submission timestamp, customer, current status, blockchain transaction and certificate availability.

Only metadata are persisted.

Document contents remain outside the database.

---

### 3.10 Blockchain Information

SQLite stores blockchain references rather than blockchain data.

Examples include the transaction ID, confirmation state, submission timestamp, blockchain network and verification URL.

The blockchain itself remains the authoritative record.

---

### 3.11 Certificate Metadata

Certificate records typically include the creation date, availability, download path, generation status and associated notarization.

This information allows certificates to be regenerated or downloaded without repeating the notarization process.

---

### 3.12 Duplicate Detection

Before creating a new notarization,

`server.js` performs a lookup using the SHA-256 hash.


**Workflow:**

```
Incoming Hash

↓

SQLite Lookup

↓

Exists ?

↓

YES

↓

Return Existing Record

↓

NO

↓

Create New Record
```

This prevents duplicate registrations.

---

### 3.13 Transaction Lifecycle

The typical lifecycle stored in SQLite is:

```
Request

↓

Registered

↓

Blockchain Submitted

↓

Pending Confirmation

↓

Confirmed

↓

Certificate Generated

↓

Completed
```

The database therefore tracks the operational state of each notarization.

---

### 3.14 Relationship with `server.js`

SQLite never executes business rules.

Instead:

```
server.js

↓

Validation

↓

SQLite

↓

Store Metadata

↓

Return Control
```

All decision-making remains inside the application layer.

---

### 3.15 Backup Strategy

The SQLite database should be backed up:

- daily;
- before software upgrades;
- before schema modifications; and
- before operating-system maintenance.

Because SQLite is a single file, backup procedures remain extremely simple.

---

### 3.16 Recovery Procedure

Typical recovery sequence

Stop `server.js`

↓

Restore

```
notary.sqlite
```

↓

Verify file permissions

↓

Restart application

↓

Validate REST API

↓

Verify recent notarizations

No blockchain data need to be restored because they remain permanently available on-chain.

---

### 3.17 Failure Scenarios

Database unavailable

↓

New notarizations rejected.

---

Filesystem full

↓

Insert operations fail.

---

Database corruption

↓

Metadata unavailable.

Blockchain evidence remains intact.

---

Missing backup

↓

Operational history lost.

Blockchain evidence still exists but user metadata may become unrecoverable.

---

### 3.18 Operational Checks

Verify file

```bash
ls -lh data/notary.sqlite
```

Verify permissions

```bash
ls -l data/
```

Verify application connectivity

Create a test notarization.

Confirm successful database insertion.

Verify lookup operations.

---

### 3.19 Architectural Advantages

Using SQLite provides several advantages.

- extremely simple deployment;
- no database server;
- low memory usage;
- excellent reliability;
- easy backup;
- easy migration;
- deterministic behavior.

These characteristics are particularly well suited to a self-hosted blockchain notarization platform.

---

### 3.20 Architectural Summary

SQLite is not intended to replace the blockchain.

Instead, it provides the operational metadata required to manage users, notarizations, certificates and application state.

Together, SQLite and the eCash blockchain form the persistence layer of the XEC Blockchain Notary:

- **SQLite** manages operational information.
- **The blockchain** provides immutable cryptographic evidence.

This separation is one of the fundamental architectural principles of the Notary platform.

## 4. REST API

### 4.1 Purpose

The REST API represents the public interface of the XEC Blockchain Notary.

Every interaction between external clients and the platform is performed through HTTP endpoints implemented by `server.js`.

These endpoints allow applications, websites and automation tools to interact with the Notary without requiring direct access to the internal implementation.

The REST API is therefore the official entry point of the service.

---

### 4.2 Design Philosophy

The API follows several design principles.

- REST-oriented design;
- stateless requests;
- JSON communication;
- explicit validation;
- predictable responses; and
- deterministic behavior.

Each request contains all information required to complete the operation.

No client session state is maintained.

---

### 4.3 API Architecture

```
                Client

                   │

               HTTPS

                   │

                   ▼

               Caddy

                   │

                   ▼

              server.js

                   │

          Request Validation

                   │

                   ▼

         Business Processing

          ┌────────┴─────────┐

          ▼                  ▼

      SQLite            bitcoind-xec RPC

          │                  │

          ▼                  ▼

       Metadata        Blockchain

          │

          ▼

      JSON Response
```

Every request follows the same processing pipeline.

---

### 4.4 Communication Format

Requests and responses use JSON whenever possible.


**Typical workflow:**

```
Client

↓

JSON Request

↓

REST API

↓

Validation

↓

Business Logic

↓

JSON Response
```

File uploads are handled separately using multipart requests.

---

### 4.5 Main Endpoints

The production API currently exposes four primary endpoints.

---

#### `POST /notarize`


**Purpose:**

Create a new blockchain notarization.

Responsibilities include:

- validating the request;
- verifying the user;
- detecting duplicates;
- creating the blockchain record;
- updating SQLite; and
- returning operation status.

This endpoint represents the primary function of the platform.

---

#### `GET /verify/:hash`


**Purpose:**

Verify whether a SHA-256 hash has already been notarized.

The endpoint returns blockchain-related information associated with the supplied digest.

Verification does not require uploading the original document.

---

#### `GET /certificate/:hash.pdf`


**Purpose:**

Download the generated notarization certificate.

If available, the PDF is returned directly to the client. Otherwise, an appropriate error response is generated.

---

#### `GET /history`


**Purpose:**

Return the notarization history associated with a specific user.

Results are generated from SQLite metadata.

Blockchain verification data remain available through the associated transaction identifiers.

---

### 4.6 Request Validation

Before executing any business logic, `server.js` validates:

Required parameters.

↓

Hash format.

↓

Email format.

↓

Customer identifier.

↓

User Key (when required).

↓

Request consistency.

Only fully validated requests continue.

---

### 4.7 User Authentication

The platform intentionally distinguishes between:

New users

↓

Existing users

A first notarization may create a new user record.

Subsequent requests require a valid

```
User Key
```

This mechanism prevents unauthorized use of existing accounts while keeping the registration workflow simple.

---

### 4.8 Duplicate Detection

Before creating a new notarization,

the API searches SQLite.

```
SHA-256

↓

Database Lookup

↓

Already Exists ?

↓

YES

↓

Return Existing Information

↓

NO

↓

Continue Processing
```

This prevents unnecessary blockchain transactions.

---

### 4.9 Response Structure

Every endpoint returns deterministic responses. Typical responses contain:

- operation status;
- a message;
- blockchain information;
- transaction identifiers;
- certificate availability; and
- additional metadata.

Clients should always evaluate the operation status before processing returned data.

---

### 4.10 Error Responses

Typical error categories include:

Invalid request.

↓

Missing parameters.

↓

Unknown user.

↓

Invalid User Key.

↓

Duplicate notarization.

↓

Blockchain unavailable.

↓

Database unavailable.

↓

Unexpected server error.

Errors are intentionally explicit to simplify client integration.

---

### 4.11 HTTP Status Codes

The API follows conventional HTTP semantics.

Typical examples include:

```
200
```

Successful request.

---

```
400
```

Invalid request.

---

```
401
```

Authentication failure.

---

```
404
```

Requested resource not found.

---

```
500
```

Unexpected server error.

Clients should rely on both the HTTP status code and the returned JSON message.

---

### 4.12 Security

The REST API never exposes:

- the SQLite schema;
- RPC credentials;
- wallet information;
- internal file paths;
- Docker configuration; or
- server configuration.

Only information required by the client is returned.

---

### 4.13 Interaction with Other Components

```
REST Client

↓

Caddy

↓

server.js

↓

Validation

↓

SQLite

↓

bitcoind-xec

↓

Certificate

↓

SMTP

↓

JSON Response
```

The REST API orchestrates every subsystem without exposing their internal implementation.

---

### 4.14 Integration Possibilities

Because the platform exposes a REST interface,

it can be integrated with:

- websites;
- ERP systems;
- document management platforms;
- workflow engines;
- desktop applications;
- Microsoft Excel;
- custom automation scripts.

This makes the Blockchain Notary suitable for future enterprise integrations without requiring modifications to its internal architecture.

---

### 4.15 Operational Checks

Recommended operational tests include:

Create a notarization.

↓

Verify returned transaction.

↓

Download certificate.

↓

Verify history.

↓

Verify hash lookup.

↓

Confirm email delivery.

These tests validate the complete API workflow.

---

### 4.16 Architectural Summary

The REST API is the public face of the XEC Blockchain Notary.

It provides a clean separation between external clients and the internal implementation while coordinating validation, blockchain interaction, metadata persistence, certificate generation and user notifications.

By concentrating all client interaction within a small set of well-defined endpoints, the platform remains secure, maintainable and easily extensible for future applications.

## 5. Document Notarization Workflow

### 5.1 Purpose

The primary function of the XEC Blockchain Notary is to transform a digital document into a permanently verifiable blockchain record.

The workflow has been designed to be:

- deterministic;
- reproducible;
- secure;
- independent from document content.

The blockchain never stores the uploaded document itself.

Only its SHA-256 fingerprint becomes permanently anchored to the eCash blockchain.

---

### 5.2 High-Level Workflow

The complete notarization process is illustrated below.

```
Document

↓

SHA-256

↓

REST API

↓

Validation

↓

Duplicate Detection

↓

SQLite

↓

Blockchain Transaction

↓

Confirmation

↓

PDF Certificate

↓

Email Notification

↓

History
```

Every stage produces a well-defined and verifiable result.

---

### 5.3 Client-Side Hash Generation

Whenever possible, the SHA-256 digest is generated **inside the user's browser** using the WebCrypto API.


**Workflow:**

```
User selects file

↓

Browser

↓

SHA-256

↓

Document Hash

↓

REST Request
```

This architecture offers two major advantages:

- the server never needs to calculate the hash;
- the original document does not need to leave the user's computer for hash generation.

The client transmits only:

- document hash;
- filename;
- metadata required for notarization.

---

### 5.4 Request Submission

The browser submits a request to

```
POST /notarize
```

The request typically includes:

- SHA-256 digest;
- filename;
- email address;
- customer identifier;
- User Key (when required).

No blockchain operation begins before the request has been validated.

---

### 5.5 Request Validation

The server validates:

✓ SHA-256 format

✓ filename

✓ customer identifier

✓ email address

✓ User Key (existing users)

✓ duplicate request

Any validation failure immediately terminates the workflow.

---

### 5.6 Duplicate Detection

Before creating a blockchain transaction,

the application searches SQLite.

```
Incoming SHA-256

↓

Database Lookup

↓

Already Registered?
```

If the hash already exists,

the existing notarization is returned.

No additional blockchain transaction is created.

This duplicate-detection workflow returns the existing blockchain proof for a previously registered hash.

---

### 5.7 Registration

For a new document,

`server.js` creates a database record containing:

- document hash;
- filename;
- customer identifier;
- email;
- submission timestamp;
- processing status.

At this stage the document has been accepted but not yet permanently recorded on the blockchain.

---

### 5.8 Blockchain Submission

After successful registration,

the application prepares the blockchain transaction.

```
SQLite Record

↓

Blockchain Payload

↓

bitcoind-xec RPC

↓

Transaction Broadcast

↓

TXID
```

The returned Transaction ID (TXID) uniquely identifies the blockchain operation.

---

### 5.9 Pending Confirmation

Immediately after submission,

the notarization enters a pending state.

```
Submitted

↓

Waiting

↓

Blockchain Confirmation
```

This reflects the normal operation of a proof of work blockchain.

The document is registered,

but final confirmation depends on block inclusion.

---

### 5.10 Confirmation

Once the transaction is confirmed,

the notarization status changes.

```
Pending

↓

Confirmed

↓

Certificate Generation

↓

Notification
```

Only confirmed notarizations are considered complete.

---

### 5.11 Certificate Generation

Following confirmation,

`server.js` generates a PDF certificate containing:

- SHA-256 hash;
- filename;
- notarization timestamp;
- blockchain name;
- TXID;
- verification URL;
- certificate generation date.

The certificate is deterministic.

It can always be regenerated from the stored metadata.

---

### 5.12 Email Workflow

The notification sequence is:

```
Submission Accepted

↓

Confirmation Email

↓

Blockchain Confirmation

↓

Certificate Generated

↓

Completion Email
```

The user is therefore informed throughout the notarization lifecycle.

---

### 5.13 History Update

After completion,

SQLite updates the user's history.

The new record becomes available through:

```
GET /history
```

Users can therefore retrieve previous notarizations without repeating the submission process.

---

### 5.14 Verification

A completed notarization can be verified independently using:

- original document;
- SHA-256 digest;
- blockchain transaction;
- PDF certificate;
- verification endpoint.

The verification process does not require access to the original upload session.

---

### 5.15 Failure Handling

Possible interruptions include:

Validation failure

↓

Duplicate request

↓

RPC unavailable

↓

Blockchain rejection

↓

Database failure

↓

Certificate generation failure

↓

SMTP failure

Each stage reports an explicit operational status.

Partial processing is avoided whenever possible.

---

### 5.16 Operational State Diagram

```
Received

↓

Validated

↓

Registered

↓

Submitted

↓

Pending Confirmation

↓

Confirmed

↓

Certificate Generated

↓

Completed
```

Every notarization progresses through these states in a deterministic order.

---

### 5.17 Design Advantages

The workflow provides several architectural benefits:

- client-side SHA-256 generation;
- deterministic validation;
- duplicate protection;
- blockchain-backed evidence;
- asynchronous confirmation;
- reproducible certificates;
- complete audit trail.

Each step has a single responsibility and can be monitored independently.

---

### 5.18 Architectural Summary

The notarization workflow is the central business process of the XEC Blockchain Notary.

It combines browser-side cryptographic hashing, rigorous server-side validation, blockchain registration, metadata persistence and certificate generation into a single, deterministic process.

This architecture ensures that every completed notarization can be independently verified long after the original submission, providing durable and trustworthy proof of document existence while preserving the privacy of the document contents.

## 6. Blockchain Integration

### 6.1 Purpose

The Blockchain Integration layer connects the XEC Blockchain Notary to the **eCash blockchain** through the locally managed `bitcoind-xec` full node.

This component is responsible for transforming a validated notarization request into an immutable blockchain transaction.

Unlike many commercial notarization platforms, the LMWPool Notary does **not** rely on external blockchain providers.

Every blockchain operation is executed against the locally controlled production node.

This provides:

- operational independence from external blockchain providers;
- deterministic behavior;
- maximum privacy;
- long-term service stability.

---

### 6.2 Design Philosophy

The blockchain is considered the **single source of cryptographic truth**.

SQLite stores operational metadata.

The blockchain stores immutable evidence.

```
SQLite

↓

Metadata

Blockchain

↓

Proof
```

Neither component replaces the other.

Together they form the persistence layer of the platform.

---

### 6.3 Architecture

```
                 server.js

                     │

             Blockchain Request

                     │

                     ▼

              bitcoind-xec

                     │

                 JSON-RPC

                     │

                     ▼

              eCash Network

                     │

                     ▼

             Confirmed Block
```

Every blockchain interaction follows this architecture.

---

### 6.4 Production Node

The production blockchain node is `bitcoind-xec`.

This is the same fully synchronized node used by the mining infrastructure.

Advantages include:

- no external API dependency;
- local blockchain validation;
- direct RPC access;
- deterministic responses;
- complete control over blockchain operations.

The Notary therefore benefits from the same blockchain infrastructure already maintained for mining.

---

### 6.5 RPC Communication

Communication between `server.js` and `bitcoind-xec` is performed through authenticated JSON-RPC.

Typical operations include:

- transaction creation;
- transaction broadcasting;
- transaction lookup;
- confirmation monitoring;
- blockchain verification.

RPC remains internal.

It is never exposed to the Internet.

---

### 6.6 Transaction Creation

After successful validation,

the application prepares the blockchain payload.

```
Validated Request

↓

Blockchain Payload

↓

RPC

↓

Transaction

↓

TXID
```

The returned Transaction ID uniquely identifies the notarization.

---

### 6.7 Transaction Identifier (TXID)

Every successful notarization receives a unique

```
TXID
```

The TXID is stored in SQLite and included in:

- verification endpoint;
- PDF certificate;
- confirmation emails;
- notarization history.

The TXID allows anyone to independently verify the blockchain evidence.

---

### 6.8 Confirmation Monitoring

Immediately after broadcast,

the transaction enters the

```
Pending
```

state.


**Workflow:**

```
Broadcast

↓

Mempool

↓

Mining

↓

Confirmed

↓

Completed
```

The application periodically checks blockchain status until confirmation is achieved.

---

### 6.9 Confirmation States

Typical transaction states include:

Pending

↓

Confirmed

↓

Completed

If confirmation cannot be obtained,

the transaction remains pending until the blockchain reaches consensus.

---

### 6.10 Blockchain Authority

One of the key architectural principles is:

> **The blockchain always has higher authority than the application database.**

If inconsistencies occur,

the blockchain state takes precedence.

SQLite is updated to reflect blockchain reality,

never the opposite.

---

### 6.11 Verification Workflow

Verification follows this sequence.

```
SHA-256

↓

SQLite

↓

TXID

↓

Blockchain

↓

Confirmation

↓

Verification Result
```

This process can be repeated indefinitely.

Blockchain evidence remains permanently available.

---

### 6.12 Failure Handling

Possible blockchain failures include:

RPC unavailable

↓

Transaction rejected

↓

Network synchronization delay

↓

Temporary node restart

↓

Blockchain maintenance

Each condition is detected independently.

No completed notarization is declared until blockchain confirmation has been obtained.

---

### 6.13 Blockchain Independence

The Notary does not require:

- blockchain explorers;
- third-party APIs;
- hosted blockchain gateways; or
- cloud blockchain services.

Everything is performed locally.

This significantly improves long-term reliability.

---

### 6.14 Security Considerations

RPC credentials remain internal.

The blockchain node is never publicly accessible.

Only `server.js` communicates with `bitcoind-xec`.

All blockchain requests originate from trusted infrastructure.

---

### 6.15 Operational Checks

Recommended verification includes:

Verify node synchronization.

↓

Verify RPC connectivity.

↓

Verify transaction creation.

↓

Verify confirmation.

↓

Verify TXID lookup.

↓

Verify certificate generation.

Successful completion confirms correct blockchain integration.

---

### 6.16 Disaster Recovery

Following infrastructure recovery:

Restore

```
bitcoind-xec
```

↓

Verify synchronization.

↓

Restore SQLite.

↓

Restart `server.js`.

↓

Resume confirmation monitoring.

Because blockchain evidence is permanent,

only operational metadata require restoration.

---

### 6.17 Relationship with Mining

Although both services use the same blockchain node,

their responsibilities remain completely separate.

Mining infrastructure

↓

Produces blocks.

Blockchain Notary

↓

Creates notarization transactions.

Neither service depends on the internal logic of the other.

The blockchain node is the only shared production component.

---

### 6.18 Architectural Advantages

Using a locally managed blockchain node provides several benefits.

- complete infrastructure ownership;
- no third-party dependency;
- deterministic RPC responses;
- lower latency;
- improved privacy;
- simplified auditing;
- long-term operational stability.

These characteristics distinguish the LMWPool Notary from cloud-based notarization platforms.

---

### 6.19 Architectural Summary

The Blockchain Integration layer transforms validated document hashes into permanent, independently verifiable blockchain evidence.

By relying exclusively on a locally controlled eCash full node, the platform maintains full ownership of every blockchain operation while ensuring that every completed notarization can be verified directly on-chain without relying on external services.

This approach embodies one of the fundamental design principles of the XEC Blockchain Notary:

**trust the blockchain itself, never an intermediary.**

## 7. PDF Certificate Generation

### 7.1 Purpose

One of the distinguishing features of the XEC Blockchain Notary is the automatic generation of a **Blockchain Notarization Certificate**.

The certificate provides a human-readable document summarizing the blockchain evidence associated with a notarized file.

Although the blockchain itself remains the authoritative proof, the certificate offers users a convenient and portable representation of the notarization.

The PDF is therefore an operational artifact derived from blockchain evidence rather than the evidence itself.

---

### 7.2 Design Philosophy

The certificate has three primary objectives.

#### Readability

Present blockchain information in a clear and understandable format.

---

#### Portability

Allow users to archive, print and distribute the notarization proof.

---

#### Reproducibility

The same notarization should always generate the same blockchain information.

The certificate is therefore deterministic.

---

### 7.3 Certificate Lifecycle

The certificate is generated only after the notarization has been successfully completed.


**Workflow:**

```
Document

↓

SHA-256

↓

Blockchain Submission

↓

Confirmation

↓

Certificate Generation

↓

Storage

↓

Download
```

The certificate is never generated before blockchain registration.

---

### 7.4 Generation Process

The complete generation workflow is:

```
SQLite Metadata

+

Blockchain Information

↓

Template

↓

PDF Generator

↓

Certificate File

↓

Download Endpoint
```

The application combines operational metadata with blockchain evidence into a single document.

---

### 7.5 Certificate Contents

A production certificate typically includes:

- document filename;
- SHA-256 hash;
- notarization timestamp;
- blockchain name;
- blockchain transaction ID (TXID);
- verification URL;
- certificate generation timestamp;
- service identification.

Only information required for independent verification is included.

---

### 7.6 Document Integrity

The certificate does **not** contain the original document.

Instead it contains the SHA-256 digest representing that document.

Consequently:

Original document

↓

SHA-256

↓

Blockchain

↓

Certificate

Anyone possessing the original document can independently calculate its SHA-256 digest and compare it with the value printed on the certificate.

---

### 7.7 Certificate Storage

Generated certificates are stored locally.


**Typical workflow:**

```
Certificate Generated

↓

Filesystem

↓

Download Endpoint

↓

User
```

Because certificates can always be regenerated from SQLite metadata and blockchain information, they are considered reproducible artifacts.

---

### 7.8 Download Service

Certificates are made available through the REST API.


**Production endpoint:**

```
GET

/certificate/:hash.pdf
```


**Workflow:**

```
Browser

↓

REST Request

↓

server.js

↓

Filesystem

↓

PDF Response
```

If the certificate exists, it is returned immediately.

Otherwise, an appropriate error response is generated.

---

### 7.9 Relationship with SQLite

SQLite stores only certificate metadata.

Examples include:

Generation status.

Generation timestamp.

Associated notarization.

Download availability.

The PDF itself remains a filesystem object.

---

### 7.10 Relationship with the Blockchain

The certificate references blockchain evidence but does not replace it.


**Verification sequence:**

```
Certificate

↓

SHA-256

↓

TXID

↓

Blockchain

↓

Independent Verification
```

If the certificate were lost, the blockchain evidence would still remain valid.

---

### 7.11 Deterministic Generation

Certificate generation is deterministic.

The same notarization always produces identical blockchain information.

Dynamic values such as the certificate creation timestamp may differ, but the underlying evidence remains unchanged.

This supports long-term reproducibility of the underlying evidence.

---

### 7.12 Security Considerations

The certificate intentionally excludes sensitive operational information.

It must never contain:

- RPC credentials;
- SQLite paths;
- internal server addresses;
- User Keys;
- internal configuration;
- application secrets.

Only publicly verifiable information is included.

---

### 7.13 Failure Scenarios

Typical failures include:

Template unavailable

↓

Certificate generation fails.

---

Filesystem unavailable

↓

Certificate cannot be written.

---

Missing metadata

↓

Certificate incomplete.

---

Unexpected application error

↓

Generation aborted.

These failures never invalidate the blockchain notarization itself.

Only certificate availability is affected.

---

### 7.14 Operational Checks

Recommended validation procedure:

Create a notarization.

↓

Wait for blockchain confirmation.

↓

Generate certificate.

↓

Download PDF.

↓

Verify displayed SHA-256.

↓

Verify TXID.

↓

Verify verification URL.

A successful test validates the complete certificate workflow.

---

### 7.15 Recovery

If stored certificates are lost:

```
SQLite Metadata

+

Blockchain Information

↓

Regenerate PDF
```

Because certificates are reproducible, filesystem loss does not imply loss of blockchain evidence.

Only regeneration is required.

---

### 7.16 Architectural Advantages

The PDF subsystem provides several operational benefits.

- human-readable proof;
- portable documentation;
- printable evidence;
- deterministic generation;
- simple regeneration;
- direct blockchain references;
- long-term archival.

The certificate complements the blockchain while remaining independent from it.

---

### 7.17 Legal Considerations

The certificate should be regarded as a **representation of blockchain evidence**, not as the evidence itself.

The immutable proof resides permanently on the eCash blockchain.

The certificate merely summarizes that proof in a standardized format suitable for operational, administrative or documentary purposes.

---

### 7.18 Architectural Summary

The PDF Certificate Generation subsystem transforms blockchain metadata into a professional document suitable for distribution, archival and independent verification.

By separating certificate generation from the notarization process itself, the platform ensures that blockchain integrity remains the primary source of trust while providing users with an accessible and portable representation of their notarization.

This approach combines cryptographic reliability with practical usability, making the certificate one of the most visible features of the XEC Blockchain Notary.

## 8. Email Notification System

### 8.1 Purpose

The Email Notification System provides users with automatic updates throughout the notarization lifecycle.

Rather than requiring users to manually monitor the platform, the system proactively communicates important events, including:

- notarization submission;
- blockchain confirmation;
- certificate availability;
- operational notifications.

The email subsystem therefore acts as the primary communication channel between the Blockchain Notary and its users.

---

### 8.2 Design Philosophy

The notification system follows three fundamental principles.

#### Event-driven

Emails are generated only when meaningful events occur.

Examples include:

- successful submission;
- blockchain confirmation;
- certificate generation.

The platform never sends periodic or unnecessary messages.

---

#### Deterministic

Each completed notarization produces a predictable sequence of notifications.

The user experience remains consistent regardless of when the notarization occurs.

---

#### Independent

Email delivery is independent from blockchain registration.

A temporary SMTP failure must **never** invalidate a successful blockchain notarization.

---

### 8.3 Architecture

```
               server.js

                   │

           Business Event

                   │

                   ▼

            Email Generator

                   │

                   ▼

               SMTP Server

                   │

                   ▼

              Recipient Mailbox
```

The blockchain transaction is completed before email delivery is attempted.

---

### 8.4 Notification Workflow

The typical notification sequence is:

```
Submission

↓

Submission Email

↓

Blockchain Confirmation

↓

Certificate Generated

↓

Completion Email
```

This sequence allows users to follow the progress of their notarization from start to finish.

---

### 8.5 Submission Notification

Immediately after a notarization request is accepted, the user receives a confirmation indicating that:

- the request has been received;
- validation has succeeded;
- blockchain processing has started.

This message confirms that the platform has successfully accepted the notarization.

---

### 8.6 Confirmation Notification

Once the blockchain transaction reaches the required confirmation state,

the system generates a second notification.

This email typically includes:

- blockchain confirmation;
- transaction identifier;
- certificate availability;
- verification information.

This message marks the successful completion of the notarization process.

---

### 8.7 Administrative Notifications

The platform may also generate internal notifications.

Typical recipients include:

System Administrator

↓

Platform Owner

↓

Support Personnel

These notifications facilitate operational monitoring without requiring continuous manual supervision.

---

### 8.8 Email Contents

Production emails may include:

Document filename

↓

SHA-256 digest

↓

Blockchain

↓

TXID

↓

Verification URL

↓

Certificate download link

Only information useful to the recipient is included.

Sensitive operational data are intentionally excluded.

---

### 8.9 Relationship with the Certificate

The completion email may reference:

```
GET

/certificate/:hash.pdf
```

allowing users to download the generated certificate directly.

The certificate itself remains available independently of the email.

---

### 8.10 Relationship with SQLite

SQLite stores metadata describing email activity.

Examples include:

Notification status.

↓

Delivery timestamp.

↓

Associated notarization.

The mailbox itself is **not** considered part of the persistence layer.

---

### 8.11 SMTP Independence

The email subsystem is intentionally isolated from blockchain operations.

```
Blockchain

↓

Completed

↓

Attempt Email
```

If email delivery fails,

the blockchain notarization remains valid.

Only user communication is affected.

---

### 8.12 Failure Scenarios

SMTP unavailable

↓

Notification delayed or skipped.

---

Invalid recipient address

↓

Delivery rejected.

---

Temporary network failure

↓

SMTP communication interrupted.

---

Mail provider unavailable

↓

Notification postponed.

None of these failures invalidate the blockchain transaction.

---

### 8.13 Operational Checks

Recommended validation:

Create a test notarization.

↓

Verify submission email.

↓

Wait for blockchain confirmation.

↓

Verify completion email.

↓

Verify certificate link.

↓

Verify blockchain information.

Successful completion validates the notification subsystem.

---

### 8.14 Security Considerations

Emails must never contain:

- RPC credentials;
- User Keys;
- internal server paths;
- SQLite information;
- Docker configuration;
- application secrets.

Only information intended for the recipient should be transmitted.

---

### 8.15 Privacy Considerations

The email system follows the same privacy principles as the Notary.

The original uploaded document is never attached.

Only metadata are transmitted.

Examples include:

- filename;
- SHA-256 digest;
- TXID;
- verification URL.

This minimizes the exposure of user information.

---

### 8.16 Operational Advantages

The notification subsystem provides several benefits:

- immediate user feedback;
- asynchronous communication;
- automatic certificate delivery;
- reduced support requests;
- improved user experience;
- complete workflow visibility.

Because notifications are generated automatically, users remain informed without manually checking the platform.

---

### 8.17 Architectural Summary

The Email Notification System completes the user experience of the XEC Blockchain Notary by providing timely communication throughout the notarization lifecycle.

Its design intentionally separates blockchain operations from message delivery, ensuring that temporary email failures never compromise the integrity of the notarization itself.

This separation preserves the reliability of the platform while offering users a simple and transparent workflow from submission to completed blockchain certification.

## 9. User Management and Authentication

### 9.1 Purpose

The XEC Blockchain Notary implements a lightweight authentication model designed to balance usability and security.

Unlike traditional account-based platforms, the Notary does not require users to create passwords or maintain interactive sessions.

Instead, authentication is based on:

- email address;
- customer identifier;
- User Key.

This approach minimizes user friction while preventing unauthorized reuse of existing identities.

---

### 9.2 Design Philosophy

The authentication subsystem follows four principles.

#### Simplicity

The first notarization should require the minimum amount of information.

No separate account-creation, password-selection or login procedure is required.

---

#### Identity Persistence

Once a user has been registered,

future notarizations are associated with the same identity.

---

#### Stateless Requests

Each REST request contains all information required for authentication.

The server maintains no login session.

---

#### Server Authority

Authentication decisions are always made by

```
server.js
```

Client-side validation is considered informational only.

---

### 9.3 User Lifecycle

The typical lifecycle is:

```
First Request

↓

Unknown User

↓

Registration

↓

User Key Assigned

↓

Subsequent Requests

↓

Authentication

↓

Authorized Operations
```

This workflow keeps onboarding extremely simple.

---

### 9.4 User Registration

When an email address is encountered for the first time,

the platform creates a new user record.

Typical information includes the customer ID, email address, registration timestamp, generated User Key and initial status.

The registration process is automatic.

---

### 9.5 User Key

The **User Key** is the primary authentication mechanism used after registration.

Characteristics include:

- unique;
- server-generated;
- persistent;
- associated with a single user.

The User Key identifies the user without requiring passwords.

---

### 9.6 Authentication Workflow

For an existing user,

the request follows this sequence.

```
Incoming Request

↓

Email

↓

SQLite Lookup

↓

Existing User?

↓

YES

↓

Validate User Key

↓

Authorized
```

If validation fails,

the request is rejected.

---

### 9.7 New User Workflow

For users not yet present in SQLite:

```
Incoming Request

↓

Email

↓

SQLite Lookup

↓

User Not Found

↓

Create User

↓

Generate User Key

↓

Continue Notarization
```

This allows first-time users to begin using the platform immediately.

---

### 9.8 Existing User Workflow

For returning users:

```
Incoming Request

↓

Email

↓

User Record

↓

User Key Verification

↓

Continue Processing
```

No blockchain operation begins until authentication succeeds.

---

### 9.9 Authorization

Authentication determines identity.

Authorization determines permitted operations.

Typical protected operations include:

- creating notarizations;
- accessing personal history;
- retrieving user-specific information.

Public verification endpoints remain accessible without authentication.

---

### 9.10 Relationship with SQLite

SQLite stores authentication metadata.

Typical fields include the email address, customer identifier, User Key, registration date and status.

The database therefore acts as the authoritative user registry.

---

### 9.11 Relationship with REST API

Authentication occurs immediately after request validation.

```
REST Request

↓

Syntax Validation

↓

User Lookup

↓

Authentication

↓

Business Logic

↓

Blockchain
```

This avoids unnecessary blockchain operations for unauthorized requests.

---

### 9.12 Security Considerations

The User Key should be treated as confidential information.

Recommended practices include:

- never expose it publicly;
- never embed it in downloadable certificates;
- never include it in blockchain transactions;
- never disclose it through error messages.

The server always validates the User Key before processing privileged requests.

---

### 9.13 Privacy Considerations

Only the minimum information required for operation is stored.

The platform intentionally avoids collecting unnecessary personal information.

Typical stored data include:

- email address;
- customer identifier;
- registration timestamp;
- User Key.

No document contents are associated with the authentication subsystem.

---

### 9.14 Failure Scenarios

Unknown user

↓

Automatic registration.

---

Invalid User Key

↓

Authentication rejected.

---

Database unavailable

↓

Authentication impossible.

---

Malformed request

↓

Validation failure.

---

Unexpected server error

↓

Request aborted.

No blockchain operation is executed after an authentication failure.

---

### 9.15 Operational Checks

Recommended verification procedure:

Register a new user.

↓

Verify User Key generation.

↓

Submit a second notarization.

↓

Verify authentication.

↓

Retrieve history.

↓

Verify authorization rules.

Successful completion validates the authentication subsystem.

---

### 9.16 Architectural Advantages

The authentication model provides several benefits:

- no passwords;
- no login sessions;
- automatic registration;
- lightweight implementation;
- deterministic validation;
- minimal administrative overhead.

This design is particularly well suited to a blockchain notarization platform where user identity is secondary to document integrity.

---

### 9.17 Architectural Summary

The User Management subsystem provides a lightweight but effective authentication mechanism based on persistent user identities rather than traditional account management.

By combining automatic registration, SQLite-backed identity management and User Key validation, the platform achieves an excellent balance between usability, operational simplicity and security.

The result is an authentication model that integrates naturally with the REST architecture while remaining fully compatible with automated workflows and future enterprise integrations.

## 10. Verification Services

### 10.1 Purpose

The primary objective of the XEC Blockchain Notary is not simply to register document hashes but to allow **independent verification** of every completed notarization.

Verification can be performed at any time by any interested party without requiring access to the original notarization process.

This capability transforms the platform from a document registration service into a verifiable blockchain evidence system.

---

### 10.2 Verification Philosophy

The verification model follows one fundamental principle.

> **Trust the blockchain, not the application.**

The application provides convenient access to verification data.

The blockchain provides the actual evidence.

Even if the Notary service becomes unavailable, notarizations remain independently verifiable using blockchain information.

---

### 10.3 Verification Architecture

```
             Original Document

                     │

                     ▼

               SHA-256 Digest

                     │

                     ▼

               Verification API

                     │

                     ▼

                  SQLite

                     │

                     ▼

                  TXID

                     │

                     ▼

             eCash Blockchain

                     │

                     ▼

           Independent Evidence
```

Every verification ultimately terminates at the blockchain.

---

### 10.4 Verification Methods

The platform supports several verification methods.

#### REST API

```
GET /verify/:hash
```

---

#### PDF Certificate

Using the information printed on the certificate.

---

#### Blockchain Explorer

Using the stored transaction identifier.

---

#### Manual Verification

Recalculate the SHA-256 hash of the original document and compare it with the notarized value.

Each method leads to the same blockchain evidence.

---

### 10.5 Hash Verification

The most common workflow is based on the SHA-256 digest.

```
Original File

↓

SHA-256

↓

GET /verify/{hash}

↓

SQLite Lookup

↓

Blockchain Reference

↓

Verification Result
```

No upload of the original document is required.

---

### 10.6 REST Verification Endpoint


**Production endpoint:**

```
GET

/verify/:hash
```

Responsibilities include:

- verify existence;
- retrieve metadata;
- return blockchain references;
- return operational status;
- provide verification information.

This endpoint is intentionally public.

No authentication is required.

---

### 10.7 Verification Response

A successful verification typically returns:

SHA-256 digest

↓

Registration timestamp

↓

Blockchain

↓

Transaction ID

↓

Confirmation status

↓

Certificate availability

↓

Verification URL

The response intentionally excludes sensitive user information.

---

### 10.8 Certificate-Based Verification

Verification can also begin from the PDF certificate.

```
Certificate

↓

SHA-256

↓

TXID

↓

Blockchain

↓

Independent Confirmation
```

The certificate serves as a convenient entry point but not as the primary evidence.

---

### 10.9 Blockchain Verification

The strongest form of verification is performed directly against the blockchain.


**Workflow:**

```
TXID

↓

Blockchain Explorer

or

↓

bitcoind-xec RPC

↓

Transaction

↓

Confirmation

↓

Verified
```

This verification does not require the Notary platform to remain operational.

---

### 10.10 Independent Verification

Any third party possessing:

- the original document;
- the PDF certificate;
- or the TXID;

can independently confirm the notarization.

No privileged platform access is required.

This property is one of the principal advantages of blockchain-based notarization.

---

### 10.11 Verification without the Original File

The original document is not always available.

Alternative verification methods include:

Certificate

↓

Transaction ID

↓

Blockchain

↓

Timestamp

↓

Registration Evidence

Although document integrity cannot be re-evaluated without the original file, blockchain registration can still be demonstrated.

---

### 10.12 Privacy

The verification process intentionally reveals only operational metadata.

It never exposes:

- uploaded documents;
- document contents;
- User Keys;
- internal database identifiers;
- application configuration.

The SHA-256 digest acts as the document identifier.

---

### 10.13 Failure Scenarios

Unknown hash

↓

No notarization found.

---

Blockchain temporarily unavailable

↓

Verification delayed.

---

Database unavailable

↓

Metadata unavailable.

---

Certificate missing

↓

Blockchain verification still possible.

The blockchain remains the authoritative verification source.

---

### 10.14 Operational Checks

Recommended verification procedure.

Generate SHA-256.

↓

Submit notarization.

↓

Wait confirmation.

↓

Download certificate.

↓

Verify

```
GET /verify/:hash
```

↓

Verify blockchain transaction.

↓

Compare SHA-256 values.

Successful completion validates the complete verification subsystem.

---

### 10.15 Long-Term Verification

One of the principal design goals is long-term reproducibility.

Even years after notarization,

verification remains possible because:

- the blockchain is immutable;
- the SHA-256 algorithm is deterministic;
- the TXID uniquely identifies the blockchain transaction;
- certificates can be regenerated from stored metadata.

The evidence therefore survives application upgrades and infrastructure changes.

---

### 10.16 Relationship with Other Components

```
Original Document

↓

SHA-256

↓

Verification API

↓

SQLite

↓

TXID

↓

Blockchain

↓

Verification Result
```

Verification interacts with nearly every subsystem while maintaining the blockchain as the final authority.

---

### 10.17 Architectural Advantages

The verification subsystem provides several important benefits.

- independent validation;
- blockchain-backed evidence;
- multiple verification methods;
- privacy preservation;
- deterministic results;
- long-term reproducibility;
- interoperability with external systems.

These characteristics distinguish the XEC Blockchain Notary from conventional document management systems.

---

### 10.18 Architectural Summary

Verification is the final and most important stage of the notarization lifecycle.

While document submission creates blockchain evidence, verification demonstrates that the evidence can be independently reproduced and validated by any interested party.

By combining SHA-256 hashing, REST services, blockchain transactions and reproducible certificates, the platform ensures that every completed notarization remains verifiable long after the original submission, fulfilling the fundamental purpose of a blockchain-based trust service.

## 11. System Deployment and Production Operations

### 11.1 Purpose

The XEC Blockchain Notary is designed as a continuously available production service.

Unlike development environments, the production deployment prioritizes:

- reliability;
- recoverability;
- operational simplicity;
- long-term maintainability.

This chapter documents how the application is deployed, started and maintained in production.

---

### 11.2 Deployment Architecture

Unlike the mining platform, which primarily consists of Docker containers, the Notary is deployed directly on the Linux host.

```
Linux Host

│

├── server.js

├── SQLite

├── PDF Generator

├── SMTP

└── bitcoind-xec RPC
```

HTTPS publication is delegated to the shared Caddy reverse proxy.

---

### 11.3 Production Components

The operational deployment consists of:

| Component | Technology | Execution |
|---|---|---|
| `server.js` | Node.js | Host |
| SQLite | SQLite | Host |
| PDF Generator | Node.js | Host |
| SMTP Client | Node.js | Host |
| `bitcoind-xec` | Docker | Container |
| Caddy | Docker | Shared Reverse Proxy |

This mixed deployment model minimizes unnecessary containerization while maintaining architectural consistency.

---

### 11.4 Startup Sequence

Recommended startup order:

```
Linux

↓

bitcoind-xec

↓

Blockchain Synchronization

↓

server.js

↓

SQLite

↓

SMTP Verification

↓

REST API

↓

Caddy

↓

Production Ready
```

The blockchain node must always be available before the Notary starts accepting requests.

---

### 11.5 Shutdown Sequence

Recommended shutdown order:

```
Stop Incoming Requests

↓

server.js

↓

Pending Operations Complete

↓

SMTP

↓

bitcoind-xec
```

Allowing current requests to complete before shutdown minimizes the risk of interrupted notarizations.

---

### 11.6 Reverse Proxy Integration

The Notary is exposed through the shared Caddy instance.

Production route:

```
https://xec.lmwpool.com/notary/
```

Caddy removes the prefix

```
/notary
```

before forwarding the request to

```
server.js
```

The Node.js application therefore remains unaware of the external URL structure.

---

### 11.7 Service Availability

The production objective is continuous availability.

The following services should remain operational:

- [ ] REST API
- [ ] SQLite
- [ ] PDF generation
- [ ] Email notifications
- [ ] Blockchain RPC

Temporary failure of one auxiliary component should not compromise completed notarizations.

---

### 11.8 Logging

Operational logs should record:

- application startup;
- incoming requests;
- validation failures;
- blockchain submissions;
- certificate generation;
- email delivery; and
- unexpected exceptions.

Logs provide the primary source of operational diagnostics.

---

### 11.9 Backup Strategy

Mandatory backups include:

#### Application

```
server.js

package.json

templates/

public/
```

---

#### Database

```
data/notary.sqlite
```

---

#### Certificates

Generated PDF files.

---

#### Configuration

SMTP configuration.

Environment variables.

Systemd service.

Reverse proxy configuration.

---

### 11.10 Recovery Procedure

A typical recovery follows these steps.

Restore application files.

↓

Restore SQLite database.

↓

Verify permissions.

↓

Start `server.js`.

↓

Verify REST API.

↓

Verify blockchain RPC.

↓

Execute test notarization.

↓

Verify email.

↓

Verify certificate generation.

Only after successful validation should the service be considered operational.

---

### 11.11 Operational Monitoring

Routine production monitoring should verify:

- REST API responsiveness;
- SQLite accessibility;
- blockchain synchronization;
- RPC connectivity;
- SMTP availability;
- certificate generation;
- disk space;
- application logs.

These checks provide early detection of operational issues.

---

### 11.12 Failure Scenarios

#### Blockchain Node Unavailable


**Effect:**

New notarizations cannot be submitted.

Previously completed notarizations remain valid.

---

#### SQLite Failure


**Effect:**

Metadata unavailable.

Blockchain evidence remains intact.

---

#### SMTP Failure


**Effect:**

Notifications delayed.

Notarizations continue.

---

#### PDF Generation Failure


**Effect:**

Certificates unavailable.

Blockchain registration unaffected.

---

#### Caddy Failure


**Effect:**

Public HTTPS unavailable.

Internal application continues running.

---

### 11.13 Software Updates

Before updating the application:

Create SQLite backup.

↓

Backup application source.

↓

Stop `server.js`.

↓

Deploy new version.

↓

Restart application.

↓

Execute operational tests.

↓

Confirm API functionality.

Production updates should always be performed in a controlled sequence.

---

### 11.14 Security Recommendations

Recommended operational practices include:

- keep Node.js updated;
- rotate SMTP credentials periodically;
- protect SQLite backups;
- restrict filesystem permissions;
- secure RPC credentials;
- monitor application logs.

Operational security depends on disciplined maintenance rather than application logic alone.

---

### 11.15 Relationship with the Platform

Although deployed independently, the Notary integrates with the rest of the LMWPool ecosystem through only two shared components:

- `bitcoind-xec`, providing blockchain access.
- Caddy, providing HTTPS publication.

No dependency exists on:

- CKPool;
- Statistics API;
- Status Page;
- Binance utilities.

This architectural separation allows both systems to evolve independently.

---

### 11.16 Architectural Summary

The deployment model of the XEC Blockchain Notary intentionally favors simplicity over unnecessary abstraction.

By running the application directly on the Linux host while leveraging the existing blockchain node and shared reverse proxy, the platform achieves:

- straightforward deployment;
- minimal operational overhead;
- simplified disaster recovery;
- clear separation of responsibilities;
- reliable long-term operation.

This concludes the operational deployment documentation for the Blockchain Notary and prepares the remaining chapters covering administration, maintenance and future evolution.

## 12. Administration and Maintenance

### 12.1 Purpose

The Blockchain Notary has been designed to require minimal routine maintenance while providing continuous production availability.

This chapter describes the operational procedures required to administer, monitor and maintain the service over time.

Unlike software development activities, the procedures documented here are intended for production system administration.

---

### 12.2 Administrative Responsibilities

Routine administration includes:

- monitoring service availability;
- verifying blockchain connectivity;
- checking database integrity;
- validating certificate generation;
- supervising email delivery;
- maintaining backups;
- reviewing application logs.

These activities ensure long-term operational stability.

---

### 12.3 Daily Operational Checklist

The following checks are recommended each day.

#### Application

Verify

```
server.js
```

is running.

---

#### Blockchain

Confirm

```
bitcoind-xec
```

is synchronized.

---

#### Database

Verify

```
notary.sqlite
```

is accessible.

---

#### REST API

Execute a verification request.

```
GET /verify/{hash}
```

Confirm successful response.

---

#### Certificates

Download a recently generated certificate.

Verify:

- PDF readability;
- TXID;
- SHA-256;
- verification URL.

---

#### Email

Confirm recent notifications were successfully delivered.

---

### 12.4 Weekly Maintenance

Recommended weekly activities include:

Review application logs.

↓

Review disk usage.

↓

Verify SQLite integrity.

↓

Verify backup completion.

↓

Verify SMTP connectivity.

↓

Review failed requests.

These activities help identify developing operational issues.

---

### 12.5 Backup Procedures

The following assets require regular backup.

#### Critical

```
notary.sqlite

server.js

package.json

templates/

public/

certificates/
```

---

#### Configuration

SMTP configuration.

Environment variables.

Systemd service.

Caddy configuration.

---

#### Optional

Application logs.

Historical certificates.

Generated reports.

Because blockchain data are immutable, they do not require backup.

---

### 12.6 SQLite Maintenance

Recommended tasks include:

Database backup.

↓

Integrity verification.

↓

Space monitoring.

↓

Permission verification.

↓

Recovery testing.

SQLite maintenance is intentionally simple because the database consists of a single file.

---

### 12.7 Certificate Maintenance

Verify:

Generated PDFs.

↓

Download endpoint.

↓

Filesystem permissions.

↓

Storage capacity.

Old certificates may be archived if organizational policies require it.

---

### 12.8 Email Maintenance

Administrative checks include:

SMTP authentication.

↓

Delivery success.

↓

Bounce reports.

↓

Mailbox quotas.

↓

Administrative notifications.

Email failures should never interrupt blockchain processing.

---

### 12.9 Blockchain Maintenance

Verify:

Node synchronization.

↓

Wallet availability.

↓

RPC connectivity.

↓

Transaction confirmations.

↓

Blockchain storage capacity.

The Notary relies on the mining platform's blockchain node.

Maintaining the node therefore benefits both production systems.

---

### 12.10 Log Management

Recommended log review includes:

Application startup.

↓

Unhandled exceptions.

↓

REST errors.

↓

Blockchain failures.

↓

Certificate generation.

↓

SMTP failures.

↓

Unexpected warnings.

Logs should be archived according to operational requirements.

---

### 12.11 Performance Monitoring

Although the Notary has modest hardware requirements, the following metrics should be monitored:

- CPU utilization;
- memory usage;
- disk usage;
- SQLite size;
- REST response time; and
- certificate generation time.

These metrics help identify future scaling requirements.

---

### 12.12 Security Maintenance

Recommended activities include:

Rotate SMTP credentials.

↓

Protect SQLite backups.

↓

Review application permissions.

↓

Update Node.js.

↓

Review REST access logs.

↓

Verify reverse proxy configuration.

Security maintenance should become part of routine operational procedures.

---

### 12.13 Disaster Recovery

Following a system failure:

Restore application.

↓

Restore SQLite.

↓

Restore configuration.

↓

Verify blockchain node.

↓

Restart `server.js`.

↓

Execute validation tests.

↓

Confirm certificate generation.

↓

Confirm email delivery.

A complete recovery should always conclude with a successful test notarization.

---

### 12.14 Operational Validation

After any maintenance activity, execute the following validation sequence.

```
Create Test Notarization

↓

Verify REST Response

↓

Verify SQLite Record

↓

Verify Blockchain TX

↓

Verify Confirmation

↓

Generate Certificate

↓

Verify Email

↓

Download PDF

↓

Verify Hash
```

Only after completing this sequence should maintenance be considered successful.

---

### 12.15 Maintenance Philosophy

The Notary follows a preventive maintenance model rather than a reactive one.

Routine monitoring significantly reduces the probability of operational incidents.

Because every subsystem is largely independent,

maintenance can usually be performed on individual components without affecting the remainder of the platform.

---

### 12.16 Operational Documentation

Every production modification should be reflected in this manual.

Examples include:

- new REST endpoints;
- database schema changes;
- SMTP configuration changes;
- certificate layout updates;
- blockchain workflow modifications;
- security enhancements.

Keeping documentation synchronized with production is considered an integral part of platform maintenance.

---

### 12.17 Architectural Summary

The operational administration of the XEC Blockchain Notary has been intentionally simplified through the use of lightweight technologies, clearly separated responsibilities and deterministic workflows.

Routine maintenance primarily consists of verification rather than intervention.

This operational model minimizes downtime while ensuring that the platform remains reliable, auditable and maintainable throughout its lifecycle.

This chapter concludes the operational management section of the Blockchain Notary documentation and prepares the final chapters covering platform evolution, best practices and future development.

## 13. Security, Privacy and Compliance

### 13.1 Purpose

The XEC Blockchain Notary has been designed to provide cryptographic proof of document existence while preserving user privacy and minimizing operational risk.

Unlike traditional document repositories, the platform is **not intended to store documents**.

Its primary objective is to produce immutable blockchain evidence while keeping confidential information outside the blockchain.

Security and privacy are therefore fundamental architectural principles rather than optional features.

---

### 13.2 Security Objectives

The platform has five principal security objectives.

- protect user identity;
- protect document confidentiality;
- protect blockchain integrity;
- protect service availability; and
- protect operational credentials.

Every design decision supports one or more of these objectives.

---

### 13.3 Privacy by Design

The Notary follows a strict "Privacy by Design" model.

The original uploaded document is **never written to the blockchain**.

Only the following information becomes permanently recorded:

- SHA-256 digest;
- blockchain transaction;
- timestamp; and
- blockchain metadata.

The original document remains under the user's control.

---

### 13.4 Data Classification

The platform manages several categories of information.

| Information | Stored in SQLite | Stored on blockchain |
|---|---|---|
| Original document | No | No |
| SHA-256 hash | Yes | Yes |
| Email address | Yes | No |
| Customer ID | Yes | No |
| User Key | Yes | No |
| TXID | Yes | Yes |
| Certificate metadata | Yes | No |

This separation minimizes unnecessary disclosure of personal information.

---

### 13.5 Cryptographic Integrity

Document integrity relies on the SHA-256 hashing algorithm.


**Workflow:**

```
Document

↓

SHA-256

↓

Blockchain

↓

Verification
```

Any modification of the original document produces a completely different digest.

Consequently, even a one-byte modification invalidates the verification process.

---

### 13.6 Blockchain Immutability

Once recorded,

a notarization cannot be modified or deleted.

This immutability is inherited directly from the eCash blockchain.

The application itself has no capability to alter previously confirmed blockchain evidence.

---

### 13.7 Authentication Security

The platform authenticates returning users through:

- email address;
- User Key.

Passwords are intentionally avoided.

The User Key should be treated as confidential information and transmitted only through trusted communication channels.

---

### 13.8 RPC Security

Communication with

```
bitcoind-xec
```

occurs through authenticated JSON-RPC.

RPC credentials:

- remain local;
- are never exposed publicly;
- are never returned through REST responses;
- are never included in certificates.

The blockchain node is accessible only from trusted production components.

---

### 13.9 HTTPS Security

All public communication is performed through HTTPS.

```
Browser

↓

HTTPS

↓

Caddy

↓

server.js
```

TLS certificates are managed centrally by the shared Caddy reverse proxy.

Users never communicate directly with the Node.js application.

---

### 13.10 Email Security

Email notifications intentionally contain only operational information.

They should never include:

- RPC credentials;
- User Keys;
- internal filesystem paths;
- application configuration;
- server information.

Attachments should contain only user-facing documentation such as the PDF certificate.

---

### 13.11 Database Security

SQLite stores operational metadata only.

Recommended protections include:

- restricted filesystem permissions;
- encrypted backups;
- secure storage;
- periodic backup verification.

The database should never be publicly accessible.

---

### 13.12 Filesystem Security

Critical production directories include:

```
/root/xec_notary
```

```
/root/xec_ckpool
```

```
/root/xec_miningcore
```

Write access should be limited to authorized administrators.

Application files should not be modified directly in production without proper change management.

---

### 13.13 Operational Security

Routine operational practices include:

- regular backups;
- software updates;
- log review;
- disk monitoring;
- certificate verification;
- blockchain synchronization checks.

Operational discipline is considered an essential part of the platform's security model.

---

### 13.14 Privacy Considerations

The platform intentionally minimizes the amount of personal information stored.

Examples:

Original document

↓

Not stored

---

Document contents

↓

Not stored

---

Passwords

↓

Not stored

---

Only the information strictly required for service operation is retained.

This significantly reduces privacy risks.

---

### 13.15 Compliance Considerations

Because the platform stores metadata rather than document contents, it is generally easier to integrate into organizational compliance frameworks.

However, organizations deploying the Notary remain responsible for ensuring compliance with applicable regulations, including:

- data protection requirements;
- document retention policies;
- electronic record management;
- sector-specific legal obligations.

The Notary provides technical evidence but does not replace organizational governance.

---

### 13.16 Security Incident Response

In the event of a suspected security incident, the recommended response sequence is:

```
Identify

↓

Contain

↓

Preserve Logs

↓

Verify Blockchain

↓

Restore Service

↓

Review Security Controls
```

Because blockchain evidence is immutable, completed notarizations remain trustworthy even if operational systems require restoration.

---

### 13.17 Security Advantages

The security architecture provides several important benefits.

- local blockchain authority;
- privacy-preserving design;
- minimal data retention;
- deterministic verification;
- isolated authentication;
- secure RPC communication;
- centralized HTTPS management.

These characteristics significantly reduce the attack surface of the platform.

---

### 13.18 Architectural Summary

Security and privacy are foundational aspects of the XEC Blockchain Notary.

By combining local blockchain infrastructure, SHA-256 cryptographic hashing, minimal metadata storage, authenticated RPC communication and centralized HTTPS protection, the platform delivers strong technical guarantees while preserving user confidentiality.

The result is a notarization system that provides immutable blockchain evidence without exposing sensitive document contents, maintaining a clear separation between operational metadata and cryptographic proof.

## 14. Disaster Recovery and Business Continuity

### 14.1 Purpose

The XEC Blockchain Notary has been designed so that no single software component represents the sole repository of notarization evidence.

The platform combines:

- local operational metadata;
- immutable blockchain records;
- reproducible certificates.

This architecture significantly simplifies disaster recovery while preserving the integrity of completed notarizations.

The objective of this chapter is to document the procedures required to restore the service after hardware failures, software failures or complete server replacement.

---

### 14.2 Disaster Recovery Philosophy

The recovery strategy is based on one fundamental principle.

> **The blockchain is permanent. The platform is reproducible.**

Even if the application server is completely lost,

the blockchain still contains the cryptographic evidence.

Only operational metadata must be restored.

---

### 14.3 Critical Assets

The following assets must be protected.

#### Application

```
server.js

package.json

templates/

public/

utilities/
```

---

#### Database

```
notary.sqlite
```

---

#### Configuration

- environment variables;
- SMTP configuration;
- systemd service; and
- Caddy configuration.

---

#### Certificates

Generated PDF certificates are optional backup assets because they are reproducible.

---

#### Blockchain

No Notary-specific blockchain backup is required.

The eCash blockchain is reconstructed by the full node.

---

### 14.4 Recovery Priorities

Recovery should follow this priority.

```
Linux Server

↓

bitcoind-xec

↓

Blockchain Synchronization

↓

SQLite

↓

server.js

↓

SMTP

↓

Caddy

↓

Operational Validation
```

Restoring the blockchain node is the highest priority because every notarization depends upon it.

---

### 14.5 Complete Recovery Procedure

A complete infrastructure rebuild follows these steps.

#### Step 1

Install Linux.

---

#### Step 2

Install Docker.

---

#### Step 3

Restore

```
bitcoind-xec
```

---

#### Step 4

Wait until blockchain synchronization completes.

---

#### Step 5

Restore

```
/root/xec_notary
```

---

#### Step 6

Restore

```
notary.sqlite
```

---

#### Step 7

Verify permissions.

---

#### Step 8

Start

```
server.js
```

---

#### Step 9

Verify REST API.

---

#### Step 10

Execute a production validation test.

---

### 14.6 Database Recovery

SQLite recovery is straightforward.

```
Restore

↓

notary.sqlite

↓

Verify Permissions

↓

Restart Application
```

The application automatically reconnects to the restored database.

---

### 14.7 Certificate Recovery

Certificates are not irreplaceable.

If PDF files are lost:

```
SQLite

+

Blockchain

↓

Regenerate

↓

PDF
```

For this reason,

certificate regeneration is preferred over restoring outdated copies.

---

### 14.8 Blockchain Recovery

The blockchain node is restored independently from the Notary.


**Workflow:**

```
bitcoind-xec

↓

Synchronization

↓

RPC Verification

↓

Notary Startup
```

The Notary should never start before the blockchain node is operational.

---

### 14.9 Configuration Recovery

Configuration files should be restored before starting the application.

Examples include:

SMTP

↓

Environment variables

↓

Systemd

↓

Caddy

↓

Application settings

Incorrect configuration often causes more downtime than missing application code.

---

### 14.10 Operational Validation

After recovery,

execute the following validation sequence.

```
REST API

↓

Create Test Notarization

↓

Blockchain TX

↓

Confirmation

↓

Certificate

↓

Email

↓

Verification

↓

History
```

Only after successful completion should the service return to production.

---

### 14.11 Recovery Time Objective

The platform has been intentionally designed to minimize recovery effort.

Typical recovery activities include:

- restoring a single SQLite database;
- restoring application files;
- reconnecting to the blockchain node.

Because no complex database cluster exists,

recovery procedures remain relatively short.

Actual recovery time depends primarily on blockchain node availability.

---

### 14.12 Business Continuity

During temporary outages:

Previously completed notarizations remain valid.

↓

Blockchain evidence remains available.

↓

Existing certificates remain valid.

↓

Verification remains possible using the TXID.

Consequently,

temporary service interruption does **not** invalidate previous notarizations.

---

### 14.13 Failure Classification

#### Infrastructure Failure

Examples:

Disk failure

Power outage

Hardware replacement

Recovery:

Restore infrastructure.

---

#### Application Failure

Examples:

Node.js crash

Configuration error

Software bug

Recovery:

Restart application.

---

#### Database Failure

Examples:

SQLite corruption

Filesystem damage

Recovery:

Restore backup.

---

#### Blockchain Failure

Examples:

Node restart

Synchronization delay

Recovery:

Restore `bitcoind-xec`.

Wait for synchronization.

---

### 14.14 Backup Verification

Backups should be periodically tested.

Recommended validation includes:

Restore SQLite.

↓

Start `server.js`.

↓

Verify REST API.

↓

Generate certificate.

↓

Verify blockchain lookup.

A backup that has never been tested should not be considered reliable.

---

### 14.15 Business Continuity Principles

The platform achieves business continuity through several architectural choices.

- Local blockchain authority.
- Lightweight SQLite database.
- Stateless REST architecture.
- Deterministic certificate generation.
- Independent verification.
- Minimal external dependencies.

These principles significantly reduce operational risk.

---

### 14.16 Architectural Summary

The disaster recovery strategy of the XEC Blockchain Notary intentionally separates **recoverable operational assets** from **permanent blockchain evidence**.

Application code, SQLite metadata and configuration can be restored from backups, while cryptographic proof remains permanently preserved on the eCash blockchain.

This architecture provides a high level of operational resilience and ensures that completed notarizations remain trustworthy even in the event of major infrastructure failures.

---

### 14.17 Final Conclusions

The XEC Blockchain Notary demonstrates how a relatively compact software architecture can provide enterprise-grade blockchain notarization services without relying on external blockchain providers or cloud-based trust services.

Its architecture combines:

- local blockchain ownership;
- deterministic cryptographic verification;
- lightweight operational infrastructure;
- reproducible certificates;
- REST-based integration;
- privacy-preserving design;
- simplified disaster recovery.

Together with the mining platform documented in Volume 1, the Notary forms the second major production system of the LMWPool ecosystem.

This concludes **Volume 3 — XEC Blockchain Notary**.


---

## Release 1.0 Editorial Notes

Release 1.0 consolidates the Volume 3 RC2 source without changing the documented production architecture or operational configuration.

Main improvements:

- standardized document heading and volume naming;
- normalized technical terminology across the Notary platform;
- improved consistency with Volumes 1 and 2;
- preserved the documented production architecture without functional changes;
- retained detailed command references, configuration listings and operational runbooks in Appendices A–F.
