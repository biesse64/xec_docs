# RC2 Editorial Revision Notice

**Release:** RC2 (Editorial Review Candidate)

This revision preserves the validated operational procedures documented in RC1 while aligning the appendix with the editorial standards adopted across the complete LMWPool documentation suite.

## RC2 objectives

- Standardize terminology across Volumes 1–4 and Appendices A–F.
- Prepare unified numbering for runbooks, figures and tables.
- Prepare explicit cross-references to operational commands and configuration references.
- Improve publication consistency for Release 1.0.

---

# APPENDIX E

# Operational Runbooks

**Document:** LMWPool Technical Documentation  
**Appendix:** E  
**Title:** Operational Runbooks  
**Platform:** LMWPool XEC Production Platform  
**Release:** RC1

---

# Document Purpose

This appendix documents the standard operational procedures required to operate, maintain and recover the LMWPool production platform.

Unlike previous appendices, which describe infrastructure, configuration and APIs, this appendix focuses on operational execution.

Every runbook is intended to provide a deterministic sequence of actions that minimizes operational risk while ensuring repeatable outcomes. Future Release 1.0 will include explicit section-level references to Appendix B (Command Reference), Appendix C (Configuration Reference) and Volume 4 (Infrastructure Operations).

---

# Table of Contents

E.1 Operational Philosophy

E.2 Daily Operations

E.3 Startup Procedures

E.4 Shutdown Procedures

E.5 Planned Maintenance

E.6 Incident Response

E.7 CKPool Recovery

E.8 bitcoind-xec Recovery

E.9 Blockchain Notary Recovery

E.10 Disaster Recovery

E.11 Health Verification

E.12 Operational Checklist

E.13 Appendix Summary

---

# E.1 Operational Philosophy

## E.1.1 Purpose

Operational procedures must be:

- deterministic;
- repeatable;
- documented;
- auditable;
- reversible whenever practical.

Every production activity should follow documented procedures rather than relying on operator memory.

---

## E.1.2 General Principles

The platform follows several operational principles.

- Production before convenience.
- Minimize downtime.
- Verify before modifying.
- Backup before major changes.
- Validate after every intervention.
- Document every significant change.

---

## E.1.3 Operational Priorities

The order of operational priorities is:

1. Safety
2. Data integrity
3. Service availability
4. Performance
5. Optimization

No optimization activity should compromise production stability.

---

# E.2 Daily Operations

## E.2.1 Daily Health Check

Daily verification should include:

- Docker status
- CKPool availability
- Blockchain synchronization
- Notary service status
- Statistics API
- Reverse Proxy
- Disk utilization
- Memory utilization
- Container restart events

---

## E.2.2 Mining Verification

Verify:

- connected miners;
- accepted shares;
- hashrate stability;
- absence of excessive rejected shares;
- block synchronization.

---

## E.2.3 Notary Verification

Verify:

- REST API availability;
- database accessibility;
- PDF generation;
- email delivery;
- blockchain connectivity.

---

# E.3 Startup Procedures

## E.3.1 Host Startup

Recommended startup sequence.

1. Verify Linux host.
2. Verify storage.
3. Verify Docker Engine.
4. Verify Docker networks.
5. Start production containers.
6. Verify blockchain synchronization.
7. Verify CKPool.
8. Verify public services.
9. Verify Blockchain Notary.
10. Execute health checks.

---

## E.3.2 Container Startup

Containers should be started according to dependency order.

```text
bitcoind-xec
        │
        ▼
ckpool-xec
        │
        ▼
Statistics API
        │
        ▼
Status Page
        │
        ▼
Blockchain Notary
```

---

## E.3.3 Startup Validation

Startup is considered complete only after:

- all required containers are running;
- blockchain synchronization is verified;
- APIs respond correctly;
- public services are accessible;
- no critical errors appear in logs.

---

# E.4 Shutdown Procedures

## E.4.1 Planned Shutdown

Recommended sequence.

1. Notify users (if applicable).
2. Stop public services.
3. Stop Statistics API.
4. Stop CKPool.
5. Stop Blockchain Notary.
6. Verify pending operations.
7. Stop blockchain services if required.

---

## E.4.2 Emergency Shutdown

Emergency shutdown should only occur when required to:

- protect data;
- prevent corruption;
- contain security incidents;
- avoid hardware damage.

Every emergency shutdown should be documented after completion.

---

# E.5 Planned Maintenance

## E.5.1 Preparation

Before maintenance:

- verify backups;
- review configuration changes;
- identify rollback procedure;
- estimate downtime.

---

## E.5.2 Execution

Maintenance should follow:

1. Backup.
2. Validation.
3. Deployment.
4. Verification.
5. Documentation update.

---

## E.5.3 Completion

Maintenance concludes only after:

- service verification;
- log review;
- functional testing;
- confirmation of normal production operation.

---

# E.6 Incident Response

## E.6.1 Objectives

Incident response aims to:

- restore service;
- preserve integrity;
- identify root cause;
- prevent recurrence.

---

## E.6.2 Incident Categories

| Severity | Description |
|----------|-------------|
| Critical | Production unavailable |
| High | Major degradation |
| Medium | Partial degradation |
| Low | Minor operational issue |

---

## E.6.3 Initial Response

Every incident should begin with:

- service assessment;
- log collection;
- impact evaluation;
- containment if necessary.

Root cause analysis should follow service stabilization.

---

# E.7 CKPool Recovery

## E.7.1 Symptoms

Typical symptoms include:

- miners disconnected;
- no accepted shares;
- repeated restart loops;
- missing block templates.

---

## E.7.2 Recovery Procedure

Recommended sequence.

1. Verify container status.
2. Inspect logs.
3. Verify blockchain node.
4. Verify RPC connectivity.
5. Restart CKPool.
6. Verify miners reconnect.
7. Confirm accepted shares.

---

## E.7.3 Recovery Validation

Recovery is complete when:

- miners reconnect;
- accepted shares increase;
- logs report normal activity.

---

# E.8 bitcoind-xec Recovery

## E.8.1 Failure Indicators

Examples:

- blockchain not synchronized;
- RPC unavailable;
- peer count equals zero;
- node not responding.

---

## E.8.2 Recovery

1. Verify process.
2. Inspect logs.
3. Verify disk space.
4. Verify network connectivity.
5. Restart service.
6. Confirm synchronization.

Mining should remain suspended until synchronization completes.

---

# E.9 Blockchain Notary Recovery

## E.9.1 Typical Problems

- API unavailable.
- Email failures.
- SQLite unavailable.
- PDF generation failures.
- Blockchain connectivity problems.

---

## E.9.2 Recovery Procedure

1. Verify application service.
2. Verify SQLite.
3. Verify blockchain connectivity.
4. Verify SMTP.
5. Verify API.
6. Verify certificate generation.

---

## E.9.3 Functional Verification

Recovery concludes after successful execution of a complete notarization workflow.

---

# E.10 Disaster Recovery

## E.10.1 Recovery Order

Recommended sequence.

1. Linux host.
2. Docker.
3. Storage.
4. Networks.
5. bitcoind-xec.
6. Reverse Proxy.
7. CKPool.
8. Statistics API.
9. Status Page.
10. Blockchain Notary.

---

## E.10.2 Validation

Every recovery should verify:

- service availability;
- blockchain synchronization;
- miner connectivity;
- API functionality;
- certificate generation.

---

# E.11 Health Verification

Routine health verification should confirm:

- all containers running;
- no restart loops;
- synchronized blockchain;
- operational APIs;
- acceptable resource utilization;
- no critical log entries.

---

# E.12 Operational Checklist

Daily checklist.

- □ Linux host operational
- □ Docker operational
- □ bitcoind synchronized
- □ CKPool operational
- □ Statistics API operational
- □ Status Page operational
- □ Blockchain Notary operational
- □ TLS certificates valid
- □ Disk space acceptable
- □ Backup completed
- □ No critical alerts

---

# E.13 Appendix Summary

This appendix defines the operational procedures supporting the LMWPool production platform.

The documented runbooks provide standardized guidance for:

- routine operations;
- service startup;
- controlled shutdown;
- maintenance;
- incident response;
- service recovery;
- disaster recovery;
- operational verification.

These procedures establish a repeatable operational framework that complements the architectural, configuration and API documentation contained in the previous volumes and appendices.

---

# End of Appendix E

**LMWPool Technical Documentation**

**Appendix E — Operational Runbooks**

**Release:** RC1