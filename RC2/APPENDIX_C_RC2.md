# RC2 Editorial Revision Notice

**Release:** RC2 (Editorial Review Candidate)

This revision preserves the production configuration reference documented in RC1 while aligning the appendix with the editorial standards adopted across the complete LMWPool documentation suite.

## RC2 objectives

- Standardize terminology across Volumes 1–4 and Appendices A–F.
- Prepare unified numbering for figures, tables and configuration listings.
- Prepare explicit cross-references to operational procedures and infrastructure inventory.
- Improve publication consistency for Release 1.0.

---

# APPENDIX C

# Configuration Reference

**Document:** LMWPool Technical Documentation  
**Appendix:** C  
**Title:** Production Configuration Reference  
**Platform:** LMWPool XEC Production Platform  
**Release:** RC1

---

# Document Purpose

This appendix documents every production configuration file used by the LMWPool platform.

Unlike the previous volumes, this appendix is intended to serve as the definitive reference for configuration management.

Only production configuration files are documented. Future Release 1.0 will introduce explicit section-level references to Volume 4 (Infrastructure Operations), Appendix A (Infrastructure Inventory) and Appendix E (Operational Runbooks).

Sensitive information (passwords, API keys, private keys and secrets) is intentionally excluded.

---

# Table of Contents

C.1 Configuration Philosophy

C.2 Configuration Overview

C.3 Docker Compose Files

C.4 Docker Networks

C.5 CKPool Configuration

C.6 bitcoind-xec Configuration

C.7 Caddy Configuration

C.8 Statistics API Configuration

C.9 Status Page Configuration

C.10 Blockchain Notary Configuration

C.11 SQLite Configuration

C.12 Environment Variables

C.13 Certificates

C.14 Configuration Backup

C.15 Configuration Change Management

C.16 Appendix Summary

---

# C.1 Configuration Philosophy

## C.1.1 Purpose

Configuration files define the operational behaviour of every production component.

The platform follows several principles.

- Explicit configuration.
- Human-readable files.
- Version-controlled configuration.
- Minimal duplication.
- Separation of configuration from source code.
- Deterministic deployment.

---

## C.1.2 Configuration Categories

Configuration assets are grouped into six categories.

| Category | Examples |
|----------|----------|
| Infrastructure | Docker Compose |
| Networking | Docker Networks |
| Blockchain | xec.conf |
| Mining | ckpool.conf |
| Reverse Proxy | Caddyfile |
| Applications | server.js, package.json |

---

## C.1.3 Configuration Ownership

Every configuration file has a clearly defined responsibility.

Configuration should never overlap between components.

Each service is responsible only for its own operational parameters.

---

# C.2 Configuration Overview

## C.2.1 Primary Configuration Assets

The production platform is configured primarily through the following files.

| Configuration File | Component |
|--------------------|-----------|
| docker-compose.yml | Docker orchestration |
| Caddyfile | Reverse Proxy |
| ckpool-xec.conf | CKPool |
| xec.conf | Blockchain node |
| server.js | Blockchain Notary |
| package.json | Node.js dependencies |

The production inventory confirms these configuration assets as the primary operational configuration files.

---

## C.2.2 Configuration Hierarchy

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

Configuration flows downward.

Applications never modify infrastructure configuration automatically.

---

# C.3 Docker Compose Files

## C.3.1 Purpose

Docker Compose files define:

- containers;
- networks;
- ports;
- restart policies;
- bind mounts;
- environment variables.

They represent the primary deployment mechanism for Docker services.

---

## C.3.2 Typical Contents

A Compose project generally defines:

- services;
- image names;
- build directives;
- volumes;
- published ports;
- restart policy;
- Docker networks.

---

## C.3.3 Operational Guidelines

Docker Compose files should remain:

- readable;
- modular;
- documented;
- version controlled.

Production configuration should always be validated before deployment.

---

# C.4 Docker Networks

## C.4.1 Purpose

Docker networking provides communication paths between production containers while maintaining service isolation.

Networks should only include containers requiring direct communication.

---

## C.4.2 Configuration Elements

Typical network parameters include:

- driver;
- subnet;
- gateway;
- aliases;
- attached containers.

---

## C.4.3 Network Isolation

Internal services should never publish ports solely for inter-container communication.

Docker networking replaces unnecessary host exposure.

---

# C.5 CKPool Configuration

## C.5.1 Purpose

The CKPool configuration defines the operational behaviour of the solo mining server.

Primary configuration areas include:

- Stratum listeners;
- share difficulty;
- RPC connectivity;
- logging;
- worker handling.

---

## C.5.2 Typical Parameters

Examples include:

- pool name;
- listening ports;
- RPC endpoint;
- authentication;
- logging options;
- difficulty configuration.

Actual production values are intentionally omitted from this appendix where they contain sensitive operational information.

---

## C.5.3 Change Policy

Any modification to CKPool configuration should be followed by:

1. configuration validation;
2. controlled restart;
3. log inspection;
4. miner connectivity verification.

---

# C.6 bitcoind-xec Configuration

## C.6.1 Purpose

The blockchain node configuration controls synchronization, networking and RPC services.

---

## C.6.2 Major Sections

Typical parameters include:

- RPC server;
- wallet;
- peer networking;
- ZMQ notifications;
- indexing;
- logging.

---

## C.6.3 Security

RPC credentials should never appear in documentation.

Authentication material must remain external to version-controlled documentation.

---

**End of Appendix C — Part 1**

*(Continues with Caddy, Statistics API, Status Page, Blockchain Notary, SQLite, Environment Variables, Certificates and Change Management.)*
---

# C.7 Caddy Configuration

## C.7.1 Purpose

The Caddy reverse proxy represents the public gateway of the LMWPool production platform.

Its responsibilities include:

- HTTPS termination;
- automatic TLS certificate management;
- reverse proxy services;
- request routing;
- HTTP to HTTPS redirection;
- security header management.

No backend production service should be directly exposed to the Internet when Caddy is intended to proxy that service.

---

## C.7.2 Main Configuration Sections

The production Caddyfile is organized into several logical sections.

| Section | Purpose |
|----------|---------|
| Global Options | Global server behaviour |
| HTTPS Configuration | TLS management |
| Virtual Hosts | Domain configuration |
| Reverse Proxy | Backend routing |
| Static Content | Website resources |
| Logging | Access and error logs |

---

## C.7.3 Reverse Proxy Rules

Typical reverse proxy definitions include:

- Status Page
- Statistics API
- Blockchain Notary
- Administrative endpoints
- Static assets

Every routing rule should be explicitly documented and validated after modification.

---

## C.7.4 TLS Certificates

TLS certificates are automatically managed by Caddy.

Operational responsibilities include:

- certificate issuance;
- automatic renewal;
- secure storage;
- HTTPS enforcement.

Manual certificate management is normally unnecessary.

---

## C.7.5 Configuration Validation

Before deployment:

```bash
caddy validate --config /etc/caddy/Caddyfile
```

Reload configuration.

```bash
caddy reload --config /etc/caddy/Caddyfile
```

Configuration validation should always precede production deployment.

---

# C.8 Statistics API Configuration

## C.8.1 Purpose

The Statistics API provides operational information for miners and public dashboards.

Its configuration determines:

- available endpoints;
- cache behaviour;
- backend communication;
- refresh intervals;
- logging.

---

## C.8.2 Configuration Areas

Typical configuration includes:

- API port;
- backend endpoint;
- cache policy;
- timeout values;
- logging configuration.

---

## C.8.3 Operational Guidelines

The Statistics API should remain stateless whenever possible.

Persistent information belongs to the blockchain or dedicated databases rather than the API layer.

---

# C.9 Status Page Configuration

## C.9.1 Purpose

The Status Page provides the public user interface of the mining platform.

It contains no mining logic.

Its responsibilities include:

- presentation;
- dashboard rendering;
- API consumption;
- user interaction.

---

## C.9.2 Configuration

Typical configuration includes:

- API endpoint URLs;
- refresh intervals;
- UI settings;
- branding;
- static resources.

---

## C.9.3 Design Principle

Business logic should never be implemented within the Status Page.

The user interface consumes APIs but should remain independent of backend processing.

---

# C.10 Blockchain Notary Configuration

## C.10.1 Purpose

The Blockchain Notary configuration controls the behaviour of the document notarization platform.

Primary configuration areas include:

- HTTP server;
- blockchain connectivity;
- SQLite database;
- SMTP services;
- PDF generation;
- authentication.

---

## C.10.2 Main Configuration Files

Typical files include:

| File | Purpose |
|------|---------|
| server.js | Main application |
| package.json | Node.js dependencies |
| package-lock.json | Dependency lock |
| configuration files | Runtime configuration |

---

## C.10.3 Sensitive Parameters

The following values must never be stored in documentation.

- SMTP credentials;
- RPC passwords;
- API keys;
- private authentication keys;
- encryption secrets.

Configuration templates may document parameter names without exposing operational values.

---

# C.11 SQLite Configuration

## C.11.1 Purpose

SQLite provides lightweight persistent storage for the Blockchain Notary platform.

Typical stored information includes:

- registered users;
- document hashes;
- notarization records;
- transaction references;
- application metadata.

---

## C.11.2 Database Management

Operational procedures include:

- regular backup;
- integrity verification;
- controlled maintenance;
- schema version tracking.

---

## C.11.3 Database Protection

Database files should remain accessible only to authorized production services.

Direct manual modification is discouraged except during controlled maintenance procedures.

---

# C.12 Environment Variables

## C.12.1 Purpose

Environment variables provide runtime configuration without embedding sensitive values directly within source code.

Examples include:

- SMTP server;
- RPC endpoint;
- API tokens;
- application mode;
- logging level.

---

## C.12.2 Security Principles

Sensitive operational values should never be committed into version-controlled repositories.

Environment-specific values should remain external to application code whenever practical.

---

## C.12.3 Documentation Policy

Documentation should identify:

- variable names;
- functional purpose;
- expected format.

Documentation should never include actual production secrets.

---

**End of Appendix C — Part 2**

*(Continues with Certificates, Configuration Backup, Configuration Change Management and Appendix Summary.)*
---

# C.13 Certificates

## C.13.1 Purpose

Digital certificates provide encrypted communications between the production platform and external clients.

The platform relies on publicly trusted TLS certificates to protect:

- administrative access;
- public web services;
- Blockchain Notary APIs;
- statistics endpoints.

Certificate management is centralized within the reverse proxy layer.

---

## C.13.2 Certificate Lifecycle

Certificate management follows a fully automated lifecycle.

The lifecycle consists of:

1. Certificate request.
2. Domain validation.
3. Certificate installation.
4. Automatic renewal.
5. Expiration monitoring.

Automation minimizes operational effort while reducing the risk of certificate expiration.

---

## C.13.3 Protected Services

TLS protection is applied to all externally accessible HTTP services.

Typical protected services include:

| Service | Protection |
|----------|------------|
| Status Page | HTTPS |
| Statistics API | HTTPS |
| Blockchain Notary | HTTPS |
| Administrative Web Interfaces | HTTPS |

Mining Stratum traffic follows the protocol requirements defined by the mining software and is managed independently from HTTPS services.

---

## C.13.4 Certificate Storage

Certificate storage should satisfy the following requirements.

- Restricted filesystem permissions.
- Automatic renewal.
- Regular backup.
- Controlled access.
- Secure private key protection.

Private keys must never be copied into documentation or public repositories.

---

# C.14 Configuration Backup

## C.14.1 Purpose

Configuration files represent one of the most valuable production assets.

A complete configuration backup allows rapid reconstruction of the production platform following hardware failure, accidental deletion or infrastructure migration.

---

## C.14.2 Backup Scope

Configuration backups should include:

- Docker Compose files;
- Docker network definitions;
- CKPool configuration;
- bitcoind-xec configuration;
- Caddy configuration;
- Blockchain Notary configuration;
- systemd service files;
- operational scripts.

Generated files and temporary artifacts should not be included unless explicitly required.

---

## C.14.3 Backup Strategy

Recommended backup policy:

| Component | Recommended Frequency |
|-----------|----------------------|
| Configuration files | After every approved modification |
| Docker Compose | After each infrastructure change |
| Reverse Proxy | After routing updates |
| Blockchain Node | After configuration changes |
| Notary Application | After application releases |

Version history should be retained whenever practical.

---

## C.14.4 Backup Verification

Backups should be periodically validated by restoring them into a non-production environment.

Validation should confirm:

- configuration readability;
- syntax correctness;
- application startup;
- successful container deployment;
- expected service behavior.

A backup that has never been tested should not be considered fully reliable.

---

# C.15 Configuration Change Management

## C.15.1 Purpose

Production configuration changes must be performed in a controlled and documented manner.

The objective is to minimize operational risk while maintaining complete traceability of infrastructure evolution.

---

## C.15.2 Change Workflow

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

---

## C.15.3 Version Control

Configuration files should remain under version control whenever possible.

Version control provides:

- complete history;
- rollback capability;
- change traceability;
- collaborative maintenance;
- configuration comparison.

Configuration repositories should exclude all confidential information.

---

## C.15.4 Production Approval

Before deployment, configuration changes should be verified for:

- syntax validity;
- dependency consistency;
- service compatibility;
- rollback availability;
- operational impact.

Only validated configurations should be promoted to production.

---

## C.15.5 Documentation Synchronization

Configuration documentation should always reflect the production environment.

Whenever a configuration file changes, the corresponding documentation should be reviewed to ensure continued alignment between operational reality and technical documentation.

---

# C.16 Appendix Summary

This appendix defines the configuration reference for the complete LMWPool production platform.

It documents the configuration assets governing:

- infrastructure deployment;
- container orchestration;
- blockchain services;
- mining operations;
- reverse proxy services;
- Blockchain Notary;
- application runtime;
- certificates;
- configuration lifecycle.

Together with the previous volumes and appendices, this document establishes the baseline configuration model for the production platform.

Future revisions should continue to reflect the actual production environment while preserving the principles of explicit configuration, version control and operational reproducibility.

---

# End of Appendix C

**LMWPool Technical Documentation**

**Appendix C — Production Configuration Reference**

**Release:** RC1
