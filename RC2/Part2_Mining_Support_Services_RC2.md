# RC2 Editorial Revision Notice

**Release:** RC2 (Editorial Review Candidate)

This RC2 preserves the production architecture and technical content of RC1 while introducing editorial improvements intended to prepare the document for Release 1.0.

## Editorial objectives

- Improved consistency with the other volumes.
- Standardized terminology.
- Preparation for numbered figures and tables.
- Improved cross-reference strategy to the Appendices.
- Preparation for a unified documentation style across the complete documentation suite.

---


# PART 2
# Mining Support Services

Version 2.0

---

# 1. Purpose

Unlike the mining engine itself, the components described in this part do **not participate directly in Proof-of-Work**.

Instead they provide every operational service required to transform CKPool into a production platform.

These services include:

- statistics
- public web interface
- solved block verification
- Telegram notifications
- maintenance utilities
- market data
- Binance integration
- administrative scripts

Without these services the mining pool would still solve blocks, but it would no longer provide operational visibility, monitoring or public information.

---

# 2. Service Architecture Overview

The support layer is intentionally independent from the mining engine.

```
                    +----------------------+
                    |     CKPool           |
                    +----------+-----------+
                               |
                               |
                 +-------------+--------------+
                 |                            |
                 |                            |
                 ▼                            ▼

         parse_block.py             block_notify_xec.sh

                 │                            │

                 ▼                            ▼

        mined_blocks.json          Telegram Bot API

                 │

                 ▼

          stats-api-xec

                 │

                 ▼

         status-page-xec

                 │

                 ▼

             Caddy HTTPS

                 │

                 ▼

               Internet
```

Every component has exactly one responsibility.

No component performs more than one operational task.

This design dramatically simplifies debugging.

---

# 3. Design Principles

The service layer follows the same architectural principles used throughout the platform.

## Separation of Concerns

Mining.

Statistics.

Notifications.

Presentation.

Administration.

These are independent responsibilities.

Each service may fail without necessarily affecting the others.

---

## Log Driven

Most support services do **not** communicate directly with CKPool.

Instead they consume:

- logs

or

- RPC

This prevents unnecessary coupling.

---

## Stateless Whenever Possible

The majority of support services generate information dynamically.

Persistent storage is intentionally minimized.

Exceptions include:

- SQLite (Notary)

- Market cache

- generated JSON

---

## Independent Verification

Whenever possible the platform independently verifies information before publishing it.

Example:

CKPool says

↓

Solved Block

↓

parse_block.py

↓

RPC verification

↓

Status Page

The public website never relies solely on CKPool log entries.

---

# 4. Production Support Services

Current production support services include:

| Service | Purpose |
|----------|----------|
| stats-api-xec | Mining statistics |
| status-page-xec | Public website |
| parse_block.py | Block verification |
| block_notify_xec.sh | Wallet monitoring |
| Binance utilities | XEC/BTC conversion |
| market cache | Cached market data |
| maintenance scripts | Administration |

---

# 5. Directory Layout

Most support software resides inside

```
/root/xec_miningcore
```

Typical structure

```
xec_miningcore

│

├── utilities

│

├── status-page

│

├── market

│

├── scripts

│

└── maintenance
```

Although the directory still contains the historical name

```
xec_miningcore
```

Miningcore is **not responsible** for XEC mining; the directory name remains for historical compatibility.

The directory has evolved into the operational support environment for the entire XEC platform.

---

# 6. Communication Model

The support services intentionally communicate through a limited and well-defined set of interfaces.

RPC

↓

bitcoind

Filesystem

↓

logs

Filesystem

↓

JSON

HTTP

↓

Stats API

HTTPS

↓

Caddy

Telegram HTTPS

↓

Bot API

No proprietary protocols are introduced.

This minimizes coupling and simplifies long-term maintenance and troubleshooting.

---

# 7. Data Flow

The complete information flow is shown below.

```
Miner

↓

CKPool

↓

Logs

↓

Parser

↓

JSON

↓

Stats API

↓

Status Page

↓

HTTPS

↓

User
```

Notifications instead follow a completely different path.

```
Wallet

↓

RPC

↓

block_notify

↓

Telegram
```

This independence increases platform resilience.

---

# 8. Operational Philosophy

The support layer has three primary objectives.

1.

Observe.

2.

Verify.

3.

Publish.

Mining itself is never modified by these services.

They only observe production activity and transform it into operational information suitable for administrators and public users.

This architectural separation has proven to be one of the major strengths of the LMWPool XEC platform.
# 9. stats-api-xec

## 9.1 Purpose

The **stats-api-xec** service provides the operational data layer for the public XEC mining platform.

Unlike CKPool, whose responsibility ends with mining, **stats-api-xec** transforms production information into JSON endpoints consumable by:

- Status Page
- JavaScript clients
- Administrative tools
- External integrations

It is therefore the presentation layer between the mining infrastructure and the public website.

---

# 9.2 Design Philosophy

The service intentionally contains **no mining logic**.

It never:

- creates mining jobs;
- validates shares;
- communicates with miners;
- submits blocks.

Instead it aggregates information coming from:

- CKPool generated data
- parser outputs
- market cache
- production JSON files
- runtime statistics

The philosophy is:

```
Read

↓

Aggregate

↓

Publish
```

Nothing else. Business logic remains outside the presentation layer.

---

# 9.3 Production Deployment

Container

```
stats-api-xec
```

Base image

```
python:3.12-alpine
```

Status

```
Always running
```

Docker Network

```
poolnet
```

Internal Port

```
8080
```

The service is intentionally not exposed directly to the Internet.

All external access is mediated by the shared Caddy reverse proxy.

---

# 9.4 Position inside the Architecture

```
                  CKPool
                     │
                     │
             Generated Data
                     │
                     ▼

              stats-api-xec

                     │
             HTTP JSON API

                     ▼

             status-page-xec

                     │

                     ▼

                  Caddy

                     ▼

                  Internet
```

The Stats API acts as the single source of operational information for the public interface.

---

# 9.5 Responsibilities

Current responsibilities include:

- pool statistics
- miner statistics
- worker statistics
- live worker information
- fidelity information
- market information
- solved block information

Each dataset is generated independently.

Failure of one dataset does not necessarily prevent the others from being served.

---

# 9.6 Main Endpoints

Current production endpoints include:

Pool

```
/api/pools/xec
```

Miners

```
/api/pools/xec/miners
```

Statistics

```
/stats/*
```

Workers summary

```
/stats/workers-summary
```

Worker details

```
/stats/miners-workers
```

Live workers

```
/stats/miner-workers-live
```

Fidelity

```
/api/fidelity
```

Market

```
/api/market
```

These endpoints are routed through Caddy.

---

# 9.7 JSON Generation

The service primarily exposes JSON documents.

Typical workflow

```
Production Data

↓

Python Objects

↓

JSON Serialization

↓

HTTP Response
```

The API intentionally avoids server-side HTML generation.

Presentation is delegated to the Status Page.

---

# 9.8 Relationship with parse_block.py

One of the most important integrations concerns solved blocks.

Workflow

```
CKPool Log

↓

parse_block.py

↓

RPC Verification

↓

mined_blocks.json

↓

stats-api-xec

↓

Status Page
```

Notice that the Stats API never parses CKPool logs directly.

It consumes only verified JSON data.

This architectural decision prevents accidental publication of unverified block information.

---

# 9.9 Relationship with Status Page

The Status Page acts exclusively as the presentation layer.

```
Browser

↓

JavaScript

↓

stats-api-xec

↓

JSON

↓

Rendering
```

Business logic remains inside the API.

Presentation logic remains inside the browser.

The two responsibilities are intentionally separated.

---

# 9.10 Relationship with Caddy

The service is never exposed directly.

Incoming requests follow this path:

```
HTTPS

↓

Caddy

↓

stats-api-xec

↓

JSON

↓

Browser
```

Advantages:

- centralized TLS
- centralized routing
- simplified firewall
- reduced attack surface

---

# 9.11 Failure Scenarios

## Container stopped

Effects

- statistics unavailable
- status page partially degraded

Mining operations continue normally.

---

## Invalid JSON

Effects

- rendering failures
- missing statistics

Mining operations continue normally.

---

## Parser failure

Effects

Solved block history stops updating.

Mining continues.

---

## Market API unavailable

Effects

Market prices unavailable.

Mining unaffected.

---

# 9.12 Operational Checks

Verify container

```bash
docker ps
```

View logs

```bash
docker logs stats-api-xec
```

Verify endpoint

```bash
curl http://stats-api-xec:8080/...
```

Verify through Caddy

```bash
curl https://xec.lmwpool.com/stats/...
```

Verify JSON

Ensure responses are:

- valid JSON
- current
- internally consistent

---

# 9.13 Performance Considerations

The service has very modest resource requirements.

Characteristics:

- CPU usage typically low.
- Memory footprint small.
- No blockchain synchronization.
- No database engine.
- Stateless operation.

Consequently, performance bottlenecks are uncommon.

The largest latency contributors are generally external data sources or filesystem access.

---

# 9.14 Security Considerations

The Stats API should never expose:

- RPC credentials
- wallet information
- internal Docker topology
- administrative endpoints
- private configuration

Only operational information intended for public consumption should be returned.

Administrative functionality must remain outside this service.

---

# 9.15 Summary

The **stats-api-xec** service is the operational information hub of the XEC platform.

It deliberately separates data collection from presentation, consumes only verified information, publishes lightweight JSON endpoints and remains completely independent from the mining engine itself.

This separation allows the mining platform to continue operating even if the statistics layer becomes temporarily unavailable, providing a robust and maintainable production architecture.

# 10. status-page-xec

## 10.1 Purpose

The **status-page-xec** service provides the public web interface of the LMWPool XEC platform.

Its primary objective is to present operational mining information in a simple, responsive and continuously updated format without exposing any internal infrastructure details.

Unlike the Stats API, the Status Page contains **no business logic**.

Its responsibilities are limited to:

- rendering user interfaces;
- requesting JSON data;
- presenting statistics;
- displaying solved blocks;
- showing market information;
- providing miner visibility.

The Status Page therefore represents the presentation layer of the platform.

---

# 10.2 Architectural Position

```
                 Browser

                    │

               HTTPS Request

                    │

                    ▼

                 Caddy

                    │

                    ▼

          status-page-xec

                    │

         JavaScript Fetch()

                    │

                    ▼

            stats-api-xec

                    │

                    ▼

            Production Data
```

The browser never communicates directly with CKPool, the blockchain node or internal RPC services.

Every request is mediated through the API layer.

---

# 10.3 Production Deployment

Container

```
status-page-xec
```

Base Image

```
nginx:alpine
```

Internal Port

```
80
```

Docker Network

```
poolnet
```

The container is intentionally lightweight.

No application server is executed inside the container.

NGINX only serves static resources.

---

# 10.4 Static Resources

Typical resources include:

```
HTML

CSS

JavaScript

Images

Icons

Fonts

Static JSON
```

Dynamic information is never generated by NGINX.

Everything dynamic originates from the Stats API.

---

# 10.5 JavaScript Architecture

The browser periodically requests production data.

Typical workflow

```
Page Load

↓

JavaScript Initialization

↓

REST Request

↓

JSON Response

↓

DOM Update

↓

Automatic Refresh
```

This model minimizes server-side complexity.

---

# 10.6 Solved Blocks

Solved blocks displayed on the website originate from:

```
parse_block.py

↓

mined_blocks.json

↓

stats-api-xec

↓

Browser
```

The Status Page never parses CKPool logs directly.

This guarantees that visitors only see independently verified blockchain information.

---

# 10.7 Miner Information

The interface displays information including:

- connected miners;
- workers;
- hashrate;
- solved blocks;
- activity;
- historical statistics.

These values are retrieved from the Stats API and rendered in real time.

---

# 10.8 Market Information

Market prices are obtained through

```
/api/market
```

served by Caddy from the cached market JSON.

This architecture avoids unnecessary external API requests from browsers while providing consistent market information to every visitor.

---

# 10.9 Failure Scenarios

## Status Page unavailable

Effects

Users cannot access the website.

Mining operations continue normally.

---

## Stats API unavailable

Effects

The website loads but statistics cannot be updated.

Mining operations continue normally.

---

## Caddy unavailable

Effects

HTTPS access unavailable.

Mining operations continue normally.

---

## Browser JavaScript failure

Effects

The interface becomes partially unusable.

No production mining component is affected.

---

# 10.10 Operational Checks

Verify container

```bash
docker ps
```

Verify NGINX

```bash
docker logs status-page-xec
```

Verify website

```
https://xec.lmwpool.com
```

Confirm:

- page loading;
- CSS;
- JavaScript;
- API communication;
- solved blocks;
- market data;
- miner information.

---

# 10.11 Relationship with Other Services

The Status Page interacts with only three production services.

```
Caddy

↓

stats-api-xec

↓

Market Cache
```

It never communicates directly with:

- CKPool;
- bitcoind;
- SQLite;
- Telegram;
- Binance.

This strict separation keeps the presentation layer simple and secure.

---

# 10.12 Design Advantages

The architecture provides several benefits.

- Completely static web server.
- Minimal resource consumption.
- No application runtime inside NGINX.
- Clear separation between presentation and business logic.
- Easy maintenance.
- Simple disaster recovery.
- Independent evolution of UI and backend.

The **status-page-xec** service therefore acts exclusively as the visual interface of the LMWPool XEC platform, leaving all operational processing to the underlying support services.
# 11. parse_block.py

## 11.1 Purpose

The **parse_block.py** utility is responsible for independently verifying every block reported by CKPool before it becomes visible on the public website.

Although CKPool already reports solved blocks in its operational log, the platform intentionally **does not trust log entries alone**.

Instead, every candidate block is validated directly against the local eCash blockchain node through RPC.

This guarantees that only blocks effectively accepted by the blockchain are published.

The parser therefore acts as an independent validation layer between the mining engine and the public interface.

---

# 11.2 Design Philosophy

The parser follows one fundamental principle:

> **"Never publish a solved block unless the blockchain confirms it."**

Instead of assuming CKPool is always correct, the parser performs its own verification.

The workflow becomes:

```
CKPool

↓

ckpool.log

↓

parse_block.py

↓

RPC Verification

↓

Verified JSON

↓

Status Page
```

This architecture completely eliminates false positives.

---

# 11.3 Production Location

Production script

```
/root/xec_miningcore/parse_block.py
```

Input

```
/root/xec_ckpool/logs/ckpool.log
```

Output

```
/root/xec_miningcore/status-page/mined_blocks.json
```

Secondary output

```
mined_blocks_ignored.json
```

---

# 11.4 Input Data

The parser consumes exactly one source.

```
ckpool.log
```

The log remains the authoritative operational history of CKPool.

Rather than monitoring the blockchain continuously, the parser analyses the mining log and extracts only solved block events.

This minimizes computational overhead.

---

# 11.5 Solved Block Recognition

The parser searches for log entries matching the production regular expression.

```
Solved and confirmed block
```

Current implementation extracts:

- timestamp
- block height
- miner identifier

Example

```
[2026-05-17 ...]

Solved and confirmed block 949533

by

ecash:.....worker
```

---

# 11.6 Miner Parsing

Worker identifiers may contain

```
address.worker
```

The parser separates:

```
Miner Address

Worker Name
```

Example

```
ecash:q....youngkk78

↓

Address

Worker
```

This information is later reused by the Status Page.

---

# 11.7 Duplicate Detection

The parser maintains an internal

```
seen
```

set.

Each solved block is identified by

```
(height, miner)
```

If an identical record already exists it is ignored.

Advantages:

- deterministic output
- no duplicated blocks
- stable JSON generation

---

# 11.8 Blockchain Verification

The parser never assumes that a CKPool log entry corresponds to a valid blockchain block.

Instead it performs two RPC operations.

Step 1

```
getblockhash(height)
```

↓

Step 2

```
getblock(blockhash)
```

The returned block is then analysed.

---

# 11.9 Validation Rules

A block is considered valid only if all conditions are satisfied.

## Height

Returned height must equal

```
parsed height
```

---

## Coinbase Exists

Transaction

```
tx[0]
```

must exist.

---

## Miner Verification

The miner address extracted from CKPool must be present inside the coinbase transaction.

If the payout address is missing:

```
verification failed
```

This is one of the strongest protections implemented by the platform.

It prevents publication of blocks that belong to another miner.

---

# 11.10 Verification Results

Every analysed candidate produces one of two outcomes.

Verified

↓

```
mined_blocks.json
```

Ignored

↓

```
mined_blocks_ignored.json
```

Nothing is discarded silently.

Every rejected candidate is preserved together with the reason for rejection.

---

# 11.11 Ignored Blocks

Typical rejection reasons include:

```
height mismatch

missing coinbase

miner address not found

RPC error

invalid blockchain data
```

This greatly simplifies operational troubleshooting.

---

# 11.12 JSON Structure

Each verified record contains information including:

Timestamp

Block height

Miner

Miner address

Worker

Block hash

Explorer URL

Verification status

Verification reason

Source

This JSON becomes the authoritative dataset for solved blocks.

---

# 11.13 Explorer Links

The parser automatically generates a public explorer URL.

Current implementation

```
https://explorer.e.cash/block-height/{height}
```

This allows administrators and visitors to independently verify every published block.

---

# 11.14 Atomic Updates

One of the most important implementation details is the use of atomic file replacement.

Instead of writing directly to

```
mined_blocks.json
```

the parser:

creates

↓

temporary file

↓

writes complete JSON

↓

flushes data

↓

renames file

↓

replaces original

Advantages

- no partially written JSON
- browser never reads incomplete files
- safe updates
- filesystem consistency

This is significantly more reliable than writing directly to the production JSON.

---

# 11.15 Error Handling

RPC failures never corrupt existing JSON.

Instead:

- exception captured;
- verification fails;
- candidate moved to ignored dataset;
- previous production JSON remains valid.

This behaviour avoids accidental publication of incomplete information.

---

# 11.16 Relationship with Other Components

```
CKPool

↓

ckpool.log

↓

parse_block.py

↓

bitcoind RPC

↓

Verification

↓

mined_blocks.json

↓

stats-api-xec

↓

status-page-xec
```

The parser communicates directly with only two components:

- CKPool (through its log file)
- bitcoind (through RPC)

Every other service consumes its output.

---

# 11.17 Operational Checks

Verify parser execution.

```bash
python3 parse_block.py
```

Expected result

```
Update successful.

Valid solved blocks : XX

Ignored candidates : YY
```

Verify JSON

```
mined_blocks.json
```

Verify ignored records

```
mined_blocks_ignored.json
```

Verify website.

Confirm newly solved blocks appear correctly.

---

# 11.18 Failure Scenarios

Missing log

↓

Parser stops.

---

RPC unavailable

↓

Verification impossible.

---

Malformed log entry

↓

Candidate ignored.

---

Blockchain inconsistency

↓

Candidate rejected.

---

Filesystem error

↓

JSON update aborted.

---

# 11.19 Design Advantages

The parser introduces an independent validation layer that significantly improves the reliability of the platform.

Advantages include:

- blockchain-backed verification;
- duplicate elimination;
- deterministic JSON generation;
- atomic file replacement;
- complete audit trail;
- explicit rejection reasons;
- separation between mining and publication.

This architecture ensures that the public website reflects only blockchain-confirmed events rather than unverified mining log entries.

---

# 11.20 Architectural Importance

Although relatively small in size, **parse_block.py** is one of the key reliability components of the LMWPool XEC platform.

It decouples CKPool from the presentation layer, independently validates solved blocks against the blockchain and guarantees that all publicly displayed mining results are supported by on-chain evidence.

For this reason, it should be considered part of the platform's core operational infrastructure rather than a simple utility script.
# 12. block_notify_xec.sh

## 12.1 Purpose

The **block_notify_xec.sh** service is responsible for detecting newly generated mining rewards and notifying administrators through Telegram.

Unlike many mining platforms that rely on CKPool log parsing, the LMWPool architecture deliberately monitors the **wallet** instead of the mining engine.

This design provides a much higher level of reliability because notifications are generated only after the blockchain node records the coinbase transaction.

The notification subsystem is therefore independent of CKPool.

---

# 12.2 Design Philosophy

The guiding principle is simple:

> **The blockchain is the authoritative source of truth, not the mining log.**

Instead of monitoring:

```
ckpool.log
```

the service continuously monitors

```
bitcoind wallet
```

using authenticated RPC calls.

Only transactions that actually belong to the wallet are considered.

This eliminates false notifications caused by:

- temporary log entries
- parser failures
- CKPool restarts
- duplicated events

---

# 12.3 Production Location

Production script

```
/root/xec_ckpool/block_notify/block_notify_xec.sh
```

Log file

```
/var/log/block_notify_xec.log
```

Persistent working directory

```
/root/block_notify_xec/
```

Internal state files

```
seen_coinbase_txids.txt

ignored_coinbase_txids.txt
```

These files preserve the notification state across script restarts.

---

# 12.4 General Architecture

```
Wallet

↓

RPC

↓

listtransactions()

↓

Generated Coinbase

↓

Validation

↓

Telegram

↓

Administrator
```

Unlike the solved block parser, this service never reads CKPool logs.

---

# 12.5 Wallet Polling

Every

```
POLL_SEC
```

seconds the script executes

```
listtransactions("*",500)
```

through the local RPC interface.

Returned transactions are analysed sequentially.

Only newly discovered transactions are processed.

Already processed transaction IDs are skipped immediately.

---

# 12.6 RPC Connection

The script communicates with

```
bitcoind-xec
```

through

```
bitcoin-cli
```

using

- wallet name
- RPC credentials
- local Docker networking

Example operations include

```
getwalletinfo

listtransactions

getblock
```

No external blockchain API is required.

---

# 12.7 Initial Synchronization

When the service starts for the first time

```
seen_coinbase_txids.txt
```

may be empty.

Instead of immediately notifying every historical block, the script performs an initialization scan.

Workflow

```
Wallet

↓

Historical Transactions

↓

Populate seen list

↓

Start Monitoring
```

This prevents notification storms after maintenance or server reboot.

---

# 12.8 Transaction Filtering

Each wallet transaction is analysed.

The script verifies

Generated

Category

Block Hash

Confirmations

Block Height

Only transactions satisfying all production rules continue through the notification pipeline.

---

# 12.9 Accepted Categories

Only categories representing mining rewards are accepted.

Examples

```
immature

generate
```

Transactions classified as

```
orphan
```

are ignored.

Standard wallet transactions are also ignored.

This ensures that only mining rewards generate Telegram messages.

---

# 12.10 Validation

Before sending a notification the script verifies:

Valid transaction ID

↓

Valid block hash

↓

Valid block height

↓

Generated transaction

↓

Allowed category

↓

Wallet confirmation

Only then is the reward considered eligible.

---

# 12.11 Duplicate Protection

Every processed transaction ID is written to

```
seen_coinbase_txids.txt
```

Future polling iterations immediately ignore previously processed rewards.

This mechanism guarantees that each mining reward is notified exactly once.

---

# 12.12 Ignored Transactions

Transactions rejected during validation are stored inside

```
ignored_coinbase_txids.txt
```

Typical reasons include

```
not generated

orphan

invalid block

missing height

missing block hash
```

Keeping rejected transactions provides a complete operational audit trail.

---

# 12.13 Telegram Integration

Notifications are delivered through the Telegram Bot API.

Typical information includes

Wallet

Category

Reward amount

Confirmations

Block height

Block hash

Transaction ID

Detection timestamp

This provides administrators with immediate visibility of newly detected mining rewards.

---

# 12.14 Service Independence

One of the strongest architectural characteristics of the notification subsystem is its independence.

The service does **not** require

- CKPool
- parser
- Stats API
- Status Page

It only depends on

```
bitcoind wallet
```

Consequently

CKPool may restart

↓

Notifications continue

Status Page may fail

↓

Notifications continue

Parser may fail

↓

Notifications continue

This significantly increases platform resilience.

---

# 12.15 Operational Checks

Verify process

```
ps

systemd

screen

tmux
```

(depending on deployment method)

Verify wallet connectivity

```
getwalletinfo
```

Verify Telegram

```
Bot Token

Chat ID
```

Inspect log

```
/var/log/block_notify_xec.log
```

Verify state files

```
seen_coinbase_txids.txt

ignored_coinbase_txids.txt
```

---

# 12.16 Failure Scenarios

Wallet unavailable

↓

Monitoring suspended.

---

RPC authentication failure

↓

No transactions retrieved.

---

Telegram unavailable

↓

Rewards detected but notifications not delivered.

---

Filesystem full

↓

State files cannot be updated.

---

Invalid wallet configuration

↓

Monitoring initialization fails.

---

# 12.17 Operational Advantages

Compared with log-based notification systems, the wallet-based architecture provides several advantages.

- blockchain-backed detection;
- no dependency on CKPool logs;
- duplicate protection;
- restart-safe operation;
- complete transaction audit trail;
- deterministic behaviour;
- simplified troubleshooting.

---

# 12.18 Architectural Importance

Although implemented as a shell script, **block_notify_xec.sh** represents one of the most critical operational monitoring components of the platform.

By relying exclusively on wallet state rather than mining logs, it ensures that production notifications accurately reflect blockchain-confirmed mining rewards.

This design aligns with the overall architectural philosophy of the LMWPool platform:

**always trust the blockchain before trusting application logs.**
# 13. Binance Integration & Automatic XEC Conversion

## 13.1 Purpose

The Binance integration provides the operational bridge between the mining platform and the exchange infrastructure.

Unlike the mining services described in previous chapters, these utilities **do not participate in mining operations**.

Their purpose is to automate post-mining financial operations, primarily:

- XEC balance monitoring;
- automatic market conversion;
- exchange interaction;
- operational reporting.

The mining platform itself remains fully functional even if Binance becomes unavailable.

For this reason the exchange layer is considered an **auxiliary service**, completely separated from the mining engine.

---

# 13.2 Design Philosophy

The exchange integration follows four fundamental principles.

## Mining Independence

Mining must never depend on Binance.

If Binance becomes unavailable:

- miners continue hashing;
- CKPool continues submitting blocks;
- bitcoind continues synchronizing.

Only automatic conversion activities are affected.

---

## API Driven

Communication is performed exclusively through Binance REST APIs.

No browser automation.

No manual interaction.

No screen scraping.

---

## Credential Isolation

Exchange credentials are isolated from the rest of the platform.

Only Binance utilities know:

```
BINANCE_API_KEY

BINANCE_API_SECRET
```

No other production service requires these credentials.

---

## Failure Isolation

Failures inside the exchange layer never propagate toward:

- CKPool
- bitcoind
- Stats API
- Status Page
- Blockchain Notary

This separation significantly reduces operational risk.

---

# 13.3 Production Components

The Binance integration currently consists of:

```
binance_convert_xec_btc_real.py
```

Configuration

```
binance.env
```

Python runtime

HTTPS communication

REST API

---

# 13.4 Configuration

Sensitive information is stored outside the application code.

Typical configuration includes:

```
BINANCE_API_KEY

BINANCE_API_SECRET
```

Keeping credentials external allows:

- safer backups;
- easier key rotation;
- source code publication without exposing secrets.

Configuration files containing credentials should never be committed to version control.

---

# 13.5 Authentication

Every request is authenticated using:

API Key

+

API Secret

Requests requiring signatures are signed before transmission.

Unauthenticated public endpoints are used only where appropriate (for example, public market data).

---

# 13.6 Communication Flow

```
Python Utility

↓

Signed REST Request

↓

Binance API

↓

JSON Response

↓

Validation

↓

Operational Action
```

All responses are parsed before being processed.

Unexpected API responses should never trigger automatic financial operations.

---

# 13.7 Automatic Conversion Workflow

Typical workflow

```
Wallet Balance

↓

Threshold Verification

↓

Conversion Request

↓

Binance REST API

↓

Order Execution

↓

Confirmation

↓

Logging
```

The implementation is intentionally sequential.

Each step must complete successfully before the next begins.

---

# 13.8 Error Handling

Typical operational errors include:

Authentication failure

↓

Invalid API permissions

↓

Network timeout

↓

Insufficient balance

↓

Exchange maintenance

↓

Rate limiting

Each condition should be detected explicitly and recorded in operational logs.

Automatic retries should be conservative to avoid repeated failed requests.

---

# 13.9 API Permissions

Only the minimum required permissions should be enabled.

Recommended permissions:

✓ Read account information

✓ Spot trading (only if automatic conversion is enabled)

Permissions that are not required should remain disabled.

Withdrawal permissions should be enabled only when explicitly required by the operational workflow.

---

# 13.10 Security Considerations

API credentials represent one of the most sensitive assets of the platform.

Recommended practices include:

- never hard-code API keys;
- never expose credentials through logs;
- rotate keys periodically;
- restrict API permissions;
- protect configuration backups;
- monitor unauthorized API activity.

Whenever possible, API keys should be restricted according to Binance security capabilities (IP restrictions, permission scopes, etc.).

---

# 13.11 Operational Checks

Verify configuration

```
binance.env
```

Verify API connectivity

```
Account Information

↓

Successful Response
```

Verify authentication

No authorization errors.

Verify balances

Expected assets visible.

Verify conversion

Order executed successfully.

Verify logging

Operation correctly recorded.

---

# 13.12 Failure Scenarios

Invalid credentials

↓

Authentication failure.

---

Disabled API permissions

↓

Trading unavailable.

---

Regional restrictions

↓

Specific Binance features unavailable.

---

Internet outage

↓

No exchange communication.

---

Unexpected API changes

↓

Request parsing failure.

---

# 13.13 Relationship with the Mining Platform

The Binance integration is intentionally positioned **outside** the mining workflow.

```
Mining Platform

↓

Wallet

↓

(Optional)

Binance Utilities

↓

Exchange

↓

BTC
```

Notice that the mining engine itself is completely unaware of Binance.

This separation guarantees that exchange-related problems never interrupt mining operations.

---

# 13.14 Architectural Summary

The Binance integration provides controlled automation for post-mining financial operations while preserving complete independence between the mining infrastructure and the exchange layer.

This separation reflects one of the key architectural principles of the LMWPool platform:

> **Mining infrastructure must never depend on third-party financial services.**

By isolating exchange functionality from the core mining engine, the platform remains resilient, maintainable and capable of continuing production even during external exchange outages or service limitations.
# 14. Market Cache & Market Data Pipeline

## 14.1 Purpose

The LMWPool XEC platform intentionally separates **market data acquisition** from the public web interface.

Rather than allowing browsers to query external financial APIs directly, the platform periodically retrieves market information and stores it locally.

This architecture provides:

- lower latency;
- reduced dependency on third-party services;
- lower API usage;
- deterministic responses;
- improved website performance.

The market cache therefore acts as an intermediate layer between external exchanges and the public Status Page.

---

# 14.2 Design Philosophy

The market subsystem follows four simple principles.

**Acquire**

↓

**Validate**

↓

**Cache**

↓

**Publish**

Only validated information is published.

The Status Page never contacts external exchanges directly.

---

# 14.3 Architectural Position

```
            External Market API

                     │

               HTTPS Request

                     │

                     ▼

           Market Update Utility

                     │

              JSON Generation

                     │

                     ▼

        /srv/market-cache

                     │

                     ▼

                 Caddy

                     │

                     ▼

             /api/market

                     │

                     ▼

              Browser
```

This architecture completely isolates visitors from external providers.

---

# 14.4 Cache Directory

Production cache

```
/srv/market-cache
```

Current production file

```
market.json
```

The directory is mounted inside the Caddy container.

```
Host

↓

/root/docker-bitcoin-pool/market-cache

↓

Docker Volume

↓

/srv/market-cache
```

The cache therefore survives Caddy restarts.

---

# 14.5 Caddy Integration

The reverse proxy exposes the market cache using a dedicated route.

```
/api/market
```

Configuration

```
rewrite * /market.json

↓

root /srv/market-cache

↓

file_server
```

Unlike the remaining APIs, no backend container is contacted.

Caddy serves the JSON file directly from disk.

This minimizes latency.

---

# 14.6 Data Flow

Complete workflow

```
External Provider

↓

HTTPS

↓

Market Utility

↓

market.json

↓

Caddy

↓

HTTPS

↓

Browser
```

The browser remains unaware of the original data source.

---

# 14.7 Cache Refresh

Market data should be refreshed periodically.

Typical workflow

```
Retrieve Market Data

↓

Validate Response

↓

Generate JSON

↓

Atomic Replacement

↓

Publish
```

The previous cache remains available until the new version has been successfully written.

---

# 14.8 Atomic Update Strategy

The cache should never be overwritten directly.

Recommended workflow

```
market.json.tmp

↓

Complete Write

↓

Validation

↓

Rename

↓

market.json
```

This guarantees that visitors never receive partially written JSON documents.

---

# 14.9 Relationship with Status Page

The Status Page simply requests

```
/api/market
```

JavaScript receives

```
JSON

↓

Parsing

↓

Rendering
```

No financial calculation is performed inside the browser.

---

# 14.10 Failure Scenarios

## External API unavailable

Effect

Old cache continues to be served.

Website remains operational.

---

## Cache generation failure

Effect

Previous JSON remains available.

---

## Missing market.json

Effect

Market widget unavailable.

Mining unaffected.

---

## Invalid JSON

Effect

Browser parsing failure.

Other Status Page functions continue operating.

---

## Caddy unavailable

Effect

Market information unavailable together with the remaining HTTPS services.

---

# 14.11 Operational Checks

Verify cache file

```bash
ls -lh /root/docker-bitcoin-pool/market-cache
```

Verify JSON

```bash
cat market.json
```

Verify Caddy route

```bash
curl https://xec.lmwpool.com/api/market
```

Confirm:

- valid JSON;
- current timestamp;
- expected market values.

---

# 14.12 Security Considerations

The market cache intentionally exposes only public information.

No sensitive production information should ever be written into

```
market.json
```

The cache must never contain:

- API Keys;
- API Secrets;
- Wallet Addresses;
- Internal IP Addresses;
- Docker Configuration;
- Production Credentials.

Only publicly distributable market data belongs in the cache.

---

# 14.13 Performance Considerations

Serving a static JSON file directly through Caddy provides several advantages.

- no Python execution;
- no Node.js execution;
- no blockchain access;
- no database queries;
- minimal CPU usage;
- minimal memory usage.

Response time is limited almost entirely by disk access and HTTPS latency.

---

# 14.14 Relationship with the Platform

The Market Cache is completely independent from:

- CKPool;
- bitcoind-xec;
- parse_block.py;
- block_notify_xec.sh;
- Blockchain Notary.

Its only responsibility is to provide cached financial information.

Consequently, any failure inside the market subsystem has **zero impact** on mining operations.

---

# 14.15 Architectural Summary

Although relatively simple, the Market Cache represents an important optimization of the LMWPool platform.

By separating external market providers from the public website, it improves reliability, reduces latency, minimizes external API usage and guarantees deterministic responses for all users.

This architecture is consistent with the general design philosophy adopted throughout the platform:

**every production service should perform one clearly defined responsibility and remain independent from unrelated components.**
# 15. Maintenance & Utility Scripts

## 15.1 Purpose

Besides the primary production services, the LMWPool XEC platform contains a collection of utility scripts supporting daily operation, diagnostics, reporting and maintenance.

Although these scripts are not directly involved in mining, they significantly reduce operational effort and improve observability.

Typical responsibilities include:

- blockchain verification
- monitoring
- statistics generation
- market updates
- administrative maintenance
- Binance automation
- JSON generation

Most utilities are executed manually or by scheduled tasks and remain independent from the production mining workflow.

---

# 15.2 Design Philosophy

Every utility follows the same principles adopted across the platform.

- One script = one responsibility
- Independent execution
- Minimal dependencies
- Read production data
- Produce deterministic output

Whenever possible utilities should avoid modifying production services directly.

Instead they should:

Read

↓

Verify

↓

Generate

↓

Publish

---

# 15.3 Utility Classification

The current production utilities can be divided into several categories.

## Blockchain

Examples

```
parse_block.py
```

Purpose

Blockchain verification.

---

## Notifications

Examples

```
block_notify_xec.sh
```

Purpose

Wallet monitoring.

Telegram notifications.

---

## Exchange

Examples

```
binance_convert_xec_btc_real.py
```

Purpose

Automatic financial operations.

---

## Market

Purpose

Retrieve market prices.

Generate cache.

---

## Administration

Purpose

Routine maintenance.

Verification.

Diagnostics.

---

# 15.4 Operational Independence

Utilities never replace production services.

Instead they support them.

For example

CKPool

↓

Mining

Parser

↓

Verification

Telegram

↓

Notification

Status Page

↓

Presentation

Each utility adds value without increasing coupling.

---

# 15.5 Logging

Every production utility should generate its own log.

Recommended characteristics:

Dedicated log file

Timestamped entries

Human-readable messages

Explicit error reporting

No silent failures

Logging greatly simplifies incident analysis.

---

# 15.6 Error Handling

Utilities should fail safely.

Typical workflow

```
Operation

↓

Validation

↓

Success

↓

Publish
```

or

```
Operation

↓

Failure

↓

Log

↓

Exit
```

No partial updates should be published.

---

# 15.7 Atomic Operations

Whenever a utility generates production files, atomic replacement is recommended.

Workflow

```
temporary file

↓

complete write

↓

validation

↓

rename

↓

production file
```

This approach is already implemented in components such as

```
parse_block.py
```

and should be adopted by future utilities whenever possible.

---

# 15.8 Scheduling

Some utilities are event-driven.

Others are periodic.

Typical execution models include:

Manual execution

Cron

Systemd timers

Service loops

Wallet polling

The scheduling mechanism should be selected according to the operational role of each utility.

---

# 15.9 Development Guidelines

New utilities should follow the existing architectural standards.

Recommended practices include:

Single responsibility

Explicit configuration

Meaningful logging

Graceful error handling

Read-only access whenever possible

Minimal external dependencies

Consistent directory structure

This keeps long-term maintenance simple.

---

# 15.10 Backup Requirements

Utility scripts should be included in production backups.

Critical examples include:

```
parse_block.py

block_notify_xec.sh

market update scripts

Binance utilities

maintenance scripts
```

Although these files are relatively small, rebuilding them from memory after a disaster would require significant effort.

---

# 15.11 Recovery

Following a server rebuild the recommended recovery order is:

Restore utility scripts

↓

Verify permissions

↓

Verify configuration

↓

Test execution

↓

Reconnect production services

↓

Validate generated output

Utilities should always be tested before reconnecting them to the production workflow.

---

# 15.12 Future Expansion

The architecture intentionally allows additional utilities to be introduced without modifying the core mining platform.

Potential future utilities include:

- health monitoring
- automatic backups
- blockchain analytics
- miner diagnostics
- performance reports
- infrastructure auditing
- notification extensions

Because every utility remains independent, future development does not require changes to CKPool or bitcoind.

---

# 15.13 Operational Inventory

The current support layer includes utilities responsible for:

| Utility | Primary Responsibility |
|----------|------------------------|
| parse_block.py | Blockchain verification |
| block_notify_xec.sh | Wallet monitoring & Telegram |
| Binance utilities | Exchange automation |
| Market utilities | Cached market generation |
| Maintenance scripts | Diagnostics & administration |

Together these utilities provide the operational capabilities that transform the core mining engine into a complete production platform.

---

# 15.14 Architectural Summary

The utility layer represents one of the distinguishing characteristics of the LMWPool XEC platform.

Rather than concentrating all functionality into a single application, operational responsibilities are distributed across small, purpose-built components.

This modular architecture provides:

- easier debugging;
- lower operational risk;
- simpler maintenance;
- greater flexibility;
- improved disaster recovery;
- straightforward future expansion.

The result is an infrastructure that remains robust, understandable and maintainable even as additional services are introduced.

# 16. Complete Support Services Workflow

## 16.1 Purpose

This chapter provides a complete operational view of every support service composing the LMWPool XEC platform.

While previous chapters documented each component individually, this chapter explains how they collaborate during normal production operation.

It should be considered the primary reference for:

- infrastructure understanding;
- operational troubleshooting;
- disaster recovery;
- onboarding of new administrators.

---

# 16.2 Complete Platform Topology

```
                                 INTERNET

                                     │

                              HTTPS / STRATUM

                                     │

                         +-----------------------+
                         |      Caddy v2         |
                         |   Shared Reverse      |
                         |        Proxy          |
                         +-----------+-----------+
                                     │
                +--------------------+----------------------+
                |                                           |
                |                                           |
                ▼                                           ▼

      +------------------+                     +----------------------+
      | status-page-xec  |                     |   XEC Notary         |
      |     NGINX        |                     |     Node.js          |
      +---------+--------+                     +----------+-----------+
                │                                          │
                │                                          │
                ▼                                          ▼

      +------------------+                       +------------------+
      | stats-api-xec    |                       | SQLite Database  |
      | Python API       |                       +------------------+
      +---------+--------+
                │
                │
                ▼

      +------------------+
      | mined_blocks.json|
      +---------+--------+
                ▲
                │
                │
      +---------+--------+
      | parse_block.py   |
      +---------+--------+
                │
                │
                ▼

      +------------------+
      | ckpool.log       |
      +---------+--------+
                ▲
                │
                │
      +---------+--------+
      | CKPool Solo      |
      +---------+--------+
                │
       RPC      │      ZMQ
                │
                ▼

      +------------------+
      | bitcoind-xec     |
      | Full Node        |
      +---------+--------+
                │
                │
                ▼

         eCash Blockchain

```

---

# 16.3 Production Responsibilities

| Component | Primary Responsibility | Critical |
|------------|-----------------------|----------|
| bitcoind-xec | Blockchain consensus | YES |
| CKPool | Mining engine | YES |
| Caddy | HTTPS reverse proxy | YES (public services) |
| stats-api-xec | Statistics | NO |
| status-page-xec | Web UI | NO |
| parse_block.py | Block verification | NO |
| block_notify_xec.sh | Telegram alerts | NO |
| Market Cache | Market prices | NO |
| Binance Utilities | Exchange automation | NO |
| Blockchain Notary | Notarization | Independent |

Mining depends only on:

- bitcoind-xec
- CKPool

Everything else enhances the platform but does not participate in Proof-of-Work.

---

# 16.4 Production Dependency Matrix

| Service | Depends On |
|----------|------------|
| CKPool | bitcoind-xec |
| parse_block.py | CKPool + bitcoind |
| stats-api-xec | JSON files |
| status-page-xec | stats-api-xec |
| Caddy | backend containers |
| block_notify_xec.sh | Wallet RPC |
| Market Cache | External Market API |
| Binance Utilities | Binance REST API |
| Notary | bitcoind-xec (RPC) |

One of the platform's strengths is the low number of dependencies between services.

---

# 16.5 Operational Data Flow

The complete mining information flow is:

```
ASIC

↓

CKPool

↓

ckpool.log

↓

parse_block.py

↓

RPC verification

↓

mined_blocks.json

↓

stats-api-xec

↓

status-page-xec

↓

Browser
```

---

# 16.6 Notification Flow

Notifications follow a completely separate path.

```
Wallet

↓

listtransactions()

↓

block_notify_xec.sh

↓

Validation

↓

Telegram Bot

↓

Administrator
```

The notification subsystem remains operational even if the Status Page is unavailable.

---

# 16.7 Market Data Flow

```
External Provider

↓

Market Utility

↓

market.json

↓

Caddy

↓

Browser
```

No interaction with the mining engine occurs.

---

# 16.8 Blockchain Notary Flow

```
Browser

↓

HTTPS

↓

Caddy

↓

server.js

↓

SQLite

↓

bitcoind RPC

↓

Blockchain
```

The Notary shares the blockchain node but remains logically independent from mining.

---

# 16.9 Service Criticality

## Level 1 (Core Infrastructure)

Failure immediately interrupts mining.

- bitcoind-xec
- CKPool

---

## Level 2 (Platform Infrastructure)

Mining continues but public services degrade.

- Caddy
- Docker networking

---

## Level 3 (Operational Services)

Mining operations continue normally.

- Stats API
- Status Page
- Parser
- Telegram
- Market Cache

---

## Level 4 (Business Services)

Independent from mining.

- Binance integration
- Blockchain Notary

---

# 16.10 Startup Order

Recommended production startup sequence.

```
1.

bitcoind-xec

↓

2.

Blockchain synchronization

↓

3.

CKPool

↓

4.

parse_block.py

↓

5.

stats-api-xec

↓

6.

status-page-xec

↓

7.

Caddy

↓

8.

block_notify_xec.sh

↓

9.

Blockchain Notary

↓

10.

Binance utilities
```

---

# 16.11 Shutdown Order

Recommended shutdown sequence.

```
Stop miners

↓

CKPool

↓

Support services

↓

Notary

↓

bitcoind-xec
```

The blockchain node should always be stopped last.

---

# 16.12 Recovery Priority

Following a disaster recovery, components should be restored in this order:

1. Linux host
2. Docker
3. Docker networks
4. bitcoind-xec
5. Wallet
6. CKPool
7. Parser
8. Stats API
9. Status Page
10. Caddy
11. Notification service
12. Notary
13. Binance utilities

This order minimizes downtime and avoids cascading failures.

---

# 16.13 Health Verification Checklist

A production platform is considered healthy when all the following conditions are satisfied:

✓ Blockchain synchronized

✓ Wallet available

✓ CKPool accepting shares

✓ Miners connected

✓ Parser generating JSON

✓ Stats API responding

✓ Status Page updating

✓ HTTPS available

✓ Telegram notifications operational

✓ Notary responding

✓ Market cache available

---

# 16.14 Monitoring Priorities

The recommended monitoring priority is:

1. Blockchain synchronization
2. CKPool health
3. Connected miners
4. Accepted shares
5. Solved blocks
6. Wallet health
7. HTTPS availability
8. API availability
9. Telegram notifications
10. Notary health

Any monitoring platform should alert in approximately this order.

---

# 16.15 Operational Principles

The LMWPool XEC platform is intentionally designed around a small number of independent services.

The architecture emphasizes:

- separation of responsibilities;
- deterministic behavior;
- local blockchain authority;
- minimal service coupling;
- straightforward disaster recovery;
- operational transparency.

Every service has a clearly defined role, and no component performs responsibilities that belong to another layer.

---

# 16.16 Part 2 Summary

The support services documented in this section transform the core mining engine into a complete production platform.

Together they provide:

- operational visibility;
- public statistics;
- verified solved blocks;
- market information;
- Telegram notifications;
- exchange automation;
- presentation services.

Most importantly, they achieve these goals **without increasing the complexity of the mining engine itself**.

This concludes **Volume 2 – Mining Support Services**.

The next section of this manual (**Part 3**) documents the **XEC Blockchain Notary**, including its Node.js architecture, SQLite database, REST API, blockchain integration, certificate generation, email workflow and operational procedures.

---

# RC1 Review Notes

This RC1 consolidates Volume 2 without changing the production architecture.

Main editorial improvements include:

- consistent terminology across support services;
- clearer separation between presentation, API and mining layers;
- normalized technical English;
- improved emphasis on service independence and production responsibilities.

Detailed inventories, configuration listings, operational commands and runbooks are intentionally deferred to the dedicated appendices (Appendices A–F). Future Release 1.0 will include explicit section-level cross references.
