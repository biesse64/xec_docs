# LMWPool XEC Platform
## Volume 1 — XEC Mining Platform

| Document metadata | Value |
|---|---|
| Document | Technical Architecture and Operations Manual |
| Release | 1.0 |
| Author | Bruno Stefanutti |
| Platform | eCash (XEC) |
| Environment | Production |
| Status | Definitive release |

> **Document scope.** This volume documents the production XEC mining architecture, its core infrastructure, security boundaries, configuration assets and first-line operational procedures. Detailed support-service internals, the Blockchain Notary and the full infrastructure runbook are documented in Volumes 2, 3 and 4.

---

## Document Control

> **Release note:** This document is Volume 1 of the LMWPool Technical Documentation Release 1.0 set. Read it together with Volumes 2–4 and Appendices A–F for the complete platform reference.

| Field | Value |
|---|---|
| Release | 1.0 |
| Source baseline | Volume 1 RC2 |
| Intended audience | Platform owner, system administrators, technical maintainers |
| Change policy | Production changes must be reflected in the relevant manual volume |
| Architectural authority | Running production configuration and verified operational evidence |

## Table of Contents

**Document conventions**

- **Architecture** sections describe design decisions.
- **Implementation** sections describe the deployed solution.
- **Operational Notes** describe production procedures.


1. [Executive Summary](#1-executive-summary)
2. [High-Level Architecture](#2-high-level-architecture)
3. [XEC Mining Platform](#3-xec-mining-platform)
4. [Architectural Principles](#4-architectural-principles)
5. [Physical Server Architecture](#5-physical-server-architecture)
6. [Docker Architecture](#6-docker-architecture)
7. [Docker Networks](#7-docker-networks)
8. [`bitcoind-xec`](#8-bitcoind-xec)
9. [CKPool Solo Mining Engine](#9-ckpool-solo-mining-engine)
10. [End-to-End Mining Workflow](#10-end-to-end-mining-workflow)
11. [Caddy Reverse Proxy](#11-caddy-reverse-proxy)
12. [Security Architecture](#12-security-architecture)
13. [Production Configuration Files](#13-production-configuration-files)
14. [Operations Runbook](#14-operations-runbook)

---

## 1. Executive Summary

### 1.1 Purpose

This document describes the production architecture of the **LMWPool XEC mining platform** and its boundaries with the platform's shared services and blockchain notarization service.

Unlike generic cryptocurrency mining pools, the platform has been engineered around a minimalist and highly reliable architecture whose primary objectives are:

- maximize reliability;
- minimize operational complexity;
- retain full control of the blockchain node;
- avoid unnecessary middleware;
- support complete disaster recovery; and
- maintain direct ownership of every software component.

This document is the official technical reference for installation, maintenance, troubleshooting and future evolution of the production mining platform.

---

### 1.2 Platform Scope

The overall XEC platform consists of two logically distinct systems.

#### XEC Mining Platform

The XEC mining platform is the production infrastructure dedicated to eCash solo mining.

`bitcoind-xec` is the Docker deployment of the official eCash Core full node.

Components include:

- Full eCash node (`bitcoind-xec`)
- CKPool Solo
- Statistics API
- Status Page
- Caddy Reverse Proxy
- Monitoring utilities
- Block parser
- Telegram notification system

---

#### XEC Blockchain Notary

The XEC Blockchain Notary is an independent production service that provides blockchain notarization.

Components include:

- Node.js server
- SQLite database
- REST API
- PDF certificate generation
- Email notifications
- Confirmation monitor
- Blockchain integration

Both systems share the same production VPS while remaining logically independent.

The complete Notary architecture is documented in Volume 3, *XEC Blockchain Notary*.

---

### 1.3 Design Philosophy

The project follows the architectural principles described below.

#### Simplicity

Every component has one specific responsibility.

Examples:

- CKPool performs only mining.
- `bitcoind-xec` maintains blockchain consensus.
- Statistics API generates mining statistics.
- Status Page renders the public interface.
- Caddy acts only as a reverse proxy.
- The Notary remains logically separate from the mining platform.

---

#### Direct Blockchain Interaction

Every blockchain operation is executed directly against the local eCash full node.

No third-party blockchain services are used.

This provides:

- independence;
- reproducibility;
- security; and
- complete ownership of blockchain data.

---

#### Production-Oriented Design

The infrastructure prioritizes production operation over convenience.

Wherever possible, the design favors:

- fewer moving parts;
- deterministic behavior;
- explicit configuration;
- no hidden automation; and
- straightforward disaster recovery.

---

#### Documentation Philosophy

This manual documents the deployed production platform as it actually operates; it is not a conceptual reference architecture.

Every configuration, path, service relationship and operational procedure shown here is derived from the running production infrastructure or verified operational evidence. Hypothetical examples are intentionally excluded.

---

## 2. High-Level Architecture

The production platform is composed of two logical environments with distinct public traffic paths.

```text
                              Internet
                                 │
                ┌────────────────┴────────────────┐
                │                                 │
              HTTPS                            Stratum
                │                                 │
                ▼                                 ▼
        +---------------+                 +---------------+
        |  caddy-btc    |                 |  ckpool-xec   |
        | HTTPS ingress |                 | Mining server |
        +-------+-------+                 +-------+-------+
                │                                 │
                ▼                                 │ RPC / ZMQ
   +--------------------------+                   │
   | HTTPS backend services   |                   │
   | Status / APIs / Notary   |                   │
   +------------+-------------+                   │
                │                                 │
                │ RPC where required              │
                └────────────────┬────────────────┘
                                 ▼
                         +---------------+
                         | bitcoind-xec  |
                         | eCash Core    |
                         +---------------+
```

HTTPS traffic is reverse proxied by Caddy to the appropriate backend service. Stratum traffic connects directly to `ckpool-xec` and never passes through Caddy. Backend services interact with `bitcoind-xec` only where their responsibilities require blockchain access.

The mining platform and Blockchain Notary share the production VPS while remaining logically independent.

---

## 3. XEC Mining Platform

The production mining stack consists of the following services.

| Component | Technology | Purpose | Availability |
|---|---|---|---|
| `bitcoind-xec` | eCash Core | Full blockchain node | Internal RPC and ZMQ |
| `ckpool-xec` | CKPool | Solo-mining server | Public Stratum endpoints |
| `stats-api-xec` | Python | Mining statistics | Internal; exposed through Caddy routes |
| `status-page-xec` | NGINX | Public web interface | Internal; exposed through Caddy |
| `caddy-btc` | Caddy v2 | HTTPS reverse proxy | Public HTTPS ingress |

Additional utilities include:

- `parse_block.py`
- `block_notify_xec.sh`
- health scripts
- maintenance utilities

---

## 4. Architectural Principles

The mining platform intentionally avoids Miningcore.

Although historical directories still contain the name:

```
/root/xec_miningcore
```

Miningcore is **not used** for XEC mining.

The directory name has been retained only because it now hosts utilities, scripts and status page resources developed during the evolution of the platform.

The production mining path consists exclusively of:

```
CKPool
        +
bitcoind-xec
```

CKPool generates mining jobs and processes shares while communicating directly with the local eCash full node, which performs authoritative block and transaction validation.

---

## 5. Physical Server Architecture

### 5.1 Overview

The LMWPool XEC platform is deployed on a single Linux VPS hosting both the mining infrastructure and the blockchain notarization service.

The architecture intentionally centralizes all production components on one server in order to:

- simplify maintenance;
- reduce infrastructure costs;
- minimize network latency between CKPool and the blockchain node;
- keep complete operational control.

Despite sharing the same host, the mining platform and the notarization service are logically separated.

---

### 5.2 High-Level Component Layout

```text
+-------------------------------------------------------------+
|                    Linux VPS (Production)                   |
|                                                             |
|  Docker                                                    |
|   ├── bitcoind-xec                                         |
|   ├── ckpool-xec                                           |
|   ├── stats-api-xec                                        |
|   ├── status-page-xec                                      |
|   └── caddy-btc (shared reverse proxy)                     |
|                                                             |
|  Native Services                                           |
|   ├── xec_notary (Node.js / systemd)                        |
|   ├── block_notify_xec.sh                                  |
|   └── scheduled maintenance utilities                      |
|                                                             |
+-------------------------------------------------------------+
```

---

### 5.3 Production Containers

Current production containers relevant to the XEC platform are:

| Container | Runtime / Image | Purpose | Criticality |
|---|---|---|---|
| `bitcoind-xec` | Ubuntu 22.04 with eCash Core binaries | Authoritative eCash full node | Critical |
| `ckpool-xec` | Locally built eCash CKPool image | Solo-mining Stratum server | Critical |
| `stats-api-xec` | `python:3.12-alpine` | Mining statistics service | Supporting |
| `status-page-xec` | `nginx:alpine` | Public XEC web interface | Supporting |
| `caddy-btc` | `caddy:2` | Shared HTTPS reverse proxy | Shared / High |

Other containers (BTC, Prometheus, Grafana, PostgreSQL, etc.) belong to the Bitcoin infrastructure and are outside the scope of this manual except where they provide shared infrastructure (e.g. Caddy).

---

## 6. Docker Architecture

### 6.1 Philosophy

Docker is used to isolate each production service.

Each container performs one specific function:

- blockchain node;
- mining server;
- statistics;
- web interface;
- reverse proxy.

This modular architecture allows individual services to be restarted or upgraded without affecting unrelated components.

---

### 6.2 CKPool Deployment

Production compose file:

`/root/xec_ckpool/docker-compose-ckpool-xec.yml`

Main characteristics:

- dedicated CKPool image built locally;
- configuration mounted read-only;
- persistent log directory;
- automatic restart;
- connection to external Docker networks.

The deployment publishes two Stratum endpoints:

| External port | Internal port | Miner profile |
|---|---|---|
| `7333` | `3333` | Standard Variable Difficulty endpoint |
| `7444` | `4444` | High Difficulty endpoint |

---

### 6.3 Configuration Volumes

Configuration is intentionally mounted read-only.

Example:

`./config/ckpool-xec.conf`

↓

`/config/ckpool-xec.conf`

This guarantees that production configuration cannot accidentally be modified from inside the running container.

Mining logs are instead persisted using a writable volume.

---

## 7. Docker Networks

The XEC platform relies on external Docker networks.

The CKPool deployment joins:

- `xec_miningcore_xec_internal`
- `poolnet`

The `poolnet` network is particularly important because it provides communication between independently managed Docker projects.

It allows services such as:

- `caddy-btc`
- `stats-api-xec`
- `status-page-xec`

to communicate without belonging to the same Docker Compose project.

This architectural choice keeps the XEC stack independent while allowing shared public infrastructure.
## 8. `bitcoind-xec`

### 8.1 Purpose

| Attribute | Production value |
|---|---|
| Container | `bitcoind-xec` |
| Role | Authoritative eCash full node |
| RPC endpoint | `bitcoind-xec:8332` |
| ZMQ endpoint | `tcp://bitcoind-xec:28332` |
| Primary dependent services | CKPool, block parser, wallet monitor, Blockchain Notary |
| Criticality | Critical |

`bitcoind-xec` is the authoritative blockchain component of the LMWPool XEC platform.

Unlike traditional mining pools that may rely on external blockchain services or remote nodes, LMWPool maintains a **fully synchronized local eCash full node**.

Every blockchain operation performed by the platform ultimately depends on this node.

Responsibilities include:

- maintaining consensus with the eCash network;
- validating every received block;
- generating block templates for CKPool;
- exposing RPC services;
- publishing ZMQ notifications;
- maintaining the mining wallet;
- broadcasting transactions;
- supporting the Blockchain Notary.

The full node is therefore the authoritative source of blockchain state for every blockchain operation performed by the platform.

---

### 8.2 Deployment

The node runs inside the Docker container

```
bitcoind-xec
```

Unlike CKPool, which is built locally, this container executes the official eCash Core binaries.

The node is permanently online and continuously synchronized with the public eCash blockchain.

The platform is intentionally designed around **one authoritative node**.

No secondary node is currently used.

---

### 8.3 Platform Responsibilities

Within the mining platform, `bitcoind-xec` is responsible for:

- blockchain synchronization;
- transaction validation;
- mempool maintenance;
- mining-template generation;
- block submission;
- RPC services;
- wallet management; and
- ZMQ publication.

No pool or Stratum logic is implemented inside `bitcoind-xec` itself.

Mining remains entirely delegated to CKPool.

---

### 8.4 RPC Interface

CKPool communicates exclusively through the JSON-RPC interface.

Current production configuration:

```
bitcoind-xec:8332
```

Authentication is configured through:

```
auth
pass
```

defined inside

```
ckpool-xec.conf
```

RPC is never exposed publicly.

Only trusted internal services can reach the interface.

Primary RPC methods used across the platform include:

- `getblocktemplate`;
- `submitblock`;
- `getblock`;
- `getblockhash`;
- `getwalletinfo`;
- `listtransactions`;
- `gettransaction`; and
- `getrawtransaction`.

These calls are used by different platform components.

Example:

```text
CKPool

↓

getblocktemplate

↓

Mining Job

↓

Miner

↓

Share

↓

submitblock

↓

bitcoind-xec

↓

Network
```

---

### 8.5 Wallet

Unlike traditional pools that collect rewards centrally before redistribution, the LMWPool architecture minimizes wallet responsibilities.

The wallet is primarily required for:

- block monitoring;
- notification services;
- transaction lookup;
- Blockchain Notary;
- administrative operations.

The notification subsystem continuously polls wallet activity through:

```
listtransactions
```

allowing immediate identification of newly generated rewards.

Wallet monitoring is implemented independently from CKPool.

This keeps notification logic independent from CKPool process state; notifications still depend on the wallet-monitoring service itself remaining active.

---

### 8.6 ZMQ Interface

The node exposes a ZeroMQ publisher.

Current production endpoint:

```
tcp://bitcoind-xec:28332
```

CKPool subscribes to this endpoint to receive immediate blockchain notifications.

Advantages:

- no polling

- immediate block detection

- reduced latency

- efficient template refresh

This architecture ensures miners always receive work generated from the latest blockchain tip.

---

### 8.7 Synchronization

A production node must always satisfy:

```
blocks == headers
```

and

```
initialblockdownload = false
```

The platform periodically verifies synchronization status before considering the node operational.

Mining is never considered reliable while Initial Block Download is active.

---

### 8.8 Failure Modes

The following situations are considered critical:

| Failure | Operational effects |
|---|---|
| Node stopped | CKPool cannot generate work; miners eventually disconnect; no new blocks can be found through the platform. |
| RPC unavailable | CKPool may fail to start; parsers fail; wallet functions are unavailable. |
| Blockchain desynchronization | Templates may become stale; blocks may be rejected; mining work may be invalid. |
| Wallet unavailable | Notifications and block monitoring are unavailable; the Notary is partially degraded. |

---

### 8.9 Recovery

Recovery procedure:

1. Verify Docker container status.

```
docker ps
```

2. Verify blockchain synchronization.

```
getblockchaininfo
```

3. Verify wallet.

```
getwalletinfo
```

4. Verify RPC.

```
getblockcount
```

5. Verify ZMQ endpoint.

6. Verify CKPool reconnect.

Only after all six checks succeed should production mining resume.

---

### 8.10 Relationship with CKPool

The relationship between the two services is intentionally simple.

```
              +----------------+
              | bitcoind-xec   |
              +----------------+
                    ▲
      RPC           │         ZMQ
                    │
                    ▼
              +----------------+
              | CKPool         |
              +----------------+
                    │
                    ▼
              Connected miners
```

`bitcoind-xec` never communicates directly with miners.

Every mining connection terminates inside CKPool.

---

### 8.11 Relationship with Other Services

In addition to CKPool, the following platform components access `bitcoind-xec`.

#### `parse_block.py`

Uses RPC to verify mined blocks directly on-chain.

Functions:

- `getblockhash`
- `getblock`

---

#### `block_notify_xec.sh`

Uses wallet RPC.

Functions include:

- `listtransactions`
- `getwalletinfo`

---

#### Blockchain Notary

Uses RPC for blockchain notarization and confirmation monitoring.

Although logically independent from mining, the Notary relies on the same authoritative blockchain node.

---

### 8.12 Operational Considerations

The decision to maintain a local full node instead of relying on third-party APIs provides several strategic advantages:

- complete blockchain sovereignty;
- no external dependencies;
- deterministic behavior;
- lower latency;
- higher reliability;
- improved disaster recovery.

This design philosophy is consistently applied throughout the entire LMWPool platform and represents one of its fundamental architectural principles.
## 9. CKPool Solo Mining Engine

### 9.1 Overview

CKPool is the core mining engine of the LMWPool XEC platform.

Unlike Miningcore or traditional share-based pools, CKPool has been configured as a **true solo-mining pool**, where miners compete independently to solve an entire blockchain block.

The software is responsible exclusively for:

- accepting Stratum connections;
- generating mining jobs;
- validating submitted shares;
- submitting solved blocks to the local eCash node;
- maintaining mining sessions;
- logging mining activity.

It does **not**:

- maintain balances;
- calculate payouts;
- distribute rewards;
- manage pool accounts.

Those functions are intentionally absent from the platform.

---

### 9.2 Design Philosophy

The architecture intentionally minimizes software layers.

```text
+------------------+
| Miner / ASIC     |
+--------+---------+
         | Stratum
         v
+------------------+
| ckpool-xec       |
+--------+---------+
         | JSON-RPC / ZMQ
         v
+------------------+
| bitcoind-xec     |
+--------+---------+
         | P2P
         v
+------------------+
| eCash Network    |
+------------------+
```

No additional middleware is inserted into the mining path.

The result is:

- lower latency;
- fewer failure points;
- deterministic behavior;
- simpler disaster recovery.

---

### 9.3 Production Deployment

Container:

```
ckpool-xec
```

Compose file:

```
/root/xec_ckpool/docker-compose-ckpool-xec.yml
```

Configuration:

```
/root/xec_ckpool/config/ckpool-xec.conf
```

Log directory:

```
/root/xec_ckpool/logs
```

Restart policy:

```
unless-stopped
```

---

### 9.4 Docker Compose

Production deployment builds CKPool locally.

```yaml
build:
  context: ./ecash-ckpool-solo
```

The executable is started with:

```bash
/src/src/ckpool \
    -x \
    -B \
    -c /config/ckpool-xec.conf
```

The command invokes the CKPool binary with the production-specific runtime flags and an explicit configuration file. The exact semantics of the locally built CKPool flags must be verified against the deployed source tree before changing them.

---

### 9.5 Configuration

The production configuration is intentionally compact and exposes only the parameters required for stable operation.

Important parameters include:

Blockchain node

```
bitcoind-xec:8332
```

Wallet address

```
btcaddress
```

Pool signature

```
btcsig
```

Donation

```
1%
```

Log directory

```
/logs
```

ZMQ endpoint

```
tcp://bitcoind-xec:28332
```

---

### 9.6 Stratum Endpoints

The production environment exposes two public mining ports.

#### Standard Variable Difficulty endpoint

External:

```
7333
```

Internal:

```
3333
```

Target users:

- Bitaxe
- USB miners
- low-hashrate ASICs
- hobby miners

---

#### High Difficulty endpoint

External:

```
7444
```

Internal:

```
4444
```

Target users:

- Antminer
- Whatsminer
- enterprise ASICs

The second endpoint supports a significantly higher share-difficulty profile, reducing unnecessary network traffic for powerful hardware.

---

### 9.7 Direct Miner Payout

One of the distinguishing characteristics of the platform is that rewards are **not collected by the pool**.

Each miner connects using **its own eCash address**.

Example worker:

```
ecash:q....
```

or

```
ecash:q....worker1
```

When a valid block is found:

1. CKPool constructs the block template.

2. Coinbase is generated.

3. The winning miner address is inserted.

4. Block submitted.

5. Network validates.

6. Coinbase reward belongs directly to that miner.

Therefore:

The pool does **not** operate a centralized reward wallet for miners.

---

### 9.8 Pool Donation

Although rewards belong to the successful miner, the pool retains its configured donation.

Production configuration:

```
donation = 1.0
```

This mechanism is implemented directly by CKPool.

It allows infrastructure costs to be partially covered while preserving the simplicity of the solo-mining pool model.

---

### 9.9 Block Solving Workflow

```
Miner

│

Share

│

CKPool

│

Share Validation

│

Candidate Block

│

submitblock()

│

bitcoind-xec

│

Network Consensus

│

Solved Block
```

If accepted:

- CKPool logs the event;
- `parse_block.py` later validates the block independently;
- `block_notify_xec.sh` monitors wallet activity;
- Telegram notification is generated.

---

### 9.10 Logging

Primary production log:

```
/root/xec_ckpool/logs/ckpool.log
```

The log represents the authoritative operational history of the mining engine.

Typical entries include:

- miner connection;
- worker authorization;
- accepted shares;
- rejected shares;
- difficulty changes;
- solved blocks;
- startup;
- shutdown;
- RPC failures.

Several platform utilities depend directly on this log.

---

### 9.11 Solved Blocks

Successful blocks appear in the log with entries similar to:

```
Solved and confirmed block XXXXX
```

These entries are later processed by:

```
parse_block.py
```

The parser independently verifies every block through RPC before publishing it on the public Status Page.

This additional verification avoids false positives originating from incomplete or invalid mining events.

---

### 9.12 Failure Scenarios

Typical CKPool failures include:

#### RPC unavailable

Symptoms:

- startup failure;
- no mining jobs.

---

#### ZMQ unavailable

Symptoms:

- delayed template updates;
- stale work.

---

#### Lost Docker network

Symptoms:

- node unreachable;
- mining interruption.

---

#### Configuration error

Symptoms:

- immediate process termination;
- container restart loop.

---

#### Invalid miner submissions

Symptoms:

- rejected shares;
- reduced effective hashrate.

---

### 9.13 Operational Checks

Routine verification should confirm:

- the container is running (`docker ps`);
- the container log is available (`docker logs ckpool-xec`);
- real-time mining activity appears in `logs/ckpool.log`;
- network connectivity is intact;
- RPC is reachable;
- ZMQ is active;
- shares are being accepted; and
- solved-block history is available.

---

### 9.14 Relationship with Other Components

CKPool exchanges information with only a small number of services.

#### `bitcoind-xec`

Provides RPC and ZMQ interfaces.

---

#### `parse_block.py`

Reads CKPool logs.

---

#### `block_notify_xec.sh`

Operates independently, using wallet monitoring rather than CKPool logs.

---

#### Status Page

Consumes processed mining information rather than raw CKPool data.

---

### 9.15 Architectural Advantages

The production architecture deliberately avoids unnecessary complexity.

Advantages include:

- direct blockchain interaction;
- no accounting subsystem;
- no payout scheduler;
- deterministic mining behavior;
- reduced maintenance;
- simplified disaster recovery;
- minimal attack surface;
- excellent observability through log-based monitoring.

This design has been validated through production operation and represents one of the defining characteristics of the LMWPool XEC platform.
## 10. End-to-End Mining Workflow

### 10.1 Purpose

This chapter describes the complete operational workflow of a mining job from the moment a miner connects to the platform until a solved block becomes part of the eCash blockchain.

Unlike traditional pool documentation, the following workflow reflects the actual production architecture of the LMWPool XEC platform.

---

### 10.2 Overall Workflow

```
                 eCash Network
                       ▲
                       │
               Solved Block Accepted
                       │
                 bitcoind-xec
                 RPC + ZMQ
                       ▲
                       │
                  submitblock()
                       ▲
                       │
                  Candidate Block
                       ▲
                       │
                Share Validation
                       ▲
                       │
                    CKPool
                 Stratum Server
                       ▲
                       │
                Mining Shares
                       ▲
                       │
                    Miner ASIC
```

Every solved block follows this sequence.

---

### 10.3 Miner Connection

A miner connects using one of the production endpoints.

Standard Variable Difficulty endpoint:

```
stratum+tcp://xec.lmwpool.com:7333
```

High Difficulty endpoint:

```
stratum+tcp://xec.lmwpool.com:7444
```

The worker name is composed of:

```
ecash_address.worker
```

Example:

```
ecash:qprn7hh6j3spzqr90lxt2zaf4fecqr2al57h0weclj.antminer01
```

The eCash address identifies the payout destination.

The worker suffix is used only for identification and statistics.

---

### 10.4 Worker Authentication

When the miner connects:

1. The TCP connection is established.
2. The Stratum handshake completes.
3. The miner sends an authorization request.
4. CKPool registers the worker.
5. CKPool creates the mining session.

No username/password database exists.

Authentication is intentionally lightweight because ownership is determined by the payout address itself.

---

### 10.5 Job Generation

Whenever a new block template is required:

CKPool calls:

```
getblocktemplate
```

through the local RPC interface.

`bitcoind-xec` builds the candidate block template using:

- latest blockchain tip;
- current mempool;
- consensus rules.

The resulting template is sent back to CKPool.

---

### 10.6 Distribution of Mining Jobs

CKPool distributes the generated template to every connected miner.

Each miner receives:

- block header;
- merkle root;
- extranonce;
- target difficulty;
- timestamp;
- job identifier.

Miners then begin hashing locally.

---

### 10.7 Share Generation

Each ASIC continuously computes hashes.

Whenever a hash satisfies the assigned share target:

```
hash < share_target
```

the share is submitted to CKPool.

Most submitted shares are **not** blockchain-valid blocks.

They are only proofs of work satisfying the current pool difficulty.

---

### 10.8 Share Validation

Upon receipt, CKPool verifies:

- that the job exists;
- that the worker is active;
- whether the share is a duplicate;
- timestamp validity;
- the extranonce;
- share difficulty; and
- stale status.

Accepted shares are recorded.

Rejected shares are logged.

---

### 10.9 Variable Difficulty

The platform exposes two different mining profiles.

Port `7333`

Optimized for:

- hobby miners;
- Bitaxe devices; and
- lower-hashrate ASICs.

Port `7444`

Optimized for:

- Antminer devices;
- WhatsMiner devices; and
- enterprise hardware.

Using separate endpoints significantly reduces unnecessary share traffic from miners using the High Difficulty endpoint.

---

### 10.10 Candidate Block Detection

Occasionally a submitted share satisfies not only the pool difficulty but also the current blockchain difficulty.

When this happens:

```
Share Difficulty

>=

Network Difficulty
```

the share becomes a candidate block.

CKPool immediately prepares the full block for submission.

---

### 10.11 Block Submission

Submission occurs through:

```
submitblock()
```

using the local RPC interface.

No external services participate.

The local node immediately validates the:

- block header;
- Merkle tree;
- transactions;
- proof of work; and
- consensus rules.

If the block is valid, the node broadcasts it to the eCash network.

---

### 10.12 Network Acceptance

After propagation, other network peers independently validate the block. If the network accepts it, the block becomes part of the canonical blockchain.

At this point the block is considered solved.

---

### 10.13 Coinbase Reward

Unlike conventional pools, the block reward is **not** first collected by the operator and redistributed later.

The production configuration embeds the winning miner's payout address directly into the coinbase transaction.

Example observed in production:

```
Height: 949533

Reward destination:

ecash:qprn7hh6j3spzqr90lxt2zaf4fecqr2al57h0weclj

Worker:

youngkk78
```

This demonstrates that the reward belongs directly to the successful miner.

The pool operator does not perform any later payout process.

---

### 10.14 Pool Donation

The pool retains its configured donation.

Production value:

```
1%
```

This deduction is handled internally by CKPool during coinbase construction.

No manual accounting process exists.

---

### 10.15 Logging

Every important mining event is recorded inside

```
/root/xec_ckpool/logs/ckpool.log
```

Events include:

- worker connections;
- worker disconnections;
- accepted shares;
- rejected shares;
- difficulty updates;
- solved blocks;
- RPC communication;
- startup;
- shutdown.

This log represents the primary operational audit trail.

---

### 10.16 Block Verification

Although CKPool reports solved blocks, the platform performs an independent verification.

Utility:

```
parse_block.py
```

Workflow:

```
CKPool Log

↓

Solved Block Entry

↓

getblockhash()

↓

getblock()

↓

Coinbase Verification

↓

JSON Export

↓

Status Page
```

Only independently verified blocks are published publicly.

---

### 10.17 Telegram Notification

Block detection does not rely on CKPool logs.

Instead:

```
block_notify_xec.sh
```

monitors wallet activity through RPC.

Workflow:

```
Wallet

↓

listtransactions()

↓

Generated Coinbase

↓

Verification

↓

Telegram Message
```

This architecture makes notifications independent of CKPool process restarts.

---

### 10.18 Public Status Page

After verification:

```
parse_block.py
```

updates

```
mined_blocks.json
```

which is consumed by the Status Page.

Visitors therefore see only verified blocks.

---

### 10.19 Complete End-to-End Sequence

```
ASIC

↓

CKPool

↓

Share Validation

↓

Candidate Block

↓

submitblock()

↓

bitcoind-xec

↓

eCash Network

↓

Blockchain Consensus

↓

Wallet Detection

↓

Telegram Notification

↓

parse_block.py

↓

mined_blocks.json

↓

Status Page
```

This represents the complete operational workflow currently implemented in the LMWPool XEC production environment.

---

### 10.20 Operational Advantages

The workflow has several important characteristics:

- no external blockchain APIs;
- direct interaction with the local full node;
- independent block verification;
- independent notification subsystem;
- direct reward assignment to the winning miner;
- simplified architecture;
- deterministic behavior;
- complete auditability through logs and RPC.

These design choices reduce operational complexity while increasing transparency and long-term maintainability.
## 11. Caddy Reverse Proxy

### 11.1 Purpose

The LMWPool platform uses **Caddy v2** as its single public reverse proxy.

Although the container is named:

```
caddy-btc
```

it serves **both** the Bitcoin and the XEC infrastructures.

This architectural choice avoids maintaining two HTTPS gateways while providing centralized TLS management, routing and request filtering.

Caddy is therefore considered a **shared infrastructure component**, not a Bitcoin-specific service.

Stratum mining traffic never traverses Caddy.

Only HTTPS traffic is reverse proxied through Caddy.

---

### 11.2 Production Deployment

Container:

```
caddy-btc
```

Image:

```
caddy:2
```

Docker Compose Project:

```
btc_miningcore
```

Configuration file:

```
/root/docker-bitcoin-pool/caddy/Caddyfile
```

Working directory:

```
/srv
```

---

### 11.3 Docker Networks

The Caddy container participates in three Docker networks.

```
bridge
```

Default Docker network.

```
btc_miningcore_btc_miningcore_internal
```

Private Bitcoin infrastructure.

```
poolnet
```

Shared production network.

The `poolnet` network is the critical element allowing Caddy to communicate with services belonging to different Docker Compose projects.

Without `poolnet`, the XEC services would not be reachable through Caddy.

---

### 11.4 Mounted Volumes

Current production mounts include:

Configuration:

```
/root/docker-bitcoin-pool/caddy/Caddyfile

↓

/etc/caddy/Caddyfile
```

Market cache

```
/root/docker-bitcoin-pool/market-cache

↓

/srv/market-cache
```

Persistent Caddy data

```
/data
```

Persistent Caddy configuration

```
/config
```

ACME-managed TLS certificates are stored inside Docker volumes rather than inside the project directory.

---

### 11.5 Virtual Hosts

The production configuration exposes three service sites and one redirect-only host.

```
www.lmwpool.com
```

↓

Permanent redirect

↓

```
https://lmwpool.com
```

---

```
lmwpool.com
```

Bitcoin platform.

---

```
btc.lmwpool.com
```

Bitcoin platform.

---

```
xec.lmwpool.com
```

Production XEC platform.

This host serves the:

- Status Page;
- Statistics API;
- Market API;
- Blockchain Notary; and
- static assets.

---

### 11.6 HTTPS

Caddy automatically manages:

- TLS certificates;
- certificate renewal;
- HTTPS redirection; and
- secure defaults.

No manual certificate management is required.

This removes the need for manual certificate lifecycle management.

---

### 11.7 Request Routing

Incoming HTTPS requests are evaluated sequentially.

The first matching rule is executed.

Example:

```
Client

↓

HTTPS

↓

Caddy

↓

Matching Handle

↓

Destination Container
```

This deterministic routing avoids ambiguous request processing.

---

### 11.8 XEC Routing

The production `xec.lmwpool.com` virtual host routes requests as follows:

| Public path | Destination | Purpose |
|---|---|---|
| `/api/market` | Static file `/srv/market-cache/market.json` | Cached market data |
| `/api/pools/xec/miners` | `stats-api-xec:8080` | Miner listing |
| `/api/pools/xec` | `stats-api-xec:8080` | Pool data |
| `/static/*` | `status-page-xec:80` | Static assets |
| `/stats/*` | `stats-api-xec:8080` | Statistics endpoints |
| `/stats/workers-summary*` | `stats-api-xec:8080` | Worker summary |
| `/stats/miners-workers*` | `stats-api-xec:8080` | Miner/worker data |
| `/stats/miner-workers-live*` | `stats-api-xec:8080` | Live worker data |
| `/api/fidelity*` | `stats-api-xec:8080` | Fidelity endpoint |
| `/admin-api/*` | Host service on `127.0.0.1:9000` after rewrite | Administrative API |
| `/notary/*` | Host service on `172.18.0.1:8088` after prefix removal | Blockchain Notary |
| All remaining paths | `status-page-xec:80` | Public XEC status page |

Route order is operationally significant. More specific handlers, especially `/api/market`, must remain before broader or fallback handlers.

---

### 11.9 Market API

A dedicated route serves cached market data.

```
/api/market
```

↓

```
market.json
```

↓

```
/srv/market-cache
```

No dynamic backend is required.

This reduces external API traffic and improves response time.

---

### 11.10 Blockchain Notary

The Blockchain Notary is intentionally isolated from Docker.

Incoming requests

```
/notary/*
```

are rewritten by Caddy.

The prefix

```
/notary
```

is removed before forwarding.

Destination:

```
172.18.0.1:8088
```

This address corresponds to the Node.js server running directly on the host.

Therefore:

```
Internet

↓

HTTPS

↓

Caddy

↓

Node.js

↓

server.js

↓

SQLite
```

The Notary service does not require its own HTTPS implementation.

---

### 11.11 Security Rules

Administrative endpoints are blocked.

Examples:

```
/api/admin*
```

↓

403 Forbidden

---

Metrics endpoints

```
/metrics
```

↓

403 Forbidden

Administrative services are therefore never exposed publicly.

---

### 11.12 Why Only One Caddy?

Maintaining a single reverse proxy provides several operational advantages:

- one TLS configuration;
- one certificate lifecycle;
- one public HTTPS ingress layer;
- shared routing logic;
- simpler disaster recovery; and
- reduced maintenance.

Although the container belongs to the Bitcoin Docker Compose project, it acts as the public gateway for the entire infrastructure.

---

### 11.13 Failure Scenarios

Typical failure scenarios include:

| Failure | Result |
|---|---|
| Caddy container stopped | HTTPS services are unavailable. |
| `poolnet` disconnected | XEC containers are unreachable from Caddy. |
| Invalid `Caddyfile` | Configuration reload fails. |
| Certificate failure | HTTPS is unavailable. |
| Backend unavailable | Caddy returns `502 Bad Gateway`. |

Each scenario can be diagnosed independently using Docker logs and Caddy logs.

---

### 11.14 Operational Checks

Container status

```bash
docker ps
```

Container inspection

```bash
docker inspect caddy-btc
```

Configuration validation

```bash
docker exec caddy-btc caddy validate \
    --config /etc/caddy/Caddyfile
```

Configuration reload

```bash
docker exec caddy-btc caddy reload \
    --config /etc/caddy/Caddyfile
```

View logs

```bash
docker logs caddy-btc
```

---

### 11.15 Architectural Notes

The XEC infrastructure intentionally does **not** have a dedicated reverse-proxy container. The shared Caddy instance acts as the common ingress layer for the BTC and XEC platforms while preserving logical separation between their backend services.

As a result:

- HTTPS and certificate management are centralized;
- routing remains deterministic;
- Docker networking remains simple; and
- operational maintenance is reduced.

This shared reverse proxy architecture has proven reliable in production and represents the recommended deployment model for the LMWPool platform.
## 12. Security Architecture

### 12.1 Security Philosophy

The LMWPool XEC platform has not been designed as a generic hosting environment but as a **dedicated production infrastructure**.

Security is based on five fundamental principles:

- minimize exposed services;
- isolate components;
- avoid unnecessary software;
- keep every blockchain operation local;
- reduce the attack surface.

Rather than adding multiple security layers, the platform reduces complexity so that every running service has a clearly defined purpose.

---

### 12.2 Security Domains

The platform can be divided into four logical security domains.

```
                 Internet
                     │
                     ▼
              Caddy Reverse Proxy
                     │
     ┌───────────────┴───────────────┐
     │                               │
     ▼                               ▼
 XEC Mining Platform          XEC Blockchain Notary
     │                               │
     ▼                               ▼
 bitcoind-xec                  SQLite Database
```

Public exposure is limited to Caddy on HTTP/HTTPS and CKPool on the two published Stratum ports.

All remaining XEC application and blockchain services remain internal.

---

### 12.3 Public Exposure

Only the following services are intentionally reachable from outside.

HTTPS

```
443
```

managed by

```
caddy-btc
```

Mining

```
7333
```

Standard Variable Difficulty endpoint

Mining

```
7444
```

High Difficulty endpoint

No other XEC services are intentionally published.

---

### 12.4 Internal Services

The following services remain internal.

```
bitcoind-xec
```

RPC

```
8332
```

ZMQ

```
28332
```

Statistics API

```
8080
```

Status Page

```
80
```

Node.js Notary

```
8088
```

These services are never accessed directly by Internet clients.

---

### 12.5 Docker Isolation

Every production component runs inside its own execution context.

Examples:

```
bitcoind-xec
```

```
ckpool-xec
```

```
stats-api-xec
```

```
status-page-xec
```

```
caddy-btc
```

Each container performs one specific role.

Compromise of one service does not automatically imply compromise of the others.

---

### 12.6 Network Isolation

The platform uses Docker networking rather than exposing services directly.

Important networks include:

```
poolnet
```

Shared reverse-proxy communication.

```
xec_miningcore_xec_internal
```

Private XEC communication.

Internal services communicate through Docker DNS.

No hard-coded public IP addresses are required.

---

### 12.7 RPC Security

The JSON-RPC interface is one of the most sensitive components.

Characteristics:

- private;
- authenticated; and
- internal only.

RPC credentials are stored in configuration files.

They are never exposed through the public web interface.

Only trusted services—the following—are allowed to communicate with the node:

- CKPool;
- the block parser;
- notification utilities; and
- the Notary.

---

### 12.8 ZMQ Security

ZeroMQ is used exclusively for blockchain notifications.

Characteristics:

- internal;
- unauthenticated; and
- reachable only through the Docker network.

Since it never leaves the Docker environment, exposure is intentionally minimized.

---

### 12.9 Reverse Proxy Protection

Caddy acts as the first security boundary.

Requests are filtered before reaching backend services.

Examples:

Administrative endpoints

```
/api/admin*
```

↓

403 Forbidden

Metrics

```
/metrics
```

↓

403 Forbidden

Only explicitly configured routes are forwarded.

---

### 12.10 Blockchain Node Security

The eCash full node is considered the most valuable production component.

It contains:

- blockchain state;
- wallet;
- mining templates;
- RPC services.

For this reason:

- direct Internet exposure is avoided;
- communication occurs only through trusted Docker networks;
- all blockchain operations remain local.

---

### 12.11 CKPool Security

CKPool accepts only Stratum traffic.

Responsibilities include:

- worker authorization;
- share validation;
- block submission.

It does not expose:

- REST APIs;
- databases;
- management interfaces.

Its operational surface therefore remains intentionally small.

---

### 12.12 Notary Security

The Blockchain Notary is logically separated from the mining platform.

Characteristics include:

- dedicated Node.js process;
- SQLite database;
- HTTPS through Caddy;
- independent REST API.

Although both systems share the same VPS, they remain operationally independent.

Failure of the Notary does not interrupt mining.

Failure of mining does not prevent the Notary API from functioning (provided the blockchain node remains available).

---

### 12.13 Authentication

Different components use different authentication mechanisms.

RPC

```
Username + Password
```

Telegram

```
Bot Token
```

Binance utilities

```
API Key
API Secret
```

Notary

```
API Key / User Key
```

Each credential has a single purpose.

Credential reuse is intentionally avoided.

---

### 12.14 Logging

Security-relevant events are recorded by several components.

CKPool

```
ckpool.log
```

Caddy

Docker logs

Notary

systemd logs

Node.js logs

Block notifications

```
block_notify_xec.log
```

Logs are considered part of the platform security model because they provide the audit trail required for incident analysis.

---

### 12.15 Operational Recommendations

Recommended operational practices include:

- keep Docker images updated;
- back up configuration files;
- back up SQLite databases;
- back up wallet data;
- periodically verify TLS certificates;
- monitor available disk space;
- verify blockchain synchronization;
- monitor container health.

---

### 12.16 Disaster Recovery Considerations

A complete platform recovery requires:

- Docker Compose files;
- CKPool configuration;
- Caddy configuration;
- utility scripts;
- Notary source code;
- SQLite database;
- blockchain wallet;
- production configuration files.

For this reason, documentation is considered an essential security component of the platform.

---

### 12.17 Security Summary

The security model of the LMWPool XEC platform is based on **controlled simplicity** rather than excessive layering.

The architecture minimizes exposed services, keeps blockchain interactions local, isolates responsibilities between components and centralizes Internet access through a single reverse proxy.

This approach has proven reliable in production while remaining easy to understand, maintain and recover in case of failure.
## 13. Production Configuration Files

### 13.1 Purpose

This chapter documents the production configuration files required to deploy, maintain and recover the LMWPool XEC platform.

Only files that actively participate in the production environment are described.

Historical or deprecated configurations are intentionally excluded.

---

### 13.2 Directory Layout

The XEC platform is organized around three primary production directories.

```
/root
│
├── xec_ckpool
│
├── xec_miningcore
│
└── xec_notary
```

These three directories contain the primary XEC mining, support-service and Notary application assets. Shared ingress and market-cache assets also reside under `/root/docker-bitcoin-pool`.

---

### 13.3 `/root/xec_ckpool`

This directory contains the mining engine.

Typical structure:

```
xec_ckpool
│
├── config/
│
├── logs/
│
├── ecash-ckpool-solo/
│
├── docker-compose-ckpool-xec.yml
│
└── utilities
```

Responsibilities:

- CKPool source
- Docker deployment
- configuration
- mining logs

---

### 13.4 `docker-compose-ckpool-xec.yml`

Purpose:

Deploys the CKPool production container.

Container:

```
ckpool-xec
```

Important characteristics:

- local image build

- restart policy

```
unless-stopped
```

- read-only configuration

```
`./config/ckpool-xec.conf`

↓

`/config/ckpool-xec.conf`
```

- persistent logs

```
./logs

↓

/logs
```

Published ports

```
7333 → 3333
7444 → 4444
```

Docker Networks:

```
xec_miningcore_xec_internal

poolnet
```

These external networks are mandatory for production operation.

---

### 13.5 `ckpool-xec.conf`

Location:

```
/root/xec_ckpool/config/ckpool-xec.conf
```

Purpose:

Main CKPool configuration.

Important parameters:

Blockchain node

```
bitcoind-xec:8332
```

RPC username

```
poolxec
```

RPC password

Configured in production.

Wallet address

```
btcaddress
```

Pool signature

```
btcsig
```

Donation

```
1%
```

Public endpoints

```
3333
4444
```

ZMQ

```
tcp://bitcoind-xec:28332
```

This file is considered one of the most critical configuration files in the platform.

---

### 13.6 CKPool Logs

Directory:

```
/root/xec_ckpool/logs
```

Primary file:

```
ckpool.log
```

The log provides the operational audit trail. It records:

- startup and shutdown events;
- miners and shares;
- RPC activity;
- solved blocks; and
- errors.

Several utilities consume this log.

---

### 13.7 `/root/xec_miningcore`

Although the name contains "miningcore", this directory no longer hosts Miningcore for XEC.

Instead, it contains production utilities.

Typical contents:

```
utilities/

status-page/

scripts/

market/

maintenance
```

Responsibilities include:

- statistics;
- parsers;
- administration;
- the market cache;
- Binance utilities; and
- production scripts.

---

### 13.8 `parse_block.py`

Purpose: independent verification of solved blocks.

Input:

```
ckpool.log
```

Verification: RPC calls to `getblockhash`, followed by `getblock`.

Output:

```
mined_blocks.json
```

Ignored blocks:

```
mined_blocks_ignored.json
```

Only independently verified blocks appear on the public website.

---

### 13.9 `block_notify_xec.sh`

Purpose:

Wallet monitoring.

Unlike `parse_block.py`, it does not read CKPool logs.

Instead, it periodically calls:

```
listtransactions()
```

through RPC.

Advantages include:

- independence from CKPool;
- continued operation across CKPool restarts;
- detection of generated rewards; and
- Telegram integration.

---

### 13.10 Binance Utilities

Utilities include

```
binance_convert_xec_btc_real.py
```

Purpose:

Automatic conversion

```
XEC

↓

BTC
```

Configuration:

```
binance.env
```

Credentials:

```
BINANCE_API_KEY

BINANCE_API_SECRET
```

The utility communicates directly with the Binance REST API.

---

### 13.11 Status Page

Directory:

```
status-page/
```

The Status Page is the public presentation layer. It consumes:

```
mined_blocks.json
```

- market data;
- worker statistics; and
- pool information.

The Status Page never communicates directly with the blockchain.

---

### 13.12 Market Cache

Purpose: reduce external API calls.

Caddy serves

```
market.json
```

directly from

```
/srv/market-cache
```

This design reduces latency and avoids unnecessary dependencies on third-party APIs.

---

### 13.13 Shared Reverse Proxy

Configuration:

```
/root/docker-bitcoin-pool/caddy/Caddyfile
```

Although stored inside the Bitcoin project, this file controls routing for both the BTC and XEC platforms.

This makes the Caddy configuration part of the XEC production configuration set.

---

### 13.14 `/root/xec_notary`

The Blockchain Notary is maintained separately.

Typical structure:

```
xec_notary
│
├── server.js
├── data/
├── templates/
├── certificates/
├── logs/
└── package.json
```

Although documented in Volume 3, this directory is included here because it forms part of the overall production platform.

---

### 13.15 Configuration Priority

The production platform follows this configuration hierarchy:

```
Docker Compose

↓

Application Configuration

↓

Runtime Environment

↓

Logs

↓

Generated Data
```

Changes should always begin with configuration files.

Generated data should never be edited manually.

---

### 13.16 Files Requiring Backup

The following files are considered mandatory for disaster recovery.

#### Critical

```
docker-compose-ckpool-xec.yml

ckpool-xec.conf

Caddyfile

server.js

SQLite database

wallet data

binance.env
```

---

#### Important

```
parse_block.py

block_notify_xec.sh

status-page

maintenance scripts
```

---

#### Generated

```
logs

market cache

JSON exports
```

Generated files can usually be recreated after recovery.

---

### 13.17 Configuration Management

Production configuration follows three principles.

#### Explicit

Every production parameter is visible.

No hidden configuration.

---

#### Version Controlled

Configuration files should be maintained under version control whenever possible.

---

#### Immutable Runtime

Configuration mounted into Docker containers is preferably read-only.

Example:

```
ckpool-xec.conf

↓

read-only mount
```

This prevents accidental modification from inside containers.

---

### 13.18 Summary

The production platform is intentionally driven by a small number of clearly defined configuration files.

Together with the procedures and inventories in the reference appendices, these files provide the basis for reconstructing the infrastructure on a new server.

This greatly simplifies maintenance, disaster recovery and long-term platform evolution.
## 14. Operations Runbook

### 14.1 Purpose

This chapter describes the operational procedures required to safely manage the LMWPool XEC production platform.

It is intended for system administrators responsible for:

- daily operation;
- maintenance;
- troubleshooting;
- disaster recovery;
- software upgrades.

The procedures documented below reflect the current production deployment.

---

### 14.2 Production Components

Before maintenance, record the state of the following components and verify each one that is expected to be operational:

| Component | Status |
|-----------|--------|
| `bitcoind-xec` | Running |
| `ckpool-xec` | Running |
| `stats-api-xec` | Running |
| `status-page-xec` | Running |
| `caddy-btc` | Running |
| `xec_notary` (`systemd`) | Running |

---

### 14.3 Daily Operational Check

The following checks should be performed daily.

#### Docker

```bash
docker ps
```

Verify that the following containers are running:

- `ckpool-xec`;
- `bitcoind-xec`;
- `stats-api-xec`;
- `status-page-xec`; and
- `caddy-btc`.

---

#### Blockchain

Verify synchronization.

Typical RPC checks:

```
getblockchaininfo

getblockcount

getbestblockhash
```

The node must not be in Initial Block Download.

---

#### CKPool

Verify:

```
docker logs ckpool-xec
```

or

```
tail -f /root/xec_ckpool/logs/ckpool.log
```

Look for accepted shares, connected miners, RPC errors and solved blocks.

---

#### Wallet

Verify:

```
getwalletinfo
```

Wallet must be reachable.

---

#### Status Page

Open:

```
https://xec.lmwpool.com
```

Verify:

- statistics

- solved blocks

- miners

- market information

---

#### Notary

Verify:

```
https://xec.lmwpool.com/notary/
```

Confirm:

- API reachable

- certificate generation

- SQLite access

---

### 14.4 Startup Procedure

Recommended order:

1. Start `bitcoind-xec` and wait until it is synchronized.
2. Start CKPool.
3. Start the Statistics API.
4. Start the Status Page.
5. Start Caddy.
6. Start the Notary.

Never start CKPool before the blockchain node becomes fully operational.

---

### 14.5 Shutdown Procedure

Use the reverse order:

1. Stop miners.
2. Stop CKPool.
3. Stop the Statistics API.
4. Stop the Status Page.
5. Stop the Notary.
6. Stop `bitcoind-xec`.

Waiting for the blockchain node to flush database changes before powering off the server is strongly recommended.

---

### 14.6 Routine Maintenance

Routine maintenance includes:

- Docker image updates;
- Linux security patches;
- disk-space verification;
- wallet backups;
- SQLite backups;
- log rotation;
- blockchain synchronization verification; and
- TLS certificate verification.

---

### 14.7 Backup Strategy

The following items are mandatory.

#### Configuration

```
docker-compose

CKPool configuration

Caddyfile

systemd units

server.js
```

---

#### Data

Wallet.

SQLite database.

Market cache (optional).

---

#### Scripts

- `parse_block.py`
- `block_notify_xec.sh`
- maintenance utilities
- Binance utilities

---

#### Logs

Although logs can become large, periodic archival is recommended.

Important files include

```
ckpool.log

block_notify_xec.log

systemd logs
```

---

### 14.8 Disaster Recovery

Recovery sequence:

1. Prepare Linux.
2. Install Docker.
3. Restore the Docker networks.
4. Restore configuration.
5. Restore the wallet.
6. Restore SQLite data.
7. Start `bitcoind-xec`.
8. Wait for blockchain synchronization.
9. Start CKPool.
10. Start the remaining services.
11. Validate production operation.

The blockchain node should always be restored before every other blockchain-dependent component.

---

### 14.9 Troubleshooting

#### No miners connected

Verify ports `7333` and `7444`. Check that CKPool is running and that the firewall, DNS and Internet connectivity are functioning.

---

#### RPC Errors

Verify `bitcoind-xec`, RPC credentials, Docker networking and node synchronization.

---

#### Solved block missing

Verify each stage in order:

1. CKPool log;
2. `parse_block.py`;
3. RPC verification;
4. Status Page JSON; and
5. the public website.

---

#### Telegram notifications missing

Verify `block_notify_xec.sh`, wallet RPC, the Telegram bot token, the chat ID and Internet connectivity.

---

#### Status Page not updating

Verify the Statistics API, JSON generation, NGINX and Caddy, in that order.

---

#### HTTPS unavailable

Verify the Caddy container, certificates, DNS and reachability of port `443`.

---

#### Notary unavailable

Verify `systemd`, `server.js`, SQLite, port `8088` and Caddy routing, in that order.

---

### 14.10 Monitoring Recommendations

The following indicators should be monitored continuously:

- blockchain synchronization;
- connected miners;
- accepted and rejected shares;
- container health;
- disk utilization;
- wallet availability;
- HTTPS availability;
- Telegram notifications; and
- Notary availability.

---

### 14.11 Operational Principles

The platform follows several operational rules:

- modify configuration before restarting services;
- never edit configuration from inside running containers;
- verify blockchain synchronization before troubleshooting CKPool;
- verify RPC before attributing an incident to mining; and
- verify Docker networking before changing application code.

---

### 14.12 Documentation Maintenance

Every production modification should be reflected in this manual.

Examples include a new Docker service, API, port, reverse-proxy rule, maintenance script or backup procedure.

Keeping documentation synchronized with production is considered part of normal platform maintenance.

---

### 14.13 Final Operational Notes

The LMWPool XEC platform intentionally favors **operational simplicity** over architectural complexity.

The entire mining infrastructure is built around a small number of clearly defined services:

- `bitcoind-xec`;
- CKPool;
- Statistics API;
- Status Page;
- Caddy; and
- Blockchain Notary.

Each component has a single responsibility, communicates through well-defined interfaces and can be diagnosed independently.

This design significantly simplifies:

- maintenance;
- troubleshooting;
- disaster recovery;
- future platform evolution.

The architecture has been validated through continuous production use and constitutes the reference implementation documented by this manual.


---

## Release 1.0 Editorial Notes

Release 1.0 consolidates the Volume 1 RC2 source into the definitive technical reference. The editorial review:

- removed drafting artifacts and duplicated or weak passages;
- standardized headings, terminology, lists, code formatting and architecture diagrams;
- corrected the production Caddy network name and documented the actual XEC route order;
- corrected the public-exposure description to include the two CKPool Stratum endpoints;
- added document control, a table of contents, criticality information and production-value tables;
- replaced unsupported interpretations of local CKPool command-line flags with a verification requirement;
- retained the documented production architecture and factual content without introducing new runtime components, commands or configuration values.

### Unresolved Factual Verification Items

The RC2 source does not contain enough verified evidence to resolve the following details. They remain assigned to the reference appendices and must be validated against production before being documented more precisely:

- exact eCash Core container build and compose definition;
- complete `xec.conf` and wallet-backup procedure;
- complete port, network, volume and service inventory;
- line-by-line configuration references;
- exact service manager or scheduler definitions for native scripts;
- production secrets inventory, with values redacted;
- command-validated startup, shutdown, backup and disaster-recovery runbooks.
