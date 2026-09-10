# Evaluating PostgreSQL Authentication Hardening Against Known-Credential Exploitation in a Containerized Stack

## 1. Project Overview

This project evaluates the effectiveness of PostgreSQL authentication hardening against remote access using a previously known credential in an authorized, local/containerized laboratory environment.

The primary test environment is the Week 1 Docker PostgreSQL stack using PostgreSQL 15 (`postgres:15-alpine`). The experiment examines the combined effect of:

- Credential rotation;
- Continued use of `SCRAM-SHA-256` password authentication;
- Restrictive `pg_hba.conf` source-address rules.

The project investigates whether these controls prevent unauthorized remote PostgreSQL authentication while preserving authorized access.

### Research Question

> To what extent do SCRAM-SHA-256 authentication and restrictive `pg_hba.conf` rules prevent unauthorized remote PostgreSQL connections using a previously known credential in a containerized environment?

---

## 2. Project Objectives

The project objectives were to:

1. Establish a controlled baseline for remote PostgreSQL authentication using the Week 1 Docker environment.
2. Verify the baseline PostgreSQL authentication method, listening configuration, and exposed database port.
3. Apply credential rotation while retaining `SCRAM-SHA-256` authentication.
4. Restrict PostgreSQL host-based authentication to the source address observed by the Docker networking layer.
5. Evaluate whether the previously known credential is rejected while an authorized current credential remains functional.

---

## 3. Scope and Authorization

All testing was performed within an authorized local/classroom laboratory environment owned or controlled by the project operator.

The primary target was the local Docker PostgreSQL container:

- Container: `medusa-postgres`
- Image: `postgres:15-alpine`
- Database: `medusa-store`
- PostgreSQL port: `5432`
- Docker network: `week1_medusa-net`

The project did not target third-party systems, public infrastructure, or systems without authorization.

No destructive exploitation, credential harvesting, brute-force activity, or unauthorized access was performed.

Passwords, private keys, API keys, access tokens, and other sensitive authentication material are intentionally excluded from the repository and final project deliverables.

---

## 4. Environment

### Primary Docker Environment

| Component | Configuration |
|---|---|
| Host | Windows with Docker Desktop |
| Container | `medusa-postgres` |
| PostgreSQL image | `postgres:15-alpine` |
| Database | `medusa-store` |
| PostgreSQL port | `5432` |
| Docker network | `week1_medusa-net` |
| Docker bridge | `DOCKER_NETWORK` |
| PostgreSQL container address | `POSTGRES_CONTAINER` |
| Docker gateway/source observed by PostgreSQL | `DOCKER_GATEWAY` |
| Kali testing host | `KALI_HOST` |
| Windows host | `WINDOWS_HOST` |

### Tools

The project used tools including:

- Docker Desktop
- Docker CLI
- PostgreSQL `psql`
- PostgreSQL system/catalog views
- Kali Linux
- Windows PowerShell/CMD
- Git
- SHA-256 hashing using Windows `certutil`

---

## 5. Methodology

The experiment followed a before-and-after design.

### Phase 1 — Baseline

The PostgreSQL container was examined before hardening to establish:

- Current `pg_hba.conf` host rules;
- PostgreSQL listening configuration;
- Published port configuration;
- Password authentication format;
- Remote authentication using the previously known lab credential;
- Client address observed by PostgreSQL.

### Phase 2 — Hardening

The following controls were applied:

1. The previously known PostgreSQL credential was rotated.
2. The password verifier was checked to confirm that `SCRAM-SHA-256` remained in use.
3. The PostgreSQL host-based authentication rule was restricted to the Docker-observed source address.
4. PostgreSQL configuration was reloaded.

### Phase 3 — Validation

The hardened configuration was tested using:

- Previously known credential → expected DENY;
- Current authorized credential → expected ALLOW;
- PostgreSQL client-address inspection;
- Final `pg_hba.conf` verification;
- Final SCRAM verifier-format verification;
- Container health verification.

A supplementary direct-LAN Windows-to-Kali PostgreSQL experiment was also performed to validate source-IP behavior and demonstrate HBA restriction without Docker Desktop's address translation.

---

## 6. Key Technical Finding: Docker Source Address

During the primary Docker experiment, the connection originated from Kali at:

`KALI_HOST`

The connection was made to the Windows host:

`WINDOWS_HOST:5432`

However, PostgreSQL reported the client address as:

`DOCKER_GATEWAY`

This occurred because Docker Desktop's networking layer forwarded the connection through the Docker bridge.

The client address was verified from PostgreSQL using:

```sql
SELECT current_user, current_database(), inet_client_addr(), inet_client_port();