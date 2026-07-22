# LMWPool XEC Platform
## Appendix B — Production Command Reference

| Document metadata | Value |
|---|---|
| Document | LMWPool Technical Documentation |
| Release | 1.0 |
| Platform | LMWPool XEC Production Platform |
| Environment | Production |
| Status | Definitive release |

> **Document scope.** This appendix is the consolidated command reference for administering, monitoring, maintaining, diagnosing and recovering the deployed LMWPool production environment. It complements the architectural descriptions in Volumes 1–4 and the operational material in Appendices A and E.

---

## Document Control

> **Release note:** This document is Appendix B of the LMWPool Technical Documentation Release 1.0 set. Read it together with Volumes 1–4 and Appendices A–F for the complete platform reference.

| Field | Value |
|---|---|
| Release | 1.0 |
| Source baseline | Appendix B RC2 |
| Intended audience | Platform owner, system administrators, technical maintainers |
| Document role | Production command reference |
| Operational authority | Deployed production platform and validated operational procedures |

## Document Purpose

This appendix provides the operational command reference for the complete LMWPool production environment.

It is designed as a practical handbook for production operations rather than an architectural narrative.

Each command supports one or more of the following objectives:

- Infrastructure administration
- Service monitoring
- Operational troubleshooting
- Maintenance
- Disaster recovery
- Diagnostics

Where indicated by the source baseline, commands correspond to procedures validated during production deployment and maintenance. For infrastructure context, see Appendix A, *Infrastructure Inventory*. For procedure-level guidance, see Appendix E, *Operational Runbooks*.

---

## Table of Contents

1. [B.1 Command Philosophy](#b1-command-philosophy)
2. [B.2 Linux Administration](#b2-linux-administration)
3. [B.3 Docker Administration](#b3-docker-administration)
4. [B.4 Docker Compose](#b4-docker-compose)
5. [B.5 Container Inspection](#b5-container-inspection)
6. [B.6 CKPool Operations](#b6-ckpool-operations)
7. [B.7 `bitcoind-xec` Operations](#b7-bitcoind-xec-operations)
8. [B.8 Statistics Services](#b8-statistics-services)
9. [B.9 Blockchain Notary](#b9-blockchain-notary)
10. [B.10 Network Diagnostics](#b10-network-diagnostics)
11. [B.11 Log Analysis](#b11-log-analysis)
12. [B.12 Backup Commands](#b12-backup-commands)
13. [B.13 Disaster Recovery](#b13-disaster-recovery)
14. [B.14 Maintenance Commands](#b14-maintenance-commands)
15. [B.15 Troubleshooting](#b15-troubleshooting)
16. [B.16 Appendix Summary](#b16-appendix-summary)

---

## B.1 Command Philosophy

### B.1.1 Purpose

This appendix documents the commands routinely used to operate and maintain the production environment.

Commands are grouped by operational responsibility rather than by software component, allowing administrators to locate the required procedure quickly during maintenance.

---

### B.1.2 Command Categories

Commands are divided into six categories.

| Category | Purpose |
|----------|---------|
| Linux | Host administration |
| Docker | Container management |
| Blockchain | XEC node |
| Mining | CKPool |
| Application | Notary platform |
| Diagnostics | Monitoring and troubleshooting |

---

### B.1.3 Operational Principles

Whenever possible, commands should:

- Be deterministic
- Avoid unnecessary downtime
- Preserve production data
- Produce readable output
- Support troubleshooting

Do not modify production systems without first understanding the implications of the command.

---

## B.2 Linux Administration

### B.2.1 System Information

Display operating system information.

```bash
uname -a
```

---

Display Linux distribution.

```bash
cat /etc/os-release
```

---

Display hostname.

```bash
hostnamectl
```

---

Display system uptime.

```bash
uptime
```

---

### B.2.2 Memory

```bash
free -h
```

---

### B.2.3 Storage

```bash
df -h
```

---

List mounted filesystems.

```bash
mount
```

---

Display block devices.

```bash
lsblk
```

---

### B.2.4 Running Processes

```bash
ps aux
```

---

Monitor processes interactively.

```bash
top
```

Alternatively:

```bash
htop
```

Use `htop` only if it is installed.

---

### B.2.5 Network

Show IP addresses.

```bash
ip addr
```

---

Show routing table.

```bash
ip route
```

---

Show listening ports.

```bash
ss -lntup
```

---

## B.3 Docker Administration

### B.3.1 Docker Status

```bash
docker version
```

---

```bash
docker info
```

---

### B.3.2 Running Containers

```bash
docker ps
```

---

Display all containers, including stopped instances.

```bash
docker ps -a
```

---

### B.3.3 Docker Images

```bash
docker image ls
```

---

### B.3.4 Docker Networks

```bash
docker network ls
```

---

Inspect a Docker network.

```bash
docker network inspect NETWORK_NAME
```

---

### B.3.5 Docker Volumes

```bash
docker volume ls
```

---

Inspect a Docker volume.

```bash
docker volume inspect VOLUME_NAME
```

---

## B.4 Docker Compose

### B.4.1 Running Projects

```bash
docker compose ls
```

---

### B.4.2 Start Project

```bash
docker compose up -d
```

---

### B.4.3 Stop Project

```bash
docker compose down
```

---

### B.4.4 Restart

```bash
docker compose restart
```

---

### B.4.5 Configuration Validation

```bash
docker compose config
```

---

### B.4.6 Logs

```bash
docker compose logs
```

---

### B.4.7 Follow Logs

```bash
docker compose logs -f
```

---


## B.5 Container Inspection

### B.5.1 Container Overview

Display all running containers.

```bash
docker ps
```

Display all containers, including stopped instances.

```bash
docker ps -a
```

---

### B.5.2 Detailed Inspection

Inspect a specific container.

```bash
docker inspect CONTAINER_NAME
```

Example:

```bash
docker inspect ckpool-xec
```

The inspection output includes useful information such as:

- Mounted volumes
- Published ports
- Docker networks
- Restart policy
- Environment variables
- Image version
- Container status

---

### B.5.3 Container Resource Utilization

Display real-time CPU and memory usage.

```bash
docker stats
```

Display resource utilization for a single container.

```bash
docker stats ckpool-xec
```

---

### B.5.4 Execute Commands

Open a shell inside a container.

```bash
docker exec -it CONTAINER /bin/bash
```

Example:

```bash
docker exec -it bitcoind-xec /bin/bash
```

If Bash is unavailable:

```bash
docker exec -it CONTAINER sh
```

---

### B.5.5 Restart Container

```bash
docker restart CONTAINER
```

Example:

```bash
docker restart ckpool-xec
```

---

### B.5.6 Container Logs

View logs.

```bash
docker logs CONTAINER
```

Follow logs.

```bash
docker logs -f CONTAINER
```

Show the last 100 lines.

```bash
docker logs --tail 100 CONTAINER
```

---

## B.6 CKPool Operations

### B.6.1 Verify CKPool Status

```bash
docker ps | grep ckpool
```

---

### B.6.2 View CKPool Logs

```bash
docker logs ckpool-xec
```

Alternatively:

```bash
docker compose logs ckpool-xec
```

---

### B.6.3 Monitor Live Activity

```bash
docker logs -f ckpool-xec
```

---

### B.6.4 Restart CKPool

```bash
docker restart ckpool-xec
```

Alternatively:

```bash
docker compose restart ckpool-xec
```

---

### B.6.5 Verify Stratum Ports

```bash
ss -lnt | grep 7333
```

```bash
ss -lnt | grep 7444
```

---

### B.6.6 Search Accepted Shares

```bash
grep "WORKER_SHARE_ACCEPTED" logs/ckpool.log
```

---

### B.6.7 Highest Difficulty Shares

```bash
grep "WORKER_SHARE_ACCEPTED" logs/ckpool.log \
| awk -F'diff=' '{print $2}' \
| awk -F'/' '{print $1}' \
| sort -nr \
| head -20
```

This command identifies the highest accepted share difficulties recorded by CKPool.

---

### B.6.8 Search Solved Blocks

```bash
grep "BLOCK FOUND" logs/ckpool.log
```

Alternatively:

```bash
grep "Solved" logs/ckpool.log
```

Use the search expression that corresponds to the current logging format.

---

## B.7 `bitcoind-xec` Operations

### B.7.1 Blockchain Status

```bash
bitcoin-cli getblockchaininfo
```

---

### B.7.2 Network Information

```bash
bitcoin-cli getnetworkinfo
```

---

### B.7.3 Wallet Information

```bash
bitcoin-cli getwalletinfo
```

---

### B.7.4 Peer Connections

```bash
bitcoin-cli getpeerinfo
```

---

### B.7.5 Current Block Height

```bash
bitcoin-cli getblockcount
```

---

### B.7.6 Mempool Information

```bash
bitcoin-cli getmempoolinfo
```

---

### B.7.7 Verify Synchronization

```bash
bitcoin-cli getblockchaininfo
```

Check the following fields:

- Blocks
- Headers
- Verification progress
- `initialblockdownload`

Mining should not be considered operational until synchronization is complete.

---

## B.8 Statistics Services

### B.8.1 Verify Service

```bash
docker ps | grep stats
```

---

### B.8.2 Logs

```bash
docker logs stats-api-xec
```

---

### B.8.3 Restart

```bash
docker restart stats-api-xec
```

---

### B.8.4 API Test

```bash
curl http://localhost:8080/
```

Alternatively, use the appropriate endpoint configured in production.

---

## B.9 Blockchain Notary

### B.9.1 Verify Service

```bash
systemctl status xec-notary
```

Alternatively:

```bash
docker ps
```

Use the command that corresponds to the documented deployment.

---

### B.9.2 Restart Service

```bash
systemctl restart xec-notary
```

---

### B.9.3 View Logs

```bash
journalctl -u xec-notary -f
```

---

### B.9.4 Verify API

```bash
curl http://localhost:8088/
```

Alternatively:

```bash
curl https://xec.lmwpool.com/notary/
```

Use the command that corresponds to the endpoint being tested.

---


## B.10 Network Diagnostics

### B.10.1 Verify Network Interfaces

Display all configured network interfaces.

```bash
ip addr
```

---

Display routing information.

```bash
ip route
```

---

Verify DNS configuration.

```bash
cat /etc/resolv.conf
```

---

### B.10.2 Connectivity Tests

Verify Internet connectivity.

```bash
ping 8.8.8.8
```

---

Verify DNS resolution.

```bash
ping google.com
```

---

Test HTTPS connectivity.

```bash
curl https://xec.lmwpool.com
```

---

Verify blockchain peer connectivity.

```bash
bitcoin-cli getpeerinfo
```

---

### B.10.3 Listening Services

Display all listening sockets.

```bash
ss -lntup
```

---

Verify HTTPS.

```bash
ss -lnt | grep 443
```

---

Verify CKPool.

```bash
ss -lnt | grep 7333
```

```bash
ss -lnt | grep 7444
```

---

### B.10.4 Docker Networking

List Docker networks.

```bash
docker network ls
```

---

Inspect a network.

```bash
docker network inspect NETWORK_NAME
```

---

Display container network configuration.

```bash
docker inspect CONTAINER \
    --format '{{json .NetworkSettings.Networks}}'
```

---

## B.11 Log Analysis

### B.11.1 Linux Logs

Display recent system log entries.

```bash
journalctl
```

---

Follow the system journal.

```bash
journalctl -f
```

---

View logs for a specific service.

```bash
journalctl -u SERVICE_NAME
```

---

### B.11.2 Docker Logs

View logs.

```bash
docker logs CONTAINER
```

---

Follow logs.

```bash
docker logs -f CONTAINER
```

---

Display the last 200 log entries.

```bash
docker logs --tail 200 CONTAINER
```

---

### B.11.3 CKPool Log Analysis

Display accepted shares.

```bash
grep "WORKER_SHARE_ACCEPTED" logs/ckpool.log
```

---

Display rejected shares.

```bash
grep "REJECT" logs/ckpool.log
```

---

Search for solved blocks.

```bash
grep -Ei "block|solved|found" logs/ckpool.log
```

---

Search for errors.

```bash
grep -i error logs/ckpool.log
```

---

### B.11.4 Blockchain Node Logs

View blockchain logs.

```bash
docker logs bitcoind-xec
```

---

Search synchronization messages.

```bash
docker logs bitcoind-xec | grep -i sync
```

---

Search RPC errors.

```bash
docker logs bitcoind-xec | grep -i rpc
```

---

### B.11.5 Notary Logs

Follow application logs.

```bash
journalctl -u xec-notary -f
```

---

Search recent errors.

```bash
journalctl -u xec-notary | grep -i error
```

---

## B.12 Backup Commands

### B.12.1 Archive Configuration

Create a compressed archive.

```bash
tar -czf backup.tar.gz DIRECTORY
```

---

### B.12.2 Backup Docker Compose

```bash
cp docker-compose.yml docker-compose.yml.bak
```

---

### B.12.3 Backup SQLite

```bash
sqlite3 database.sqlite ".backup backup.sqlite"
```

---

### B.12.4 Backup Application

```bash
tar -czf xec_notary_backup.tar.gz xec_notary/
```

---

### B.12.5 Verify Archive

```bash
tar -tf backup.tar.gz
```

---

## B.13 Disaster Recovery

### B.13.1 Verify Docker

```bash
docker ps -a
```

---

### B.13.2 Restart Docker

```bash
systemctl restart docker
```

---

### B.13.3 Restart Production Stack

```bash
docker compose up -d
```

---

### B.13.4 Verify Containers

```bash
docker ps
```

---

### B.13.5 Verify Blockchain

```bash
bitcoin-cli getblockchaininfo
```

---

### B.13.6 Verify Mining

```bash
docker logs ckpool-xec
```

---

### B.13.7 Verify Public Services

```bash
curl https://xec.lmwpool.com
```

---

## B.14 Maintenance Commands

### B.14.1 Update Docker Images

```bash
docker compose pull
```

---

### B.14.2 Restart Updated Services

```bash
docker compose up -d
```

---

### B.14.3 Remove Unused Images

```bash
docker image prune
```

---

### B.14.4 Remove Unused Volumes

```bash
docker volume prune
```

---

### B.14.5 Remove Unused Networks

```bash
docker network prune
```

---

### B.14.6 Remove All Unused Resources

```bash
docker system prune
```

> **Warning:** Verify the impact before executing this command in a production environment.

---

## B.15 Troubleshooting

### B.15.1 Container Not Running

Verify status.

```bash
docker ps -a
```

Inspect logs.

```bash
docker logs CONTAINER
```

Restart.

```bash
docker restart CONTAINER
```

---

### B.15.2 Blockchain Not Synchronizing

Verify blockchain status.

```bash
bitcoin-cli getblockchaininfo
```

Verify peers.

```bash
bitcoin-cli getpeerinfo
```

Check logs.

```bash
docker logs bitcoind-xec
```

---

### B.15.3 CKPool Not Accepting Connections

Verify ports.

```bash
ss -lnt
```

Verify container.

```bash
docker ps
```

Inspect logs.

```bash
docker logs ckpool-xec
```

---

### B.15.4 Reverse Proxy Issues

Verify Caddy.

```bash
docker logs caddy-btc
```

Inspect configuration.

```bash
docker exec caddy-btc caddy validate --config /etc/caddy/Caddyfile
```

---

### B.15.5 General Diagnostics

Run the following commands for a rapid system overview.

```bash
docker ps
docker stats
free -h
df -h
uptime
```

This command sequence provides a rapid assessment of the overall health of the production platform.

---

## B.16 Appendix Summary

This appendix provides a consolidated operational command reference for the LMWPool production platform.

Commands are organized by operational responsibility rather than by application, allowing administrators to identify the procedures required for routine maintenance, diagnostics, incident response and disaster recovery quickly.

Future revisions should incorporate newly validated production procedures and remove deprecated operational commands. Test commands in a non-production environment before introducing them into operational runbooks.

---

*End of Appendix B — Production Command Reference, Release 1.0*
