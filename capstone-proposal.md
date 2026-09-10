# Capstone Project Proposal

## Project Title

**Evaluating PostgreSQL Authentication Hardening Against Known-Credential Exploitation in a Containerized Stack**

## 1. Problem Statement

Containerized database services can expose authentication interfaces through published network ports. If a known, weak, reused, or default-style credential remains valid, an attacker who can reach the database service may be able to authenticate without exploiting a software vulnerability.

PostgreSQL provides authentication controls including password-based authentication and host-based access rules through `pg_hba.conf`. These controls determine which clients are permitted to connect and how they must authenticate.

This project evaluates how credential replacement, SCRAM-SHA-256 authentication, and restrictive host-based authentication rules affect remote PostgreSQL access within an authorized local/classroom environment.

The project specifically examines whether a previously known laboratory credential can be used to access the PostgreSQL service before hardening and whether credential rotation and restrictive HBA rules reduce or prevent that access afterward.

## 2. Research Question

> To what extent do SCRAM-SHA-256 authentication and restrictive `pg_hba.conf` rules prevent unauthorized remote PostgreSQL connections using a previously known credential in a containerized environment?

## 3. Aim

The aim of this project is to experimentally evaluate PostgreSQL authentication hardening controls against remote access using a previously known credential in a controlled containerized environment.

The evaluation focuses on the interaction between credential rotation, SCRAM-SHA-256 authentication, and source-address restrictions implemented through `pg_hba.conf`.

## 4. Objectives

The project objectives are to:

1. Examine the baseline PostgreSQL authentication configuration in the authorized Week 1 containerized environment, including the active `pg_hba.conf` rules, listening configuration, exposed port, and authentication mechanism.

2. Demonstrate whether a previously known PostgreSQL laboratory credential permits remote access to the database before hardening.

3. Implement authentication hardening by rotating the known laboratory credential, maintaining `SCRAM-SHA-256` authentication, and applying restrictive `pg_hba.conf` rules appropriate to the observed network path.

4. Test the hardened configuration using authorized connection attempts with the previously known credential, the new laboratory credential, and restricted-source conditions, and record the resulting authentication decisions.

5. Evaluate the effectiveness and limitations of SCRAM-SHA-256, credential rotation, and restrictive `pg_hba.conf` rules against unauthorized remote PostgreSQL access in a containerized environment.

## 5. Hypothesis

### Null Hypothesis

Changing the PostgreSQL credential and applying restrictive `pg_hba.conf` rules will not materially reduce unauthorized remote authentication when the database service remains reachable.

### Alternative Hypothesis

Changing the PostgreSQL credential and applying restrictive `pg_hba.conf` rules will reduce unauthorized remote authentication by invalidating the previously known credential and restricting which source addresses are permitted to authenticate.

The alternative hypothesis is supported if the previously known credential no longer authenticates successfully after credential rotation and source-address restrictions prevent connections from unauthorized sources.

## 6. Variables

### Independent Variables

- PostgreSQL credential state.
- `pg_hba.conf` source-address restriction.
- Authentication method configuration.

### Dependent Variable

- PostgreSQL authentication outcome for remote connection attempts.

### Controlled Variables

- PostgreSQL role under test.
- Database service.
- Test environment.
- PostgreSQL authentication mechanism.
- Test client.
- Network path for each experiment.
- Test procedure.
- Database and role used for connection testing.

## 7. Scope

The primary experiment is restricted to the authorized Week 1 Docker PostgreSQL environment.

The primary evaluation covers:

- Baseline PostgreSQL authentication configuration.
- Baseline HBA rules.
- PostgreSQL listening configuration.
- Docker port exposure.
- Authentication verifier.
- Remote access using the previously known laboratory credential.
- Credential rotation.
- Continued use of SCRAM-SHA-256.
- Restrictive `pg_hba.conf` source-address rules.
- Post-hardening authentication attempts.
- PostgreSQL configuration reload and verification.
- Evidence collection and comparison of before-and-after results.

A supplementary direct-LAN PostgreSQL test on Kali Linux is included to validate source-IP visibility and HBA enforcement independently of Docker Desktop's networking layer.

The supplementary experiment does not replace the primary Docker experiment. It is used to provide additional evidence about how PostgreSQL observes client source addresses when Docker's published-port networking is not involved.

## 8. Out of Scope

The following activities are outside the project scope:

- Testing against public or third-party systems.
- Unauthorized credential attacks.
- Exploitation of real user accounts.
- Persistence mechanisms.
- Privilege escalation beyond the controlled database test.
- Destructive database operations.
- Production infrastructure testing.
- Denial-of-service testing.
- Password cracking or credential harvesting.
- Exploitation of vulnerabilities unrelated to PostgreSQL authentication hardening.
- Repeating the Week 1 Trivy vulnerability-scanning objective.

## 9. Test Environment

The primary test environment uses the authorized Week 1 Docker stack containing a PostgreSQL database service.

The primary PostgreSQL service uses:

- PostgreSQL 15.
- Alpine-based PostgreSQL container image.
- Docker Desktop Linux containers.
- A Docker bridge network.
- Published PostgreSQL port 5432.
- A PostgreSQL database and role created for the authorized laboratory environment.

The test client is Kali Linux.

A supplementary test uses direct Windows-to-Kali PostgreSQL connectivity to validate PostgreSQL source-IP visibility and HBA enforcement outside the Docker published-port path.

All identities, credentials, and database values used for the project are laboratory values.

## 10. Test Strategy

The experiment follows a controlled before-and-after design.

### Phase 1 — Baseline Examination

The baseline configuration will be documented before hardening.

The following will be examined:

- PostgreSQL `pg_hba.conf` configuration.
- PostgreSQL listening configuration.
- Docker port exposure.
- PostgreSQL authentication verifier.
- Remote authentication using the previously known laboratory credential.
- Client address observed by PostgreSQL.

The purpose of this phase is to establish the original authentication and network exposure state.

### Phase 2 — Authentication Hardening

The following controls will be applied:

- Replacement of the previously known PostgreSQL credential.
- Verification that SCRAM-SHA-256 remains active.
- Restriction of the applicable `pg_hba.conf` source address.
- PostgreSQL configuration reload.
- Verification that the new configuration is active.

The purpose of this phase is to change the authentication state while keeping the database service available for controlled validation.

### Phase 3 — Post-Hardening Validation

Authorized connection attempts will be performed to determine whether:

- The previously known credential is rejected.
- The new laboratory credential is accepted when the source is permitted.
- The active HBA rule matches the intended source.
- PostgreSQL remains operational.
- The observed source address is consistent with the configured HBA rule.

Authentication failures will be interpreted according to PostgreSQL's reported reason, distinguishing between:

- Password authentication failure.
- HBA rule denial.
- Successful authentication.

### Phase 4 — Supplementary Direct-LAN Validation

A separate direct Windows-to-Kali PostgreSQL test will be used to examine source-address behavior without the Docker Desktop published-port path.

The supplementary test will:

- Confirm the client source IP observed by PostgreSQL.
- Demonstrate rejection of the previously known credential.
- Demonstrate HBA denial when the source address does not match the permitted rule.
- Restore the temporary HBA configuration after testing.

The supplementary results will be reported separately from the primary Docker experiment.

## 11. Expected Results

The expected results are:

1. The baseline configuration will permit the previously known laboratory credential to authenticate remotely when the connection matches the baseline HBA policy.

2. The PostgreSQL authentication verifier will show that SCRAM-SHA-256 is active.

3. After credential rotation, the previously known credential will fail password authentication.

4. The new laboratory credential will authenticate successfully when the connection originates through the permitted network path.

5. A restrictive HBA rule will deny connections whose observed source address does not match the permitted source.

6. The Docker experiment will demonstrate that Docker Desktop's networking layer can affect the source address observed by PostgreSQL.

7. The supplementary direct-LAN experiment will demonstrate PostgreSQL visibility of the actual Windows client source address when the Docker published-port path is not involved.

The actual findings will be determined from the recorded test results and evidence rather than assumed from these expectations.

## 12. Expected Contribution

The project is expected to demonstrate that PostgreSQL authentication security depends on multiple interacting controls rather than password authentication alone.

The results should provide evidence about:

- The security impact of retaining a previously known credential.
- The effect of credential rotation.
- Continued use of SCRAM-SHA-256.
- The effectiveness of source-address restrictions through `pg_hba.conf`.
- The difference between password authentication failure and HBA denial.
- The effect of Docker Desktop networking on source-IP-based PostgreSQL controls.
- The need for complementary network-level controls where appropriate.

## 13. Ethical and Safety Controls

All testing will remain within the authorized local/classroom environment.

Safety controls include:

- Using only lab-controlled systems.
- Using fictional or laboratory credentials.
- Avoiding third-party systems.
- Avoiding unauthorized credential attacks.
- Avoiding destructive database operations.
- Avoiding production infrastructure.
- Capturing evidence during testing.
- Recording test outcomes without retaining unnecessary secrets.
- Restoring temporary configuration changes after testing.
- Removing or redacting passwords, private keys, tokens, and other secrets from project artifacts.
- Using the minimum privileges and access necessary for the experiment.

Testing will stop if the activity unexpectedly affects systems outside the authorized laboratory scope.

## 14. Evidence Plan

Evidence will be collected during each testing phase and organized according to the project evidence structure:

Evidence/
├── BEFORE/
├── AFTER/
└── Network Validation/

## 15. Evidence Integrity and Handling

Evidence will be collected at the time of testing and retained in its original or appropriately redacted form.

Where practical, important evidence files will be assigned integrity hashes to support verification that the evidence has not been altered after collection.

Evidence records will document:

- Evidence ID.
- Filename or source.
- Associated test or event.
- Date and time of collection.
- Description of the evidence.
- Redaction status.
- Source system or tool where relevant.

Passwords, private keys, access tokens, and other unnecessary secrets will not be included in the final project artifacts.

## 16. Analysis Approach

The analysis will distinguish between direct observations, interpretations, inferences, assumptions, limitations, and recommendations.

For each test, the observed PostgreSQL response will be compared with the expected result.

Particular attention will be given to distinguishing:

- Successful authentication from network reachability.
- Password authentication failure from HBA rule denial.
- Docker-observed source addresses from the actual client address.
- Primary Docker experiment results from supplementary direct-LAN validation results.

The before-and-after results will then be used to assess whether the alternative hypothesis is supported within the limits of the experiment.

The analysis will avoid treating a successful network connection as evidence of successful authentication. Authentication decisions will be determined from PostgreSQL's reported connection and authentication outcomes.

## 17. Limitations

The project has several limitations:

- The primary experiment uses a local Docker Desktop environment rather than production infrastructure.
- Docker Desktop networking can alter the source address observed by PostgreSQL.
- The experiment evaluates a controlled set of authentication conditions rather than all PostgreSQL authentication mechanisms.
- The previously known laboratory credential represents a known-credential scenario and does not establish the prevalence of such credentials in real deployments.
- The supplementary Kali experiment uses a different network path and PostgreSQL installation from the primary Docker experiment.
- The experiment does not evaluate all possible network controls, firewall configurations, or identity-management mechanisms.
- Results obtained from the laboratory environment may not generalize directly to other container runtimes or production architectures.
- The experiment focuses on authentication and source-address controls and does not attempt to assess the full security posture of the PostgreSQL or Docker environment.

These limitations will be considered when interpreting the results and making recommendations.

## 18. Success Criteria

The project will be considered successful if it:

- Establishes a reproducible baseline.
- Demonstrates the effect of the previously known credential before hardening.
- Verifies that SCRAM-SHA-256 remains active.
- Applies and validates credential rotation.
- Applies and validates restrictive HBA rules.
- Demonstrates successful and denied authentication outcomes supported by the test design.
- Documents Docker source-IP behavior.
- Separately validates direct-LAN source-IP behavior.
- Provides traceable evidence for the reported results.
- Protects sensitive information in the repository and final submission.
- Provides practical recommendations based on the observed evidence.

## 19. Project Deliverables

The final project will contain:

- Final written report.
- Project presentation.
- Project README.
- Capstone project proposal.
- Hardened PostgreSQL security policy.
- PostgreSQL authentication test procedure.
- Completed test matrix.
- Evidence index.
- Before-and-after evidence.
- Supplementary network validation evidence.
- AI use and verification record.
- References.
- Supporting diagrams and analysis artifacts.

## 20. Repository Structure

The project repository will use the following structure:

week1/
├── README.md
├── docker-compose.yml
├── capstone-proposal.md
├── AI-use-record.md
├── report/
├── src/
│   └── hardened-postgresql-policy.md
├── tests/
│   └── postgres-auth-tests.md
├── Evidence/
│   ├── BEFORE/
│   ├── AFTER/
│   ├── Network Validation/
│   ├── evidence-index.md
│   └── test-matrix.md
├── analysis/
├── diagrams/
├── references/
└── .gitignore/

## 21. Safety Boundary

This project was conducted strictly within an authorized local and classroom-controlled environment. The testing was limited to PostgreSQL instances created and administered for the project and did not target third-party systems, public infrastructure, or accounts belonging to other individuals.

The project did not involve unauthorized credential attacks, brute-force authentication attempts, exploitation of production systems, or attempts to obtain credentials from external sources. The previously known credential used during the baseline test was a controlled laboratory credential associated with the authorized Week 1 environment.

Testing activities were limited to controlled authentication verification, PostgreSQL configuration examination, credential rotation, `pg_hba.conf` rule modification, connection testing, and validation of the resulting access-control behavior.

Sensitive information was excluded from the report, repository, presentation, and supporting documentation wherever practical. Password values, private keys, access tokens, and other authentication secrets are not included in the project artifacts. Where screenshots or logs could expose sensitive information, they should be reviewed and redacted before final submission.

The Docker PostgreSQL environment remained the primary experimental system. The supplementary Kali PostgreSQL instance was used only to validate direct-LAN source-address and HBA behavior and was not used as a replacement for the primary containerized experiment.

The testing was stopped after the required before-and-after authentication behavior, HBA behavior, and network-source observations had been established. No further intrusive testing was performed because it was outside the defined project scope.

## Conclusion

This project evaluated the extent to which SCRAM-SHA-256 authentication and restrictive PostgreSQL `pg_hba.conf` rules can reduce the risk of unauthorized remote access when a previously known credential is available in a containerized environment.

The primary Docker experiment demonstrated that the baseline PostgreSQL instance accepted a remote connection using the known laboratory credential. The baseline configuration already used SCRAM-SHA-256 for host authentication and exposed PostgreSQL through the published port, demonstrating that the presence of a stronger authentication mechanism alone does not prevent access when a valid credential remains known to an unauthorized party.

The hardening phase introduced credential rotation while retaining SCRAM-SHA-256 and applied a restrictive `pg_hba.conf` rule based on the source address observed by PostgreSQL through the Docker networking layer. After the changes were applied and the PostgreSQL configuration was reloaded, the previously known credential was rejected, while the authorized connection using the replacement credential remained successful. This demonstrated that credential rotation directly invalidated the previously known authentication secret, while HBA restrictions provided an additional network-level access-control layer.

The supplementary direct-LAN PostgreSQL experiment provided an additional validation of source-address behavior without the Docker Desktop network translation observed in the primary experiment. In this environment, PostgreSQL observed the Windows host's actual LAN address, allowing the HBA rule to demonstrate how source-network restrictions can permit or deny connections based on the originating address.

The findings therefore support the project hypothesis within the defined laboratory scope. Credential rotation prevented continued use of the previously known credential, while restrictive HBA rules provided an additional control against connections from unauthorized source addresses. However, the experiment does not demonstrate that SCRAM-SHA-256 alone prevents unauthorized access, because SCRAM protects the authentication exchange but does not make a valid, known credential invalid.

Overall, the results show that PostgreSQL authentication hardening is most effective when implemented as a layered control strategy. Credential lifecycle management, strong authentication, restrictive network access rules, controlled port exposure, and continuous configuration validation should be treated as complementary controls rather than independent safeguards.

The project also demonstrated the importance of understanding the underlying network architecture when designing PostgreSQL access-control rules. In containerized environments, the address observed by PostgreSQL may differ from the original client address because of NAT or other networking behavior. Consequently, HBA rules should be based on verified connection-source observations rather than assumptions about how traffic is routed.

Within the limitations of the authorized classroom environment, the project provides evidence that combining credential rotation with SCRAM-SHA-256 and restrictive `pg_hba.conf` rules can substantially reduce the likelihood of successful unauthorized remote PostgreSQL authentication using a previously known credential.

# References

1. PostgreSQL Global Development Group. (2026). *PostgreSQL Documentation: Client Authentication*. PostgreSQL Documentation.

2. PostgreSQL Global Development Group. (2026). *PostgreSQL Documentation: `pg_hba.conf` File*. PostgreSQL Documentation.

3. PostgreSQL Global Development Group. (2026). *PostgreSQL Documentation: Password Authentication*. PostgreSQL Documentation.

4. PostgreSQL Global Development Group. (2026). *PostgreSQL Documentation: Authentication Methods*. PostgreSQL Documentation.

5. Docker Inc. (2026). *Docker Documentation: Networking Overview*. Docker Documentation.

6. Docker Inc. (2026). *Docker Documentation: Bridge Network Driver*. Docker Documentation.

7. OWASP Foundation. (2021). *OWASP Top 10: A05 – Security Misconfiguration*. OWASP.

8. OWASP Foundation. (2021). *OWASP Top 10: A07 – Identification and Authentication Failures*. OWASP.

9. National Institute of Standards and Technology. (2020). *Security and Privacy Controls for Information Systems and Organizations (SP 800-53 Rev. 5)*. NIST.

10. National Institute of Standards and Technology. (2024). *Digital Identity Guidelines: Authentication and Lifecycle Management (SP 800-63B)*. NIST.

11. Center for Internet Security. (2024). *CIS Benchmarks*. Center for Internet Security.

12. PostgreSQL Global Development Group. (2026). *PostgreSQL Documentation: `pg_reload_conf` and Configuration Reloading*. PostgreSQL Documentation.

# Appendix A — Evidence Index

The evidence collected for this project is organized by test/event identifier. All evidence was collected from the authorized local/classroom environment. Password values, private keys, access tokens, and other authentication secrets are intentionally excluded from the report and repository.

| Evidence ID | File / Source | Test / Event | Description | Redaction Status |
|---|---|---|---|---|
| E-001 | `Evidence/BEFORE/Current_PostgreSQL_HBA_Configuration.png` | T-002 | Baseline PostgreSQL host-based authentication rules before hardening. | Reviewed; no password value included |
| E-002 | `Evidence/BEFORE/Listen_Addresses.png` | T-003 | Baseline PostgreSQL listening-address configuration. | Reviewed |
| E-003 | `Evidence/BEFORE/PostgreSQL_Expose_Port5432.png` | T-004 | Verification that PostgreSQL port 5432 was published by the Docker container. | Reviewed |
| E-004 | `Evidence/BEFORE/Valid_Baselne_Postgresql.png` | T-001 | Baseline PostgreSQL authentication verification showing the configured authentication mechanism. | Reviewed |
| E-005 | `Evidence/BEFORE/Remote_Access_Evidence.png` | T-001 | Baseline remote connection evidence showing successful access from the authorized test client and the source address observed by PostgreSQL. | Reviewed; credential value excluded |
| E-006 | `Evidence/AFTER/Changing_Known_Password.png` | T-005 | Evidence of the controlled PostgreSQL credential-rotation step. | Reviewed; password value excluded |
| E-007 | `Evidence/AFTER/Password_Format_Check.png` | T-006 | Verification that PostgreSQL continued to use SCRAM-SHA-256 after credential rotation. | Reviewed |
| E-008 | `Evidence/AFTER/Postgres_Port.png` | T-004 | Verification of PostgreSQL port exposure after hardening. | Reviewed |
| E-009 | `Evidence/AFTER/pg_reload_conf.png` | T-007 | Evidence that the PostgreSQL configuration was reloaded after HBA changes. | Reviewed |
| E-010 | `Evidence/AFTER/HBA_Restriction_Rule_Set.png` | T-008 | Evidence of the restrictive HBA rule applied during the hardening process. | Reviewed |
| E-011 | `Evidence/AFTER/Active_HBA_Rule.png` | T-009 | Verification of the active PostgreSQL HBA rule following the configuration reload. | Reviewed |
| E-012 | `Evidence/AFTER/Current_State.png` | T-010 | Post-hardening authentication state and successful authorized connection evidence. | Reviewed; credential value excluded |
| E-013 | `Evidence/AFTER/Active_HBA_Rule_2.png` | T-011 | Final verification of the active restrictive HBA configuration. | Reviewed |
| E-014 | `Evidence/Network Validation/Windows_to_Kali_Source_IP.png` | T-012 | Supplementary direct-LAN validation showing the source address observed by the Kali PostgreSQL server. | Reviewed |
| E-015 | `Evidence/Network Validation/Windows_to_Kali_Old_Password_Rejected.png` | T-013 | Supplementary validation showing rejection of the previously known credential after credential rotation. | Reviewed; credential value excluded |
| E-016 | `Evidence/Network Validation/Windows_to_Kali_HBA_Denied.png` | T-014 | Supplementary validation showing HBA denial when the client source address did not match the restrictive rule. | Reviewed |

## Evidence Handling Notes

Evidence was collected during the execution of the project's authorized test procedures. The evidence index provides traceability between individual files, test identifiers, and the observations reported in the Results and Analysis chapters.

The primary experiment evidence is associated with the containerized PostgreSQL environment. Evidence E-014 through E-016 relates to the supplementary direct-LAN PostgreSQL validation performed on the Kali Linux system.

Evidence files should be reviewed before final submission to ensure that no authentication secrets, private keys, access tokens, or unnecessary sensitive information are visible. Where necessary, screenshots should be redacted before being committed to the repository.

The evidence files are retained in their corresponding `BEFORE`, `AFTER`, and `Network Validation` directories to preserve the distinction between baseline observations, post-hardening observations, and supplementary network validation.

# Appendix B — Test Matrix

The following test matrix provides a traceable summary of the authentication-hardening tests performed during the project. The primary Docker experiment is distinguished from the supplementary direct-LAN validation.

| Test ID | Input / Condition | Expected Result | Observed Result | Evidence ID | Interpretation |
|---|---|---|---|---|---|
| T-001 | Authorized remote client uses the previously known laboratory credential against the baseline Docker PostgreSQL service. | Connection should succeed if the credential remains valid and the HBA configuration permits the connection. | Connection succeeded. PostgreSQL reported the Docker-observed client source address. | E-004, E-005 | Baseline confirmed that a previously known valid credential could be used for remote authentication. |
| T-002 | Inspect the baseline PostgreSQL host-based authentication rules. | A host rule permitting the remote connection should be identifiable. | Baseline host authentication included a broad `host all all all scram-sha-256` rule. | E-001 | SCRAM-SHA-256 was already enabled, but the baseline HBA rule was broad. |
| T-003 | Inspect PostgreSQL `listen_addresses`. | PostgreSQL should be listening on an address that permits the tested remote connection. | PostgreSQL was configured to listen on `*`. | E-002 | The service was configured to accept connections through available network interfaces. |
| T-004 | Inspect Docker PostgreSQL port publication. | PostgreSQL port 5432 should be externally published for the baseline remote test. | Port 5432 was published to the host. | E-003, E-008 | Published port exposure enabled the remote connection path used during testing. |
| T-005 | Rotate the PostgreSQL authentication credential. | The previously known credential should no longer authenticate successfully. | Credential rotation was completed successfully. | E-006 | Credential lifecycle control was applied. |
| T-006 | Verify PostgreSQL authentication mechanism after credential rotation. | SCRAM-SHA-256 should remain active. | PostgreSQL continued to report `SCRAM-SHA-256`. | E-007 | Credential rotation did not require weakening the authentication mechanism. |
| T-007 | Reload PostgreSQL configuration after modifying HBA rules. | PostgreSQL should accept the new HBA configuration without an error. | Configuration reload completed successfully. | E-009 | The updated access-control configuration became active. |
| T-008 | Apply a restrictive HBA rule based on the source address observed through Docker networking. | A connection from a source not matching the rule should be denied by HBA. | The initial restrictive rule did not match the Docker-observed source address and produced an HBA denial. | E-010 | Docker networking affected the source address visible to PostgreSQL and had to be accounted for. |
| T-009 | Verify the active Docker HBA configuration after adjustment. | The intended restrictive rule should appear as an active rule without configuration errors. | The active HBA rule matched the Docker-observed source address and used SCRAM-SHA-256. | E-011 | The HBA configuration was aligned with the observed Docker network path. |
| T-010 | Connect remotely using the authorized replacement credential after hardening. | Authorized authentication should succeed. | Connection succeeded using the replacement credential. | E-012 | Hardening preserved authorized access while invalidating the previously known credential. |
| T-011 | Inspect the final active Docker HBA rules. | The final configuration should contain the restrictive source-address rule and SCRAM-SHA-256. | Final HBA configuration showed the restrictive rule with SCRAM-SHA-256 and no reported errors. | E-013 | Final Docker access control was more restrictive than the baseline configuration. |
| T-012 | Supplementary Windows-to-Kali direct-LAN connection using the authorized replacement credential. | PostgreSQL should observe the Windows client's actual LAN source address. | PostgreSQL observed the Windows host's LAN address. | E-014 | Direct-LAN testing demonstrated source-address behavior without the Docker source translation observed in the primary experiment. |
| T-013 | Supplementary Windows-to-Kali connection using the previously known credential after rotation. | Authentication should fail because the credential has been changed. | PostgreSQL rejected the previously known credential. | E-015 | Credential rotation prevented continued authentication using the known credential. |
| T-014 | Supplementary direct-LAN connection from a source address excluded by the HBA rule. | PostgreSQL should deny the connection before successful authentication. | PostgreSQL returned an HBA `no pg_hba.conf entry` denial. | E-016 | HBA source-address restrictions can prevent connections independently of whether a credential is valid. |
| T-015 | Restore the supplementary Kali PostgreSQL HBA configuration after validation. | The authorized test-network rule should be restored and PostgreSQL should reload the configuration successfully. | The original supplementary network rule was restored and the final active HBA configuration was verified. | N/A — cleanup verification | Cleanup returned the supplementary environment to its intended laboratory state. |

## Matrix Interpretation

The test sequence demonstrates two complementary control effects.

First, credential rotation invalidated the previously known credential. This was demonstrated by the rejection of the old credential after the password change.

Second, restrictive HBA rules controlled whether a client source address was permitted to reach PostgreSQL authentication. The Docker experiment demonstrated that the source address visible to PostgreSQL can differ from the original client address because of container networking. The supplementary direct-LAN experiment demonstrated the corresponding behavior when PostgreSQL could observe the client's LAN address directly.

The matrix therefore supports the conclusion that PostgreSQL authentication hardening is stronger when credential lifecycle controls and network-level access restrictions are implemented together rather than relying on SCRAM-SHA-256 alone.


# Appendix C — Authentication Test Procedure

This appendix documents the reproducible procedure used to evaluate PostgreSQL authentication hardening. The procedure was performed only against the authorized local/classroom laboratory systems defined in the project scope.

## C.1 Primary Docker Environment

The primary experiment used the PostgreSQL container from the Week 1 Docker environment. The PostgreSQL service was exposed through the host's published port 5432.

The baseline configuration was recorded before any hardening changes were made. The baseline included verification of:

- PostgreSQL service availability.
- PostgreSQL listening configuration.
- Published port 5432.
- Active host-based authentication rules.
- Authentication mechanism.
- Successful remote authentication using the previously known laboratory credential.
- Client source address observed by PostgreSQL.

## C.2 Baseline Authentication Test

The authorized Kali Linux test client was used to connect to the Docker PostgreSQL service through the Windows host.

The connection was performed using the PostgreSQL client:

    psql -h <authorized-host> -p 5432 -U postgres -d medusa-store

The previously known laboratory credential was supplied interactively and was not recorded in the project repository.

After successful authentication, the following query was used to establish the authenticated identity, database, and source address observed by PostgreSQL:

    SELECT current_user, current_database(), inet_client_addr(), inet_client_port();

The successful connection established the baseline condition against which the hardening changes were evaluated.

## C.3 Baseline Configuration Examination

The PostgreSQL HBA configuration was examined using the PostgreSQL system view:

    SELECT line_number, type, database, user_name, address, auth_method, error
    FROM pg_hba_file_rules
    WHERE type = 'host';

The PostgreSQL listening configuration was verified using:

    SHOW listen_addresses;

Docker port publication was verified using the Docker command:

    docker port medusa-postgres

The PostgreSQL password verifier was also examined to confirm that the authentication mechanism remained SCRAM-SHA-256.

## C.4 Credential Rotation

The PostgreSQL authentication credential was rotated within the authorized laboratory environment.

The credential value itself is intentionally omitted from this appendix and from the project repository.

After rotation, the authentication mechanism was verified again to ensure that the credential change did not result in a weaker authentication method.

## C.5 HBA Restriction

The PostgreSQL `pg_hba.conf` configuration was modified to restrict host authentication according to the source address observed by PostgreSQL.

An initial network restriction based on the original client LAN address produced an HBA denial because Docker networking caused PostgreSQL to observe the connection as originating from the Docker gateway address.

The HBA rule was therefore adjusted to match the verified source address presented by the Docker networking layer.

The configuration was reloaded using PostgreSQL's configuration reload mechanism.

The active HBA rules were then queried again to confirm that the intended rule was loaded without configuration errors.

## C.6 Post-Hardening Authentication Tests

The following conditions were tested after hardening:

1. Connection using the authorized replacement credential.
2. Connection using the previously known credential.
3. Verification that SCRAM-SHA-256 remained active.
4. Verification of the final HBA rule.
5. Verification of the source address observed by PostgreSQL.

The expected result was that the authorized replacement credential would succeed while the previously known credential would fail.

The previously known credential was subsequently tested and PostgreSQL returned a password-authentication failure, demonstrating that credential rotation had invalidated the former credential.

## C.7 Supplementary Direct-LAN Validation

A separate PostgreSQL instance running on Kali Linux was used as a supplementary validation environment.

This test was not a replacement for the primary Docker experiment. Its purpose was to demonstrate PostgreSQL source-address behavior when the database server could observe the client over the local LAN without the Docker networking translation observed in the primary experiment.

The Kali PostgreSQL instance was configured to listen on its authorized LAN address and to permit the laboratory network through an appropriate SCRAM-SHA-256 HBA rule.

The Windows host then connected directly to the Kali PostgreSQL service.

The PostgreSQL query:

    SELECT current_user, current_database(), inet_client_addr(), inet_client_port();

was used to verify the source address observed by the Kali PostgreSQL server.

## C.8 Supplementary HBA Denial Test

The supplementary HBA rule was temporarily restricted to an address that did not include the Windows test client.

A connection attempt was then performed from the Windows host.

PostgreSQL returned a `no pg_hba.conf entry` error, demonstrating that HBA source-address restrictions can deny a connection before successful password authentication.

The supplementary HBA configuration was restored after the test and the PostgreSQL configuration was reloaded.

## C.9 Evidence Collection

Evidence was captured throughout the test sequence and organized into:

- `Evidence/BEFORE/`
- `Evidence/AFTER/`
- `Evidence/Network Validation/`

Each evidence item was assigned an identifier from E-001 through E-016 and mapped to the corresponding test identifier in Appendix A and Appendix B.

Screenshots and logs were reviewed to avoid intentionally retaining authentication secrets in the project artifacts.

## C.10 Test Completion and Cleanup

Testing was considered complete after the baseline, credential-rotation, HBA restriction, authentication, source-address, and supplementary validation conditions had been established.

The supplementary PostgreSQL HBA configuration was restored after the direct-LAN validation.

No testing was performed against systems outside the authorized laboratory environment.


# Appendix D — Hardened PostgreSQL Security Policy

## D.1 Purpose

This policy defines the authentication and network-access controls applied to the PostgreSQL environment used in this project.

The objective is to reduce the risk of unauthorized remote PostgreSQL access when an authentication credential has previously become known to an unauthorized party.

The policy applies to the authorized classroom/laboratory PostgreSQL environments used for this project.

## D.2 Authentication Requirement

PostgreSQL host-based authentication must use `SCRAM-SHA-256` for the remote connections evaluated by this project.

The authentication mechanism must not be weakened to plaintext password authentication or other weaker methods for the purpose of making testing easier.

The project baseline already used `SCRAM-SHA-256`. Therefore, the hardening exercise verifies that the stronger authentication mechanism is retained while additional access-control measures are applied.

## D.3 Credential Rotation

Authentication credentials must be rotated when a credential is known, suspected to be compromised, reused outside its intended purpose, or otherwise no longer considered trustworthy.

The previously known laboratory credential used in the baseline test was replaced during the hardening phase.

Credential values must not be stored in the repository, report, presentation, evidence index, test matrix, or other project documentation.

Credentials should be supplied interactively or through an appropriately protected secret-management mechanism rather than being embedded directly in commands or source files.

## D.4 Host-Based Authentication Rules

PostgreSQL `pg_hba.conf` rules must restrict access according to the intended database, user, connection source, and authentication method.

Broad host rules should be avoided where a narrower source range or specific source address can satisfy the legitimate access requirement.

For the primary Docker experiment, PostgreSQL observed the remote connection through the Docker networking layer rather than seeing the original Kali LAN address. The final laboratory rule therefore used the verified Docker-observed source address and retained `SCRAM-SHA-256`.

This behavior demonstrates that HBA configuration should be based on the actual network path observed by PostgreSQL rather than an assumed client address.

## D.5 Network Exposure

PostgreSQL port 5432 should not be unnecessarily exposed beyond the systems and networks that require access.

Where containerized PostgreSQL requires host-published access for legitimate testing or application functionality, exposure should be limited to the required interface, network, and source addresses where the deployment architecture permits such restriction.

Network exposure and HBA rules should be evaluated together because restricting authentication does not necessarily eliminate the attack surface created by an unnecessarily exposed database service.

## D.6 Configuration Validation

After modifying `pg_hba.conf`, the PostgreSQL configuration must be reloaded and the active rules verified.

The following PostgreSQL system view can be used to inspect the active HBA configuration:

    SELECT line_number, type, database, user_name, address, auth_method, error
    FROM pg_hba_file_rules
    WHERE type = 'host';

The `error` field should be checked to confirm that the intended HBA configuration has been loaded without configuration errors.

## D.7 Authentication Validation

After hardening, authentication must be tested using both:

- The authorized replacement credential.
- The previously known credential.

The expected security condition is:

    Authorized replacement credential → ALLOW
    Previously known credential → DENY

A successful connection with the replacement credential demonstrates that legitimate access remains available.

Failure of the previously known credential demonstrates that credential rotation has removed the former authentication path.

## D.8 Source-Address Validation

The source address observed by PostgreSQL should be verified during testing using:

    SELECT inet_client_addr(), inet_client_port();

This validation is particularly important in containerized environments where NAT, bridge networking, or published ports can cause the database server to observe a different source address from the original client address.

## D.9 Supplementary Direct-LAN Control

The supplementary Kali PostgreSQL environment was used to demonstrate direct-LAN behavior.

In this environment, PostgreSQL observed the Windows test client's actual LAN source address. A temporary restrictive HBA rule was used to demonstrate that a connection from an excluded source address could be denied with a `no pg_hba.conf entry` error.

This supplementary environment is considered a validation/control experiment and does not replace the primary Docker-based experiment.

## D.10 Evidence and Sensitive Information

Evidence must be collected during testing and linked to the corresponding test identifier.

Evidence should not contain:

- Plaintext passwords.
- Private keys.
- API keys.
- Access tokens.
- Unnecessary personal information.
- Credentials belonging to real users or external systems.

Screenshots and logs must be reviewed before being committed to the repository.

Where sensitive information is visible, the evidence should be redacted or replaced with a clean reproduction where possible.

## D.11 Security Boundary

All authentication testing under this policy must remain within the authorized classroom/laboratory environment.

The policy does not authorize:

- Testing third-party PostgreSQL servers.
- Credential attacks against external accounts.
- Brute-force authentication attempts.
- Unauthorized database access.
- Scanning unrelated networks.
- Attempting to obtain credentials from systems outside the project environment.

Testing must stop when the defined project objectives and validation conditions have been satisfied.

## D.12 Expected Security Outcome

The combined control strategy is expected to provide stronger protection than reliance on authentication protocol selection alone.

The intended control relationship is:

    Credential rotation
            +
    SCRAM-SHA-256
            +
    Restrictive pg_hba.conf rules
            +
    Controlled network exposure
            =
    Reduced unauthorized remote PostgreSQL access risk

These controls are complementary. SCRAM-SHA-256 protects the authentication process, credential rotation invalidates a previously known credential, and HBA restrictions limit which connection sources are permitted to authenticate.


