# PostgreSQL Authentication Hardening Test Matrix

## Test Scope

The tests were performed in the authorized local/classroom environment. The primary experiment used the Week 1 Docker PostgreSQL stack. A supplementary direct-LAN PostgreSQL test on Kali Linux was used to validate source-IP visibility and HBA enforcement.

## Test Matrix

| Test ID | Input / Condition | Expected Result | Observed Result | Evidence ID | Interpretation |
|---|---|---|---|---|---|
| T-001 | Docker PostgreSQL accessed remotely using the known baseline credential | The known baseline credential should authenticate successfully if remote access is permitted. | Remote authentication succeeded. | E-004, E-005 | Baseline configuration permitted remote authentication using the known credential. |
| T-002 | Docker PostgreSQL HBA configuration inspected before hardening | The active HBA rules should reveal how remote connections are authenticated. | A remote `host all all all scram-sha-256` rule was present. | E-001 | The baseline already used SCRAM-SHA-256 for remote host authentication. |
| T-003 | Docker PostgreSQL `listen_addresses` inspected before hardening | PostgreSQL should reveal whether it accepts connections beyond localhost. | PostgreSQL was configured to listen on `*`. | E-002 | The database service was configured to accept connections on available interfaces. |
| T-004 | Docker PostgreSQL host port inspected before hardening | PostgreSQL port 5432 should be mapped to the Docker host if remote host access is possible. | Port 5432 was published to the host. | E-003, E-007 | Docker exposed PostgreSQL through the host's port 5432 before and after hardening. |
| T-005 | Docker `postgres` role password changed | The previous credential should no longer be valid after the password replacement. | Password replacement completed successfully. | Not retained | Credential rotation was successfully performed. The screenshot containing the actual credential was intentionally excluded from the repository. |
| T-006 | PostgreSQL role verifier inspected after password change | The password verifier should remain SCRAM-SHA-256. | `SCRAM-SHA-256` was confirmed. | E-006 | Credential replacement retained the intended authentication mechanism. |
| T-007 | PostgreSQL configuration reloaded after HBA modification | PostgreSQL should reload the modified configuration successfully. | `pg_reload_conf()` returned `t`. | E-008 | The modified configuration was successfully reloaded. |
| T-008 | Restrictive HBA rule configured in Docker PostgreSQL | The HBA configuration should restrict the permitted source address/range while retaining SCRAM authentication. | A restrictive source-address HBA rule was configured. | E-009 | HBA source restriction was implemented as part of the hardening procedure. |
| T-009 | Active Docker HBA rules inspected after restriction | The restrictive rule should appear in the active PostgreSQL HBA configuration without errors. | The restrictive rule was active and no HBA error was reported. | E-010 | The intended HBA restriction was successfully applied. |
| T-010 | Docker PostgreSQL final state inspected after hardening | PostgreSQL should remain operational after the hardening changes. | Final PostgreSQL state was captured after configuration changes. | E-011 | The hardening procedure did not intentionally leave the primary PostgreSQL service in a failed state. |
| T-011 | Docker HBA configuration verified again after testing | The active configuration should remain syntactically valid and show the final rule state. | Active HBA configuration was verified without errors. | E-012 | Final HBA configuration remained valid after testing. |
| T-012 | Windows connected directly to Kali PostgreSQL using the authorized new credential | The authorized credential should permit a direct-LAN connection. | Connection succeeded; PostgreSQL identified client address as `WINDOWS_HOST`. | E-013 | Direct-LAN PostgreSQL preserved the actual Windows source IP. |
| T-013 | Windows attempted direct-LAN connection using the previous credential | The previous credential should be rejected after credential rotation. | PostgreSQL returned `password authentication failed for user "postgres"`. | E-014 | Credential rotation prevented authentication using the previous credential. |
| T-014 | Kali HBA temporarily restricted to `KALI_HOST/32`; Windows attempted connection using the authorized new credential | Windows should be denied because its source address is `WINDOWS_HOST`, which is outside the permitted `/32`. | PostgreSQL returned `no pg_hba.conf entry for host "WINDOWS_HOST"`. | E-015 | HBA source-address restriction denied the Windows client despite use of the authorized credential. |

## Cleanup and Configuration Restoration

After the supplementary HBA restriction test, the Kali PostgreSQL HBA configuration was restored to the authorized `AUTHORIZED_LAN_RANGE` range and reloaded.

This restoration was a cleanup action rather than an experimental test and therefore does not have a separate test ID or evidence item.

## Key Findings

1. The baseline Docker PostgreSQL service permitted remote authentication using the known baseline credential.
2. The baseline remote HBA rule already used `scram-sha-256`; therefore, the hardening exercise did not introduce SCRAM from an unsecured plaintext-password configuration.
3. Changing the PostgreSQL credential invalidated the previous credential.
4. The PostgreSQL role continued to use the `SCRAM-SHA-256` verifier after the credential change.
5. Docker Desktop networking caused PostgreSQL in the containerized environment to observe the published connection source as the Docker bridge gateway (`DOCKER_GATEWAY`) rather than the original Windows/Kali client address.
6. The supplementary direct-LAN test demonstrated that PostgreSQL can observe the actual Windows source address (`WINDOWS_HOST`) when Docker Desktop NAT is not involved.
7. The supplementary HBA test demonstrated that a source-address restriction can deny a reachable client even when the client uses the authorized password.
8. The Kali HBA configuration was restored to the authorized `AUTHORIZED_LAN_RANGE` range after the temporary restriction test.

## Interpretation Discipline

### Observation

Statements describing directly observed command output, PostgreSQL responses, configuration values, or captured evidence.

### Interpretation

Conclusions directly supported by the observed results, such as successful credential rotation or HBA-based source-address denial.

### Inference

Broader conclusions about the effectiveness of authentication hardening that are derived from the observed tests.

### Limitation

The Docker Desktop networking path affects the source address visible to PostgreSQL. Therefore, source-IP-based HBA testing in the primary Docker environment must account for Docker's bridge/NAT behavior.

### Recommendation

Production PostgreSQL deployments should use strong credential management, SCRAM-SHA-256, restrictive `pg_hba.conf` rules, and network-level access controls appropriate to the deployment architecture.