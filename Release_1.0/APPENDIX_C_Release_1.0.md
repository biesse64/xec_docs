# LMWPool XEC Platform
## Appendix C — Production Configuration Reference

| Document metadata | Value |
|---|---|
| Document | Technical Documentation |
| Appendix | C |
| Title | Production Configuration Reference |
| Release | 1.0 |
| Platform | LMWPool XEC Production Platform |
| Environment | Production |
| Status | Definitive release |

> **Document scope.** This appendix documents the production configuration assets used by the LMWPool platform. It defines their responsibilities, management principles and operational controls. Sensitive information—including passwords, API keys, private keys and other secrets—is intentionally excluded.

---

## Document Control

> **Release note:** This document is Appendix C of the LMWPool Technical Documentation Release 1.0 set. Read it together with Volumes 1–4 and Appendices A–F for the complete platform reference.

| Field | Value |
|---|---|
| Release | 1.0 |
| Source baseline | Appendix C RC2 |
| Intended audience | Platform owner, system administrators, technical maintainers |
| Related references | Volume 4 — Infrastructure Operations; Appendix A — Infrastructure Inventory; Appendix E — Operational Runbooks |
| Change policy | Production configuration changes must be reflected in the corresponding documentation |
| Operational authority | Running production configuration and verified operational evidence |

Unlike the main volumes, this appendix serves as the definitive configuration-management reference. It documents production configuration files only.

## Table of Contents

1. [Configuration Philosophy](#c1-configuration-philosophy)
2. [Configuration Overview](#c2-configuration-overview)
3. [Docker Compose Files](#c3-docker-compose-files)
4. [Docker Networks](#c4-docker-networks)
5. [CKPool Configuration](#c5-ckpool-configuration)
6. [`bitcoind-xec` Configuration](#c6-bitcoind-xec-configuration)
7. [Caddy Configuration](#c7-caddy-configuration)
8. [Statistics API Configuration](#c8-statistics-api-configuration)
9. [Status Page Configuration](#c9-status-page-configuration)
10. [Blockchain Notary Configuration](#c10-blockchain-notary-configuration)
11. [SQLite Configuration](#c11-sqlite-configuration)
12. [Environment Variables](#c12-environment-variables)
13. [Certificates](#c13-certificates)
14. [Configuration Backup](#c14-configuration-backup)
15. [Configuration Change Management](#c15-configuration-change-management)
16. [Appendix Summary](#c16-appendix-summary)

---

## C.1 Configuration Philosophy

### C.1.1 Purpose

Configuration files define the operational behavior of every production component.

The platform follows these principles:

- explicit configuration;
- human-readable files;
- version-controlled configuration;
- minimal duplication;
- separation of configuration from source code; and
- deterministic deployment.

### C.1.2 Configuration Categories

Configuration assets are grouped into six categories.

| Category | Examples |
|---|---|
| Infrastructure | Docker Compose |
| Networking | Docker networks |
| Blockchain | `xec.conf` |
| Mining | `ckpool.conf` |
| Reverse proxy | `Caddyfile` |
| Applications | `server.js`, `package.json` |

### C.1.3 Configuration Ownership

Every configuration file has a clearly defined responsibility. Configuration responsibilities should not overlap between components. Each service is responsible only for its own operational parameters.

---

## C.2 Configuration Overview

### C.2.1 Primary Configuration Assets

The production platform is configured primarily through the following files.

| Configuration file | Component |
|---|---|
| `docker-compose.yml` | Docker orchestration |
| `Caddyfile` | Reverse proxy |
| `ckpool-xec.conf` | CKPool |
| `xec.conf` | Blockchain node |
| `server.js` | Blockchain Notary |
| `package.json` | Node.js dependencies |

The production inventory confirms these assets as the primary operational configuration files.

### C.2.2 Configuration Hierarchy

```text
Linux Host
        │
Docker Compose
        │
Containers
        │
Application Configuration
        │
Business Logic
```

Configuration flows downward through this hierarchy. Applications never modify infrastructure configuration automatically.

---

## C.3 Docker Compose Files

### C.3.1 Purpose

Docker Compose files define:

- containers;
- networks;
- ports;
- restart policies;
- bind mounts; and
- environment variables.

They are the primary deployment mechanism for Docker services.

### C.3.2 Typical Contents

A Compose project generally defines:

- services;
- image names;
- build directives;
- volumes;
- published ports;
- restart policies; and
- Docker networks.

### C.3.3 Operational Guidelines

Docker Compose files should remain:

- readable;
- modular;
- documented; and
- version controlled.

Production configuration should always be validated before deployment.

---

## C.4 Docker Networks

### C.4.1 Purpose

Docker networking provides communication paths between production containers while maintaining service isolation. Networks should include only containers that require direct communication.

### C.4.2 Configuration Elements

Typical network parameters include:

- driver;
- subnet;
- gateway;
- aliases; and
- attached containers.

### C.4.3 Network Isolation

Internal services should never publish ports solely for inter-container communication. Docker networking replaces unnecessary host exposure.

---

## C.5 CKPool Configuration

### C.5.1 Purpose

The CKPool configuration defines the operational behavior of the solo-mining server.

Primary configuration areas include:

- Stratum listeners;
- share difficulty;
- RPC connectivity;
- logging; and
- worker handling.

### C.5.2 Typical Parameters

Examples include:

- pool name;
- listening ports;
- RPC endpoint;
- authentication;
- logging options; and
- difficulty configuration.

Actual production values are intentionally omitted from this appendix when they contain sensitive operational information.

### C.5.3 Change Policy

Any modification to CKPool configuration should be followed by:

1. configuration validation;
2. controlled restart;
3. log inspection; and
4. miner-connectivity verification.

---

## C.6 `bitcoind-xec` Configuration

### C.6.1 Purpose

The blockchain-node configuration controls synchronization, networking and RPC services.

### C.6.2 Major Sections

Typical parameters include:

- RPC server;
- wallet;
- peer networking;
- ZMQ notifications;
- indexing; and
- logging.

### C.6.3 Security

RPC credentials should never appear in documentation. Authentication material must remain external to version-controlled documentation.

---

## C.7 Caddy Configuration

### C.7.1 Purpose

The Caddy reverse proxy is the public HTTPS gateway of the LMWPool production platform.

Its responsibilities include:

- HTTPS termination;
- automatic TLS-certificate management;
- reverse-proxy services;
- request routing;
- HTTP-to-HTTPS redirection; and
- security-header management.

No backend production service should be exposed directly to the Internet when Caddy is intended to proxy that service.

### C.7.2 Main Configuration Sections

The production `Caddyfile` is organized into several logical sections.

| Section | Purpose |
|---|---|
| Global options | Global server behavior |
| HTTPS configuration | TLS management |
| Virtual hosts | Domain configuration |
| Reverse proxy | Backend routing |
| Static content | Website resources |
| Logging | Access and error logs |

### C.7.3 Reverse-Proxy Rules

Typical reverse-proxy definitions include:

- Status Page;
- Statistics API;
- Blockchain Notary;
- administrative endpoints; and
- static assets.

Every routing rule should be explicitly documented and validated after modification.

### C.7.4 TLS Certificates

TLS certificates are automatically managed by Caddy.

Operational responsibilities include:

- certificate issuance;
- automatic renewal;
- secure storage; and
- HTTPS enforcement.

Manual certificate management is normally unnecessary.

### C.7.5 Configuration Validation

Validate the configuration before deployment:

```bash
caddy validate --config /etc/caddy/Caddyfile
```

Reload the configuration:

```bash
caddy reload --config /etc/caddy/Caddyfile
```

Configuration validation should always precede production deployment.

---

## C.8 Statistics API Configuration

### C.8.1 Purpose

The Statistics API provides operational information for miners and public dashboards.

Its configuration determines:

- available endpoints;
- cache behavior;
- backend communication;
- refresh intervals; and
- logging.

### C.8.2 Configuration Areas

Typical configuration includes:

- API port;
- backend endpoint;
- cache policy;
- timeout values; and
- logging configuration.

### C.8.3 Operational Guidelines

The Statistics API should remain stateless whenever possible. Persistent information belongs to the blockchain or dedicated databases rather than the API layer.

---

## C.9 Status Page Configuration

### C.9.1 Purpose

The Status Page provides the mining platform's public user interface. It contains no mining logic.

Its responsibilities include:

- presentation;
- dashboard rendering;
- API consumption; and
- user interaction.

### C.9.2 Configuration

Typical configuration includes:

- API endpoint URLs;
- refresh intervals;
- UI settings;
- branding; and
- static resources.

### C.9.3 Design Principle

Business logic should never be implemented within the Status Page. The user interface consumes APIs but should remain independent of backend processing.

---

## C.10 Blockchain Notary Configuration

### C.10.1 Purpose

The Blockchain Notary configuration controls the behavior of the document-notarization platform.

Primary configuration areas include:

- HTTP server;
- blockchain connectivity;
- SQLite database;
- SMTP services;
- PDF generation; and
- authentication.

### C.10.2 Main Configuration Files

Typical files include:

| File | Purpose |
|---|---|
| `server.js` | Main application |
| `package.json` | Node.js dependencies |
| `package-lock.json` | Dependency lock |
| Configuration files | Runtime configuration |

### C.10.3 Sensitive Parameters

The following values must never be stored in documentation:

- SMTP credentials;
- RPC passwords;
- API keys;
- private authentication keys; and
- encryption secrets.

Configuration templates may document parameter names without exposing operational values.

---

## C.11 SQLite Configuration

### C.11.1 Purpose

SQLite provides lightweight persistent storage for the Blockchain Notary platform.

Typical stored information includes:

- registered users;
- document hashes;
- notarization records;
- transaction references; and
- application metadata.

### C.11.2 Database Management

Operational procedures include:

- regular backup;
- integrity verification;
- controlled maintenance; and
- schema-version tracking.

### C.11.3 Database Protection

Database files should remain accessible only to authorized production services. Direct manual modification is discouraged except during controlled maintenance procedures.

---

## C.12 Environment Variables

### C.12.1 Purpose

Environment variables provide runtime configuration without embedding sensitive values directly in source code.

Examples include:

- SMTP server;
- RPC endpoint;
- API tokens;
- application mode; and
- logging level.

### C.12.2 Security Principles

Sensitive operational values should never be committed to version-controlled repositories. Environment-specific values should remain external to application code whenever practical.

### C.12.3 Documentation Policy

Documentation should identify:

- variable names;
- functional purpose; and
- expected format.

Documentation should never include actual production secrets.

---

## C.13 Certificates

### C.13.1 Purpose

Digital certificates provide encrypted communication between the production platform and external clients.

The platform relies on publicly trusted TLS certificates to protect:

- administrative access;
- public web services;
- Blockchain Notary APIs; and
- statistics endpoints.

Certificate management is centralized within the reverse-proxy layer.

### C.13.2 Certificate Lifecycle

Certificate management follows a fully automated lifecycle:

1. Certificate request.
2. Domain validation.
3. Certificate installation.
4. Automatic renewal.
5. Expiration monitoring.

Automation minimizes operational effort while reducing the risk of certificate expiration.

### C.13.3 Protected Services

TLS protection is applied to all externally accessible HTTP services.

Typical protected services include:

| Service | Protection |
|---|---|
| Status Page | HTTPS |
| Statistics API | HTTPS |
| Blockchain Notary | HTTPS |
| Administrative web interfaces | HTTPS |

Mining Stratum traffic follows the protocol requirements defined by the mining software and is managed independently of HTTPS services.

### C.13.4 Certificate Storage

Certificate storage should satisfy the following requirements:

- restricted filesystem permissions;
- automatic renewal;
- regular backup;
- controlled access; and
- secure private-key protection.

Private keys must never be copied into documentation or public repositories.

---

## C.14 Configuration Backup

### C.14.1 Purpose

Configuration files are among the most valuable production assets. A complete configuration backup enables rapid reconstruction of the production platform following hardware failure, accidental deletion or infrastructure migration.

### C.14.2 Backup Scope

Configuration backups should include:

- Docker Compose files;
- Docker network definitions;
- CKPool configuration;
- `bitcoind-xec` configuration;
- Caddy configuration;
- Blockchain Notary configuration;
- `systemd` service files; and
- operational scripts.

Generated files and temporary artifacts should not be included unless explicitly required.

### C.14.3 Backup Strategy

Recommended backup policy:

| Component | Recommended frequency |
|---|---|
| Configuration files | After every approved modification |
| Docker Compose | After each infrastructure change |
| Reverse proxy | After routing updates |
| Blockchain node | After configuration changes |
| Notary application | After application releases |

Version history should be retained whenever practical.

### C.14.4 Backup Verification

Backups should be validated periodically by restoring them in a non-production environment.

Validation should confirm:

- configuration readability;
- syntax correctness;
- application startup;
- successful container deployment; and
- expected service behavior.

A backup that has never been tested should not be considered fully reliable.

---

## C.15 Configuration Change Management

### C.15.1 Purpose

Production configuration changes must be performed in a controlled and documented manner. The objective is to minimize operational risk while maintaining complete traceability of infrastructure evolution.

### C.15.2 Change Workflow

Recommended workflow:

```text
Configuration Change
          │
          ▼
Documentation Update
          │
          ▼
Configuration Validation
          │
          ▼
Controlled Deployment
          │
          ▼
Operational Verification
          │
          ▼
Documentation Revision
```

Every production change should follow this sequence.

### C.15.3 Version Control

Configuration files should remain under version control whenever possible.

Version control provides:

- complete history;
- rollback capability;
- change traceability;
- collaborative maintenance; and
- configuration comparison.

Configuration repositories should exclude all confidential information.

### C.15.4 Production Approval

Before deployment, configuration changes should be verified for:

- syntax validity;
- dependency consistency;
- service compatibility;
- rollback availability; and
- operational impact.

Only validated configurations should be promoted to production.

### C.15.5 Documentation Synchronization

Configuration documentation should always reflect the production environment. Whenever a configuration file changes, the corresponding documentation should be reviewed to maintain alignment between operational reality and the technical documentation.

---

## C.16 Appendix Summary

This appendix defines the configuration reference for the complete LMWPool production platform. It documents the configuration assets governing:

- infrastructure deployment;
- container orchestration;
- blockchain services;
- mining operations;
- reverse-proxy services;
- Blockchain Notary;
- application runtime;
- certificates; and
- the configuration lifecycle.

Together with the preceding volumes and appendices, this document establishes the baseline configuration model for the production platform.

Future revisions should continue to reflect the deployed production environment while preserving the principles of explicit configuration, version control and operational reproducibility.

---

**End of Appendix C — Production Configuration Reference**
