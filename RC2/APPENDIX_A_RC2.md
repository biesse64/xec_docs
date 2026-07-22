# RC2 Editorial Revision Notice

**Release:** RC2 (Editorial Review Candidate)

This revision preserves the production infrastructure inventory documented in RC1 while aligning the appendix with the editorial standards of the complete LMWPool documentation suite.

## RC2 objectives

- Standardize terminology across Volumes 1–4 and Appendices A–F.
- Prepare unified numbering for figures and tables.
- Prepare explicit cross-references to the corresponding volume sections.
- Improve consistency for the future Release 1.0.

---

# APPENDIX A

# Infrastructure Inventory

**Document:** LMWPool Technical Documentation  
**Appendix:** A  
**Title:** Infrastructure Inventory  
**Platform:** eCash (XEC) Production Platform  
**Status:** Production  
**Release:** RC1

---

# Document Purpose

This appendix provides the complete inventory of the production infrastructure supporting the LMWPool ecosystem.

Unlike the previous four volumes, which describe architecture, software components and operational procedures, this appendix inventories the actual production environment.

Its objectives are to:

- document every infrastructure component;
- identify production dependencies;
- support maintenance activities;
- simplify disaster recovery;
- provide a baseline for future infrastructure evolution.

Whenever possible, the information contained in this appendix is derived directly from the production environment inventory rather than from theoretical examples. Future Release 1.0 will include explicit section-level cross references to Volumes 1–4. :contentReference[oaicite:0]{index=0}

---

# Table of Contents

A.1 Executive Summary

A.2 Host Infrastructure

A.3 Operating System

A.4 Hardware Inventory

A.5 Filesystem Layout

A.6 Docker Architecture

A.7 Container Inventory

A.8 Docker Networks

A.9 Published Services

A.10 Port Inventory

A.11 Storage Inventory

A.12 Configuration Inventory

A.13 Native Services

A.14 Security Inventory

A.15 Backup Scope

A.16 Disaster Recovery Priority

A.17 Infrastructure Statistics

A.18 Appendix Summary

---

# A.1 Executive Summary

## A.1.1 Purpose

The LMWPool platform operates as a production environment hosting two independent business systems:

- the XEC Solo Mining Platform;
- the XEC Blockchain Notary.

Although both systems execute on the same Linux virtual server, they remain logically separated and communicate only where strictly required.

This architectural approach minimizes operational complexity while maximizing maintainability and disaster recovery capabilities.

The inventory presented in this appendix represents the production environment at the time of generation and should be considered the authoritative infrastructure reference. :contentReference[oaicite:1]{index=1}

---

## A.1.2 Infrastructure Overview

The production platform is composed of four major infrastructure layers.

```text
Internet
    │
    ▼
Public Services
    │
    ▼
Docker Platform
    │
    ▼
Linux Host
```

The Linux host provides the operating system, networking, storage and process management.

Docker provides application isolation.

Application services implement mining and blockchain notarization.

Public services expose only the interfaces intentionally made available to external users.

---

## A.1.3 Architectural Characteristics

The infrastructure has been designed according to the following principles.

- Production before convenience.
- One service, one responsibility.
- Minimal coupling.
- Local blockchain authority.
- Explicit configuration.
- Independent recovery.
- Complete operational ownership.

These principles are consistently reflected throughout all four documentation volumes.

---

## A.1.4 Business Systems

The current production platform supports two independent production systems.

| System | Primary Purpose |
|---------|-----------------|
| XEC Mining Platform | Solo mining infrastructure |
| XEC Blockchain Notary | Blockchain document notarization |

Although both systems share the same host, they remain operationally independent except where blockchain services are intentionally reused.

---

# A.2 Host Infrastructure

## A.2.1 Host Overview

The production environment is hosted on a single Linux virtual private server.

The inventory identifies the current production host as:

| Property | Value |
|----------|-------|
| Hostname | blockchain-pool |
| Platform Type | Virtual Machine |
| Virtualization | KVM |
| Primary User | root |

This server hosts both the mining infrastructure and the blockchain notarization platform. :contentReference[oaicite:2]{index=2}

---

## A.2.2 Infrastructure Model

The infrastructure follows a layered architecture.

```text
Linux VPS
│
├── Docker Engine
│
├── XEC Mining Platform
│
│   ├── bitcoind-xec
│   ├── ckpool-xec
│   ├── stats-api-xec
│   ├── status-page-xec
│   └── support utilities
│
└── XEC Blockchain Notary
    ├── Node.js
    ├── SQLite
    ├── REST API
    └── PDF generation
```

This organization minimizes dependencies while preserving complete operational control.

---

## A.2.3 Host Responsibilities

The Linux host provides:

- operating system;
- Docker runtime;
- filesystem management;
- networking;
- firewall;
- storage;
- process scheduling;
- service supervision.

Business logic is implemented by the application layer and remains independent from host administration.

---

# A.3 Operating System

## A.3.1 Distribution

The production platform currently executes on:

| Attribute | Value |
|----------|-------|
| Distribution | Ubuntu |
| Release | 22.04.5 LTS |
| Codename | Jammy Jellyfish |
| Architecture | x86-64 |

The operating system is maintained as a Long-Term Support (LTS) release in order to maximize platform stability. :contentReference[oaicite:3]{index=3}

---

## A.3.2 Kernel

At the time of inventory generation the production server was executing the following Linux kernel family:

- Linux 5.15.x
- x86_64 architecture

Kernel updates should always be validated against Docker and blockchain services before deployment into production. :contentReference[oaicite:4]{index=4}

---

## A.3.3 Time Configuration

The server operates using the Europe/Rome time zone.

System time synchronization is enabled through NTP, ensuring consistent timestamps for:

- blockchain operations;
- mining events;
- application logs;
- certificates;
- operational auditing. :contentReference[oaicite:5]{index=5}

---

# A.4 Hardware Inventory

## A.4.1 Processor

The production host is equipped with an AMD EPYC virtualized processor.

| Property | Value |
|----------|-------|
| Vendor | AMD |
| Processor Family | EPYC Milan |
| Architecture | x86_64 |
| Logical CPUs | 4 |

The available processing capacity is sufficient for concurrent execution of the mining platform, support services and blockchain notarization environment. :contentReference[oaicite:6]{index=6}

---

## A.4.2 Memory

System memory is shared between:

- Docker containers;
- blockchain node;
- CKPool;
- support services;
- Node.js runtime;
- operating system.

Memory utilization should be periodically monitored to ensure sufficient capacity for blockchain growth and future platform expansion.

---

## A.4.3 Hardware Philosophy

The platform intentionally favors a compact and centralized deployment.

Rather than distributing services across multiple servers, related components execute on the same host while remaining logically isolated through Docker networking and independent application boundaries.

This approach provides:

- simplified administration;
- lower operational costs;
- reduced latency between dependent services;
- straightforward backup procedures;
- deterministic disaster recovery.

---

**End of Appendix A — Part 1**

*(Continues with A.5 Filesystem Layout, A.6 Docker Architecture, A.7 Container Inventory and A.8 Docker Networks.)*
---

# A.5 Filesystem Layout

## A.5.1 Overview

The production platform follows a structured filesystem organization that separates infrastructure, mining services, blockchain notarization services and persistent operational data.

The objective is to maintain a clear distinction between:

- application code;
- configuration;
- persistent data;
- logs;
- backups;
- generated artifacts.

This organization simplifies maintenance, backup procedures and disaster recovery activities. :contentReference[oaicite:0]{index=0}

---

## A.5.2 Primary Production Directories

The production environment is organized around several primary directories.

| Directory | Purpose |
|-----------|---------|
| `/root/xec_ckpool` | XEC Solo Mining Platform |
| `/root/xec_notary` | Blockchain Notary application |
| `/root/docker-bitcoin-pool` | Shared Docker infrastructure |
| `/root/backups` | Operational backups (where applicable) |
| `/root/logs` | Operational log storage (where applicable) |

Each directory has a single operational responsibility and should remain independent whenever possible. :contentReference[oaicite:1]{index=1}

---

## A.5.3 XEC Mining Directory

The XEC mining directory contains the complete CKPool production environment.

Typical contents include:

- Docker Compose files
- CKPool configuration
- startup scripts
- persistent logs
- maintenance scripts
- documentation
- configuration backups

The directory represents the operational center of the mining platform.

---

## A.5.4 Blockchain Notary Directory

The Notary directory contains the complete blockchain notarization application.

Typical components include:

- Node.js application
- REST API
- SQLite database
- PDF generation engine
- email subsystem
- configuration files
- maintenance scripts

The Notary service operates independently from CKPool while sharing the same blockchain infrastructure.

---

## A.5.5 Shared Infrastructure Directory

The shared Docker directory contains components used by multiple production services.

Typical resources include:

- reverse proxy configuration;
- Docker Compose projects;
- shared static resources;
- TLS certificate storage;
- cached market data.

Shared infrastructure should remain independent from application-specific business logic.

---

# A.6 Docker Architecture

## A.6.1 Overview

Docker provides application isolation throughout the production platform.

Every major software component executes inside its own container with clearly defined responsibilities.

Benefits include:

- service isolation;
- simplified upgrades;
- reproducible deployments;
- controlled networking;
- simplified backup procedures.

---

## A.6.2 Architectural Layers

The production platform can be represented as four logical layers.

```text
+------------------------------------------------------+
|                 Public Internet                      |
+--------------------------+---------------------------+
                           |
                           ▼
+------------------------------------------------------+
|                 Caddy Reverse Proxy                  |
+--------------------------+---------------------------+
                           |
                           ▼
+------------------------------------------------------+
|              Docker Internal Networks                |
+--------------------------+---------------------------+
                           |
                           ▼
+------------------------------------------------------+
|          Production Service Containers               |
+------------------------------------------------------+
```

Only explicitly published services are accessible from outside the host.

All internal communications remain confined to Docker networks. :contentReference[oaicite:2]{index=2}

---

## A.6.3 Container Philosophy

Each production container implements one primary responsibility.

Examples include:

| Container | Responsibility |
|-----------|----------------|
| bitcoind-xec | Blockchain node |
| ckpool-xec | Solo mining server |
| stats-api-xec | Statistics API |
| status-page-xec | Web presentation |
| caddy-btc | Reverse proxy |

This design minimizes coupling and simplifies operational troubleshooting.

---

## A.6.4 Container Lifecycle

Production containers are configured to restart automatically after:

- host reboot;
- Docker restart;
- unexpected service termination.

Automatic restart policies reduce operational downtime while preserving deterministic recovery behavior. :contentReference[oaicite:3]{index=3}

---

# A.7 Container Inventory

## A.7.1 Production Containers

The inventory identifies the following primary production containers.

| Container | Purpose | Criticality |
|-----------|----------|-------------|
| bitcoind-xec | XEC blockchain node | Critical |
| ckpool-xec | Solo mining server | Critical |
| stats-api-xec | Statistics service | High |
| status-page-xec | Public status website | Medium |
| caddy-btc | Reverse proxy | Critical |

Additional containers supporting other platform components may coexist on the host but remain logically independent from the XEC mining platform. :contentReference[oaicite:4]{index=4}

---

## A.7.2 Critical Containers

The following containers are considered essential for mining operations.

### bitcoind-xec

Responsibilities:

- blockchain synchronization;
- block validation;
- JSON-RPC services;
- ZMQ notifications;
- template generation.

Loss of this container prevents mining operations.

---

### ckpool-xec

Responsibilities:

- Stratum protocol;
- miner communication;
- share validation;
- block submission;
- mining statistics.

Loss of this container interrupts all mining activity.

---

### caddy-btc

Responsibilities:

- HTTPS termination;
- reverse proxy;
- certificate management;
- public routing.

Loss of this container affects all public web services.

---

## A.7.3 Supporting Containers

Supporting containers provide non-critical but important services.

These include:

- statistics collection;
- web presentation;
- cached market information;
- monitoring endpoints.

Although mining may continue during certain supporting service failures, user experience and platform observability are significantly reduced.

---

# A.8 Docker Networks

## A.8.1 Purpose

Docker networking provides logical isolation between services.

Internal networks prevent direct exposure of backend services while enabling controlled communication between authorized containers.

---

## A.8.2 Network Inventory

The production inventory identifies multiple Docker networks supporting the platform. :contentReference[oaicite:5]{index=5}

Each network has a specific responsibility:

- application communication;
- reverse proxy connectivity;
- service isolation;
- infrastructure segmentation.

---

## A.8.3 Internal Communications

Typical communication paths include:

```text
Internet
      │
      ▼
Caddy Reverse Proxy
      │
      ▼
Stats API
      │
      ▼
CKPool
      │
      ▼
bitcoind-xec
```

The browser never communicates directly with blockchain services.

All access is mediated through the reverse proxy or dedicated internal APIs.

---

## A.8.4 Network Design Principles

The Docker networking model follows these principles.

- Least exposure.
- Explicit routing.
- Service isolation.
- Independent deployment.
- Predictable communication paths.
- Minimal trust boundaries.

These principles reduce the attack surface while maintaining operational flexibility.

---

**End of Appendix A — Part 2**

*(Continues with A.9 Published Services, A.10 Port Inventory, A.11 Storage Inventory and A.12 Configuration Inventory.)*
---

# A.9 Published Services

## A.9.1 Overview

Only a limited number of production services are intentionally exposed outside the hosting environment.

This design significantly reduces the external attack surface while ensuring that all publicly accessible functions remain available through controlled entry points.

All remaining services communicate exclusively over internal Docker networks or local interfaces. :contentReference[oaicite:0]{index=0}

---

## A.9.2 Public Internet Services

The production platform exposes the following logical services.

| Service | Primary Function | Public |
|----------|-----------------|---------|
| HTTPS Reverse Proxy | Secure web access | Yes |
| XEC Status Page | Public monitoring | Yes |
| Statistics API | Mining statistics | Yes (via proxy) |
| XEC Blockchain Notary | REST API | Yes (via proxy) |
| CKPool Stratum | Mining connectivity | Yes |

Direct access to backend application containers is intentionally prohibited.

---

## A.9.3 Reverse Proxy

The reverse proxy represents the primary entry point into the platform.

Its responsibilities include:

- HTTPS termination
- TLS certificate management
- request routing
- URL rewriting
- security headers
- backend isolation

Every public HTTP request is processed by the reverse proxy before reaching any backend application.

---

## A.9.4 Mining Services

Mining services are intentionally separated from traditional web services.

Published mining interfaces provide:

- Stratum protocol
- share submission
- mining notifications
- work distribution

No web browser interaction occurs directly with the mining engine.

---

## A.9.5 Internal Services

The following services remain internal.

| Component | Visibility |
|------------|------------|
| bitcoind-xec RPC | Internal |
| ZMQ interfaces | Internal |
| SQLite database | Internal |
| Docker APIs | Internal |
| Host administration | Internal |

These services must never be published directly to the Internet.

---

# A.10 Port Inventory

## A.10.1 Overview

The production environment exposes only a small number of ports.

All other ports remain accessible exclusively through internal communication paths.

This approach simplifies firewall administration while reducing unnecessary exposure. :contentReference[oaicite:1]{index=1}

---

## A.10.2 Public Ports

| Port | Protocol | Service |
|------|----------|----------|
| 80 | HTTP | Redirect / ACME |
| 443 | HTTPS | Reverse Proxy |
| 443/UDP | HTTP/3 | Reverse Proxy |
| 7333 | TCP | CKPool Standard Stratum |
| 7444 | TCP | CKPool High-Difficulty Stratum |

These ports represent the complete externally accessible interface of the XEC platform.

---

## A.10.3 Internal Ports

Several application services communicate through private Docker networking.

Typical internal ports include:

| Service | Typical Function |
|----------|------------------|
| JSON-RPC | Blockchain communication |
| ZMQ | Block notifications |
| Statistics API | Backend API |
| Node.js | Notary application |
| SQLite | Local persistence |

Internal ports are intentionally unreachable from the public Internet.

---

## A.10.4 Port Management Principles

Port allocation follows several operational rules.

- Public ports are explicitly documented.
- Internal services never publish unnecessary interfaces.
- Docker networking isolates backend communication.
- Firewall policies remain minimal and deterministic.

---

# A.11 Storage Inventory

## A.11.1 Overview

Persistent storage is divided into multiple logical categories.

These categories separate configuration from operational data, improving backup procedures and disaster recovery.

---

## A.11.2 Persistent Data Categories

| Category | Examples |
|-----------|----------|
| Configuration | Docker Compose, Caddy, CKPool |
| Blockchain | XEC blockchain data |
| Application | Node.js application |
| Database | SQLite |
| Logs | Application logs |
| Certificates | TLS certificates |
| Generated Documents | PDF certificates |
| Cached Data | Market cache |

---

## A.11.3 Bind Mount Philosophy

The production platform relies primarily on bind mounts.

Advantages include:

- transparent administration;
- simplified backup;
- direct inspection;
- deterministic recovery;
- version control of configuration files.

This approach also simplifies migration to future infrastructure.

---

## A.11.4 Log Storage

Operational logs are retained independently from application code.

Typical logging categories include:

- Docker logs;
- CKPool logs;
- Notary logs;
- system logs;
- reverse proxy logs.

Centralized logging simplifies troubleshooting and incident analysis.

---

## A.11.5 Backup Scope

The following information should always be included in routine backups.

- configuration files;
- Docker Compose definitions;
- SQLite databases;
- generated certificates;
- scripts;
- application code;
- persistent blockchain-related metadata where applicable.

Large blockchain datasets may be regenerated and therefore require a different backup strategy.

---

# A.12 Configuration Inventory

## A.12.1 Overview

The production environment is controlled through a relatively small number of configuration files.

Each configuration file has a clearly defined operational responsibility.

---

## A.12.2 Primary Configuration Files

| Configuration | Purpose |
|---------------|---------|
| Docker Compose | Container orchestration |
| CKPool configuration | Mining server |
| Caddyfile | Reverse proxy |
| xec.conf | Blockchain node |
| server.js | Notary application |
| package.json | Node dependencies |
| systemd service files | Native services |
| Shell scripts | Operational automation |

The inventory collected from the production environment identifies these files as the primary operational configuration assets. :contentReference[oaicite:2]{index=2}

---

## A.12.3 Configuration Principles

Configuration management follows several guiding principles.

- Explicit configuration.
- Version-controlled files.
- Separation of code and configuration.
- Minimal duplication.
- Human-readable formats.
- Deterministic deployments.

---

## A.12.4 Configuration Protection

Configuration files may contain operational parameters but should never expose confidential information.

The following items must always remain outside documentation and public repositories.

- passwords;
- API keys;
- SMTP credentials;
- wallet secrets;
- private keys;
- authentication tokens.

Sensitive values should instead be injected through protected operational procedures or dedicated secret management mechanisms.

---

**End of Appendix A — Part 3**

*(Continues with A.13 Native Services, A.14 Security Inventory, A.15 Backup Strategy, A.16 Disaster Recovery Priority, A.17 Infrastructure Statistics and A.18 Appendix Summary.)*
---

# A.13 Native Services

## A.13.1 Overview

In addition to the Docker-based infrastructure, the production platform relies on a limited number of native Linux services.

These services provide operating system integration, scheduled execution, process supervision and infrastructure support functions.

Native services intentionally remain lightweight and independent from application business logic. :contentReference[oaicite:0]{index=0}

---

## A.13.2 Service Categories

Native services are divided into the following operational groups.

| Category | Purpose |
|-----------|---------|
| systemd Services | Long-running processes |
| systemd Timers | Scheduled execution |
| Cron Jobs | Legacy scheduled tasks |
| Shell Scripts | Operational automation |
| Linux Services | Host administration |

This separation simplifies operational maintenance and troubleshooting.

---

## A.13.3 systemd Services

Systemd provides lifecycle management for native production applications.

Typical responsibilities include:

- automatic startup after reboot;
- automatic restart after failures;
- dependency management;
- centralized logging through the system journal;
- service monitoring.

The inventory confirms that production services are managed through systemd where appropriate. :contentReference[oaicite:1]{index=1}

---

## A.13.4 Scheduled Activities

Routine operational activities include:

- maintenance;
- cleanup;
- monitoring;
- report generation;
- notification handling;
- periodic synchronization.

Scheduling mechanisms should remain deterministic and fully documented.

---

## A.13.5 Operational Scripts

Several shell scripts automate recurring infrastructure activities.

Typical examples include:

- backup procedures;
- service restart;
- maintenance tasks;
- monitoring;
- notification handling;
- operational diagnostics.

Automation reduces operational risk while improving repeatability.

---

# A.14 Security Inventory

## A.14.1 Security Philosophy

The production platform follows a defense-in-depth approach.

Security is achieved through multiple independent protection layers rather than relying on a single control.

These layers include:

- operating system security;
- Docker isolation;
- reverse proxy filtering;
- firewall policies;
- application authentication;
- service segregation.

---

## A.14.2 Security Layers

```text
Internet
      │
Firewall
      │
Reverse Proxy
      │
Docker Network Isolation
      │
Application Authentication
      │
Business Logic
```

Each layer contributes independently to reducing operational risk.

---

## A.14.3 Public Exposure

Only explicitly documented production services are reachable from the Internet.

Backend infrastructure remains isolated through Docker networking and host firewall configuration. :contentReference[oaicite:2]{index=2}

---

## A.14.4 Authentication

Authentication mechanisms differ according to service responsibilities.

Examples include:

| Component | Authentication |
|------------|----------------|
| Linux Host | SSH |
| Docker | Local administration |
| CKPool | Stratum protocol |
| Notary API | Application-level authentication |
| Administration | Restricted access |

Authentication responsibilities remain localized within each subsystem.

---

## A.14.5 Secrets Management

Operational secrets include:

- SMTP credentials;
- RPC credentials;
- API keys;
- application secrets;
- user authentication keys;
- TLS private keys.

Secrets must never be stored within documentation or public repositories.

---

## A.14.6 Operational Logging

Security-related events should be logged whenever practical.

Typical logging categories include:

- authentication events;
- service failures;
- application exceptions;
- blockchain events;
- administrative operations.

Logging provides operational traceability and supports incident investigation.

---

# A.15 Backup Scope

## A.15.1 Purpose

Backup procedures protect configuration, operational data and application components required to restore the production platform.

Backups should prioritize information that cannot be regenerated automatically.

---

## A.15.2 Backup Categories

| Category | Backup Required |
|-----------|-----------------|
| Configuration | Yes |
| Source Code | Yes |
| SQLite Database | Yes |
| PDF Certificates | Yes |
| Scripts | Yes |
| Docker Compose | Yes |
| Blockchain Data | Strategy dependent |

The blockchain itself may be resynchronized from the network and therefore requires a different recovery strategy than application data.

---

## A.15.3 Backup Frequency

Operational policy should define backup frequency according to business criticality.

Typical recommendations include:

- daily application backup;
- frequent database backup;
- configuration backup after every infrastructure change;
- periodic verification of backup integrity.

---

## A.15.4 Backup Validation

A backup is considered valid only after successful restoration testing.

Validation procedures should verify:

- file integrity;
- database readability;
- application startup;
- service dependencies;
- operational functionality.

---

# A.16 Disaster Recovery Priority

## A.16.1 Recovery Principles

Disaster recovery should restore infrastructure following dependency order.

Attempting to restore higher-level services before foundational infrastructure increases recovery complexity and risk.

---

## A.16.2 Recovery Sequence

Recommended recovery order:

1. Linux operating system
2. Docker Engine
3. Docker networks
4. Persistent storage
5. Blockchain node
6. Reverse proxy
7. CKPool
8. Statistics services
9. Status Page
10. Blockchain Notary

Each stage depends upon the successful completion of the previous stage.

---

## A.16.3 Recovery Objectives

Recovery procedures should prioritize:

- platform availability;
- configuration integrity;
- data consistency;
- operational verification;
- controlled return to production.

---

# A.17 Infrastructure Statistics

## A.17.1 Inventory Summary

The production inventory identifies a compact but highly structured infrastructure.

Major inventory categories include:

- Linux host;
- Docker Engine;
- production containers;
- Docker networks;
- persistent storage;
- native services;
- configuration assets;
- operational scripts.

The resulting architecture is intentionally simple while supporting multiple production workloads. :contentReference[oaicite:3]{index=3}

---

## A.17.2 Architectural Characteristics

The platform demonstrates the following characteristics.

- Centralized infrastructure.
- Strong service separation.
- Containerized deployment.
- Explicit configuration.
- Limited public exposure.
- Local blockchain authority.
- Independent business services.

These characteristics contribute to long-term maintainability and operational reliability.

---

# A.18 Appendix Summary

The Infrastructure Inventory provides the definitive reference for the production environment supporting the LMWPool platform.

Unlike the architectural descriptions contained in Volumes 1 through 4, this appendix focuses on the physical and logical assets currently deployed in production.

The inventory documents:

- infrastructure components;
- operating system;
- Docker architecture;
- production containers;
- networking;
- storage;
- configuration assets;
- native services;
- security controls;
- backup scope;
- disaster recovery priorities.

Future revisions of this appendix should be generated from updated production inventory data in order to ensure continued alignment between documentation and the operational environment. :contentReference[oaicite:4]{index=4}

---

# End of Appendix A

**LMWPool Technical Documentation**

**Appendix A — Infrastructure Inventory**

**Release:** RC1