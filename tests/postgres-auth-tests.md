# PostgreSQL Authentication Hardening Test Procedure

## Purpose

This document records the reproducible commands used to evaluate PostgreSQL authentication hardening in the authorized local/classroom environment.

The primary experiment uses the Week 1 Docker PostgreSQL container.

The supplementary tests use PostgreSQL running directly on Kali Linux to validate source-IP visibility and HBA enforcement without Docker Desktop networking.

## Safety

All commands must be executed only against the authorized project environment.

Do not use real credentials or third-party systems.

Do not include actual password values in documentation, scripts, screenshots, or source files.

---

## Test 1 — Inspect Docker PostgreSQL HBA Configuration

### Command

Run from Windows CMD:

`docker exec medusa-postgres psql -U postgres -d medusa-store -c "SELECT line_number, type, database, user_name, address, auth_method, error FROM pg_hba_file_rules WHERE type = 'host';"`

### Expected Result

The active PostgreSQL host-based authentication rules are displayed.

### Recorded Baseline

The baseline configuration included a broad remote rule using:

`host all all all scram-sha-256`

### Evidence

`Evidence/BEFORE/Current_PostgreSQL_HBA_Configuration.png`

---

## Test 2 — Inspect PostgreSQL Listening Configuration

### Command

Run:

`docker exec medusa-postgres psql -U postgres -d medusa-store -c "SHOW listen_addresses;"`

### Expected Result

The configured PostgreSQL listening address is displayed.

### Recorded Baseline

The service was configured to listen on:

`*`

### Evidence

`Evidence/BEFORE/Listen_Addresses.png`

---

## Test 3 — Verify Docker PostgreSQL Port Exposure

### Command

Run:

`docker port medusa-postgres`

### Expected Result

PostgreSQL port 5432 is mapped to the Docker host.

### Recorded Baseline

Port 5432 was published on the host.

### Evidence

`Evidence/BEFORE/PostgreSQL_Expose_Port5432.png`

---

## Test 4 — Verify PostgreSQL Password Verifier

### Command

Run:

`docker exec medusa-postgres psql -U postgres -d medusa-store -c "SELECT rolname, rolpassword IS NOT NULL AS password_set, CASE WHEN rolpassword LIKE 'SCRAM-SHA-256%' THEN 'SCRAM-SHA-256' ELSE 'OTHER' END AS verifier_type FROM pg_authid WHERE rolname = 'postgres';"`

### Expected Result

The PostgreSQL role has a password and the verifier is identified as:

`SCRAM-SHA-256`

### Evidence

`Evidence/BEFORE/Valid_Baselne_Postgresql.png`

---

## Test 5 — Baseline Remote Authentication

### Procedure

From the authorized Kali client, connect to the Docker-published PostgreSQL service using the known baseline lab credential.

### Command

`psql -h <AUTHORIZED_HOST> -p 5432 -U postgres -d medusa-store`

### Expected Result

The known baseline credential authenticates successfully while the baseline configuration permits remote access.

### Additional Validation

Inside the PostgreSQL session:

`SELECT current_user, current_database(), inet_client_addr(), inet_client_port();`

### Recorded Observation

The Docker PostgreSQL instance observed the connection source as:

`DOCKER_GATEWAY`

### Evidence

`Evidence/BEFORE/Remote_Access_Evidence.png`

---

## Test 6 — Replace the PostgreSQL Credential

### Command

Run inside the authorized PostgreSQL administrative session:

`ALTER ROLE postgres WITH PASSWORD '<NEW_LAB_CREDENTIAL>';`

### Expected Result

The PostgreSQL role credential is replaced successfully.

### Security Note

The actual credential value is intentionally omitted from this procedure and repository documentation.

### Evidence

`Evidence/AFTER/Changing_Known_Password.png`

---

## Test 7 — Verify SCRAM After Credential Replacement

### Command

Run:

`docker exec medusa-postgres psql -U postgres -d medusa-store -c "SELECT rolname, rolpassword IS NOT NULL AS password_set, CASE WHEN rolpassword LIKE 'SCRAM-SHA-256%' THEN 'SCRAM-SHA-256' ELSE 'OTHER' END AS verifier_type FROM pg_authid WHERE rolname = 'postgres';"`

### Expected Result

The verifier remains:

`SCRAM-SHA-256`

### Evidence

`Evidence/AFTER/Password_Format_Check.png`

---

## Test 8 — Configure Restrictive Docker HBA Rule

### Condition

The Docker PostgreSQL connection was observed by PostgreSQL as originating from the Docker bridge gateway.

The restrictive test rule therefore used the observed Docker source:

`DOCKER_GATEWAY/32`

### Policy Form

`host all all <OBSERVED_AUTHORIZED_SOURCE> scram-sha-256`

### Expected Result

The remote authentication rule is restricted to the source address observed by PostgreSQL.

### Evidence

`Evidence/AFTER/HBA_Restriction_Rule_Set.png`

---

## Test 9 — Reload Docker PostgreSQL Configuration

### Command

Run:

`docker exec medusa-postgres psql -U postgres -d medusa-store -c "SELECT pg_reload_conf();"`

### Expected Result

The command returns:

`t`

### Evidence

`Evidence/AFTER/pg_reload_conf.png`

---

## Test 10 — Verify Active Docker HBA Rule

### Command

Run:

`docker exec medusa-postgres psql -U postgres -d medusa-store -c "SELECT line_number, type, database, user_name, address, auth_method, error FROM pg_hba_file_rules WHERE type = 'host';"`

### Expected Result

The restrictive rule is active and the `error` column is empty.

### Evidence

`Evidence/AFTER/Active_HBA_Rule.png`

---

## Test 11 — Verify Docker PostgreSQL Final State

### Procedure

Verify that PostgreSQL remains operational after the authentication-hardening changes.

### Expected Result

The PostgreSQL service remains available and the final configuration can be queried successfully.

### Evidence

`Evidence/AFTER/Current_State.png`

`Evidence/AFTER/Active_HBA_Rule_2.png`

---

# Supplementary Direct-LAN Validation

## Test 12 — Windows to Kali PostgreSQL Connection

### Procedure

From Windows, connect directly to the authorized Kali PostgreSQL server:

`psql -h KALI_HOST -p 5432 -U postgres -d postgres`

Use the authorized lab credential when prompted.

### Expected Result

Authentication succeeds.

Inside the session:

`SELECT current_user, current_database(), inet_client_addr(), inet_client_port();`

### Observed Result

PostgreSQL reported:

`inet_client_addr = WINDOWS_HOST`

### Interpretation

The direct-LAN connection preserved the actual Windows client source address.

### Evidence

`Evidence/Network Validation/Windows_to_Kali_Source_IP.png`

---

## Test 13 — Previous Credential Rejection

### Procedure

From Windows, attempt to connect to the Kali PostgreSQL server using the previous lab credential.

### Expected Result

Authentication is rejected.

### Observed Result

PostgreSQL returned:

`password authentication failed for user "postgres"`

### Interpretation

Credential rotation prevented authentication using the previous credential.

### Evidence

`Evidence/Network Validation/Windows_to_Kali_Old_Password_Rejected.png`

---

## Test 14 — Direct-LAN HBA Source Restriction

### Temporary Condition

The Kali HBA rule was temporarily restricted to:

`KALI_HOST/32`

The Windows client address was:

`WINDOWS_HOST`

### Configuration Reload

The modified HBA configuration was reloaded before testing.

### Procedure

From Windows, attempt a PostgreSQL connection to Kali using the authorized new lab credential.

### Expected Result

The connection is denied because the Windows source address is outside the permitted HBA range.

### Observed Result

PostgreSQL returned:

`no pg_hba.conf entry for host "WINDOWS_HOST"`

### Interpretation

The HBA source-address restriction denied the Windows client before successful password authentication could occur.

### Evidence

`Evidence/Network Validation/Windows_to_Kali_HBA_Denied.png`

---

## Test 15 — Restore Temporary Kali HBA Configuration

### Configuration

The temporary `/32` restriction was restored to:

`AUTHORIZED_LAN_RANGE`

### Reload Command

`sudo -u postgres psql -c "SELECT pg_reload_conf();"`

### Expected Result

The command returns:

`t`

### Final Verification

The active HBA configuration was queried and confirmed to contain:

`AUTHORIZED_LAN`

with:

`scram-sha-256`

and no HBA configuration errors.

---

# Interpretation Guidelines

## Observation

Only directly observed command output, PostgreSQL responses, configuration values, and captured evidence should be classified as observations.

## Interpretation

An interpretation explains what the observation demonstrates within the tested environment.

## Inference

Broader conclusions about PostgreSQL authentication-hardening effectiveness must be supported by multiple observations and must account for environmental limitations.

## Limitation

The Docker Desktop networking layer caused PostgreSQL to observe the Docker bridge gateway rather than the original client address.

Therefore, source-address HBA behavior in the primary Docker environment must be interpreted according to the network path visible to PostgreSQL.

## Reproducibility

The test procedure should be executed only against the authorized project environment.

Temporary configuration changes must be restored after testing.