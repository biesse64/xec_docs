# RC2 Editorial Revision Notice

**Release:** RC2 (Editorial Review Candidate)

This revision preserves the validated production command reference documented in RC1 while aligning the appendix with the editorial standards adopted throughout the LMWPool documentation suite.

## RC2 objectives

- Harmonize terminology with Volumes 1–4.
- Prepare unified numbering for command listings, figures and tables.
- Introduce explicit cross-reference placeholders to the related operational sections.
- Prepare the appendix for Release 1.0 publication.

---

# APPENDIX B

# Command Reference

**Document:** LMWPool Technical Documentation  
**Appendix:** B  
**Title:** Production Command Reference  
**Platform:** LMWPool XEC Production Platform  
**Release:** RC1

---

# Document Purpose

This appendix provides the operational command reference for the complete LMWPool production environment.

Unlike the previous documentation volumes, this appendix is intended as a practical operational handbook.

Every command documented here has one or more of the following objectives:

- infrastructure administration;
- service monitoring;
- operational troubleshooting;
- maintenance;
- disaster recovery;
- diagnostics.

Where possible, commands correspond to procedures validated during production deployment and maintenance of the platform. Future Release 1.0 will introduce section-level cross references to the Operational Runbooks (Appendix E) and Infrastructure Inventory (Appendix A).

---

# Table of Contents

B.1 Command Philosophy

B.2 Linux Administration

B.3 Docker Administration

B.4 Docker Compose

B.5 Container Inspection

B.6 CKPool Operations

B.7 bitcoind-xec Operations

B.8 Statistics Services

B.9 Blockchain Notary

B.10 Network Diagnostics

B.11 Log Analysis

B.12 Backup Commands

B.13 Disaster Recovery

B.14 Maintenance Commands

B.15 Troubleshooting

B.16 Appendix Summary

---

# B.1 Command Philosophy

## B.1.1 Purpose

This appendix documents the commands routinely used to operate and maintain the production environment.

Commands are grouped according to operational responsibility rather than software component.

This organization allows administrators to locate procedures rapidly during maintenance activities.

---

## B.1.2 Command Categories

Commands are divided into six major categories.

| Category | Purpose |
|----------|---------|
| Linux | Host administration |
| Docker | Container management |
| Blockchain | XEC node |
| Mining | CKPool |
| Application | Notary platform |
| Diagnostics | Monitoring and troubleshooting |

---

## B.1.3 Operational Principles

Whenever possible commands should:

- be deterministic;
- avoid unnecessary downtime;
- preserve production data;
- produce readable output;
- support troubleshooting.

Production systems should never be modified without understanding command implications.

---

# B.2 Linux Administration

## B.2.1 System Information

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

## B.2.2 Memory

```bash
free -h
```

---

## B.2.3 Storage

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

## B.2.4 Running Processes

```bash
ps aux
```

---

Monitor processes.

```bash
top
```

or

```bash
htop
```

(if installed)

---

## B.2.5 Network

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

# B.3 Docker Administration

## B.3.1 Docker Status

```bash
docker version
```

---

```bash
docker info
```

---

## B.3.2 Running Containers

```bash
docker ps
```

---

All containers.

```bash
docker ps -a
```

---

## B.3.3 Docker Images

```bash
docker image ls
```

---

## B.3.4 Docker Networks

```bash
docker network ls
```

---

Inspect network.

```bash
docker network inspect NETWORK_NAME
```

---

## B.3.5 Docker Volumes

```bash
docker volume ls
```

---

Inspect volume.

```bash
docker volume inspect VOLUME_NAME
```

---

# B.4 Docker Compose

## B.4.1 Running Projects

```bash
docker compose ls
```

---

## B.4.2 Start Project

```bash
docker compose up -d
```

---

## B.4.3 Stop Project

```bash
docker compose down
```

---

## B.4.4 Restart

```bash
docker compose restart
```

---

## B.4.5 Configuration Validation

```bash
docker compose config
```

---

## B.4.6 Logs

```bash
docker compose logs
```

---

## B.4.7 Follow Logs

```bash
docker compose logs -f
```

---

**End of Appendix B — Part 1**

*(Continues with CKPool, bitcoind-xec, Notary, diagnostics, backups, disaster recovery and troubleshooting.)*
---

# B.5 Container Inspection

## B.5.1 Container Overview

Display all running containers.

```bash
docker ps
```

Display all containers, including stopped instances.

```bash
docker ps -a
```

---

## B.5.2 Detailed Inspection

Inspect a specific container.

```bash
docker inspect CONTAINER_NAME
```

Example:

```bash
docker inspect ckpool-xec
```

Useful information includes:

- mounted volumes;
- published ports;
- Docker networks;
- restart policy;
- environment variables;
- image version;
- container status.

---

## B.5.3 Container Resource Utilization

Display real-time CPU and memory usage.

```bash
docker stats
```

Single container.

```bash
docker stats ckpool-xec
```

---

## B.5.4 Execute Commands

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

## B.5.5 Restart Container

```bash
docker restart CONTAINER
```

Example:

```bash
docker restart ckpool-xec
```

---

## B.5.6 Container Logs

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

# B.6 CKPool Operations

## B.6.1 Verify CKPool Status

```bash
docker ps | grep ckpool
```

---

## B.6.2 View CKPool Logs

```bash
docker logs ckpool-xec
```

or

```bash
docker compose logs ckpool-xec
```

---

## B.6.3 Monitor Live Activity

```bash
docker logs -f ckpool-xec
```

---

## B.6.4 Restart CKPool

```bash
docker restart ckpool-xec
```

or

```bash
docker compose restart ckpool-xec
```

---

## B.6.5 Verify Stratum Ports

```bash
ss -lnt | grep 7333
```

```bash
ss -lnt | grep 7444
```

---

## B.6.6 Search Accepted Shares

```bash
grep "WORKER_SHARE_ACCEPTED" logs/ckpool.log
```

---

## B.6.7 Highest Difficulty Shares

```bash
grep "WORKER_SHARE_ACCEPTED" logs/ckpool.log \
| awk -F'diff=' '{print $2}' \
| awk -F'/' '{print $1}' \
| sort -nr \
| head -20
```

This command identifies the highest accepted share difficulties recorded by CKPool.

---

## B.6.8 Search Solved Blocks

```bash
grep "BLOCK FOUND" logs/ckpool.log
```

or

```bash
grep "Solved" logs/ckpool.log
```

depending on the current logging format.

---

# B.7 bitcoind-xec Operations

## B.7.1 Blockchain Status

```bash
bitcoin-cli getblockchaininfo
```

---

## B.7.2 Network Information

```bash
bitcoin-cli getnetworkinfo
```

---

## B.7.3 Wallet Information

```bash
bitcoin-cli getwalletinfo
```

---

## B.7.4 Peer Connections

```bash
bitcoin-cli getpeerinfo
```

---

## B.7.5 Current Block Height

```bash
bitcoin-cli getblockcount
```

---

## B.7.6 Mempool Information

```bash
bitcoin-cli getmempoolinfo
```

---

## B.7.7 Verify Synchronization

```bash
bitcoin-cli getblockchaininfo
```

Check:

- blocks;
- headers;
- verification progress;
- initialblockdownload.

Mining should not be considered operational until synchronization is complete.

---

# B.8 Statistics Services

## B.8.1 Verify Service

```bash
docker ps | grep stats
```

---

## B.8.2 Logs

```bash
docker logs stats-api-xec
```

---

## B.8.3 Restart

```bash
docker restart stats-api-xec
```

---

## B.8.4 API Test

```bash
curl http://localhost:8080/
```

or the appropriate endpoint configured in production.

---

# B.9 Blockchain Notary

## B.9.1 Verify Service

```bash
systemctl status xec-notary
```

or

```bash
docker ps
```

depending on deployment.

---

## B.9.2 Restart Service

```bash
systemctl restart xec-notary
```

---

## B.9.3 View Logs

```bash
journalctl -u xec-notary -f
```

---

## B.9.4 Verify API

```bash
curl http://localhost:8088/
```

or

```bash
curl https://xec.lmwpool.com/notary/
```

depending on the endpoint being tested.

---

**End of Appendix B — Part 2**

*(Continues with Network Diagnostics, Log Analysis, Backup Commands, Disaster Recovery, Maintenance Commands and Troubleshooting.)*
---

# B.10 Network Diagnostics

## B.10.1 Verify Network Interfaces

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

## B.10.2 Connectivity Tests

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

## B.10.3 Listening Services

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

## B.10.4 Docker Networking

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

# B.11 Log Analysis

## B.11.1 Linux Logs

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

## B.11.2 Docker Logs

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

## B.11.3 CKPool Log Analysis

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

## B.11.4 Blockchain Node Logs

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

## B.11.5 Notary Logs

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

# B.12 Backup Commands

## B.12.1 Archive Configuration

Create a compressed archive.

```bash
tar -czf backup.tar.gz DIRECTORY
```

---

## B.12.2 Backup Docker Compose

```bash
cp docker-compose.yml docker-compose.yml.bak
```

---

## B.12.3 Backup SQLite

```bash
sqlite3 database.sqlite ".backup backup.sqlite"
```

---

## B.12.4 Backup Application

```bash
tar -czf xec_notary_backup.tar.gz xec_notary/
```

---

## B.12.5 Verify Archive

```bash
tar -tf backup.tar.gz
```

---

# B.13 Disaster Recovery

## B.13.1 Verify Docker

```bash
docker ps -a
```

---

## B.13.2 Restart Docker

```bash
systemctl restart docker
```

---

## B.13.3 Restart Production Stack

```bash
docker compose up -d
```

---

## B.13.4 Verify Containers

```bash
docker ps
```

---

## B.13.5 Verify Blockchain

```bash
bitcoin-cli getblockchaininfo
```

---

## B.13.6 Verify Mining

```bash
docker logs ckpool-xec
```

---

## B.13.7 Verify Public Services

```bash
curl https://xec.lmwpool.com
```

---

# B.14 Maintenance Commands

## B.14.1 Update Docker Images

```bash
docker compose pull
```

---

## B.14.2 Restart Updated Services

```bash
docker compose up -d
```

---

## B.14.3 Remove Unused Images

```bash
docker image prune
```

---

## B.14.4 Remove Unused Volumes

```bash
docker volume prune
```

---

## B.14.5 Remove Unused Networks

```bash
docker network prune
```

---

## B.14.6 Remove All Unused Resources

```bash
docker system prune
```

> **Warning:** Verify the impact before executing this command in a production environment.

---

# B.15 Troubleshooting

## B.15.1 Container Not Running

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

## B.15.2 Blockchain Not Synchronizing

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

## B.15.3 CKPool Not Accepting Connections

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

## B.15.4 Reverse Proxy Issues

Verify Caddy.

```bash
docker logs caddy-btc
```

Inspect configuration.

```bash
docker exec caddy-btc caddy validate --config /etc/caddy/Caddyfile
```

---

## B.15.5 General Diagnostics

Useful overview.

```bash
docker ps
docker stats
free -h
df -h
uptime
```

This command sequence provides a rapid assessment of the overall health of the production platform.

---

# B.16 Appendix Summary

This appendix provides a consolidated operational command reference for the LMWPool production platform.

Commands are organized by operational responsibility rather than by application, allowing administrators to rapidly identify the procedures required for routine maintenance, diagnostics, incident response and disaster recovery.

Future revisions of this appendix should incorporate newly validated production procedures while removing deprecated operational commands. Commands included in this appendix should always be tested in a non-production environment before being introduced into operational runbooks.

---

# End of Appendix B

**LMWPool Technical Documentation**

**Appendix B — Production Command Reference**

**Release:** RC1