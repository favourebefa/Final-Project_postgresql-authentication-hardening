# Evidence Index

## Evidence Handling Note

All evidence was collected from the authorized local/classroom test environment. The PostgreSQL identities, credentials, database names, network addresses, and test conditions used for this project are fictional or lab-only values. Screenshots were reviewed for unnecessary sensitive information before inclusion.

The primary experiment evaluates PostgreSQL authentication hardening in the containerized Docker stack. The Network Validation evidence is supplementary and was collected from a separate direct-LAN PostgreSQL test on Kali Linux to validate source-IP visibility and HBA enforcement without Docker Desktop networking/NAT affecting the observed client address.

## Evidence Register

| Evidence ID | File / Source | Test / Event | Date / Time | Description | Redaction Status |
|---|---|---|---|---|---|
| E-001 | `BEFORE/Current_PostgreSQL_HBA_Configuration.png` | T-002 | 2026-09-08 05:57 | Captures the PostgreSQL HBA configuration before hardening, including the remote `host all all all scram-sha-256` rule. | Reviewed; no unnecessary sensitive data |
| E-002 | `BEFORE/Listen_Addresses.png` | T-003 | 2026-09-08 05:59 | Shows the PostgreSQL `listen_addresses` configuration before hardening. | Reviewed; no unnecessary sensitive data |
| E-003 | `BEFORE/PostgreSQL_Expose_Port5432.png` | T-004 | 2026-09-08 06:02 | Shows Docker publishing PostgreSQL port 5432 to the host. | Reviewed; no unnecessary sensitive data |
| E-004 | `BEFORE/Valid_Baselne_Postgresql.png` | T-001 | 2026-09-08 06:07 | Baseline PostgreSQL connection/authentication evidence before the credential hardening change. | Reviewed; no unnecessary sensitive data |
| E-005 | `BEFORE/Remote_Access_Evidence.png` | T-001 | 2026-09-08 06:12 | Demonstrates successful remote access to the containerized PostgreSQL service using the known baseline credential. | Reviewed; no unnecessary sensitive data |
| E-006 | `AFTER/Password_Format_Check.png` | T-006 | 2026-09-08 06:37 | Verifies the PostgreSQL password verifier remained `SCRAM-SHA-256` after the credential change. | Reviewed; no password value included |
| E-007 | `AFTER/Postgres_Port.png` | T-004 | 2026-09-08 06:39 | Captures the PostgreSQL service port configuration after hardening. | Reviewed; no unnecessary sensitive data |
| E-008 | `AFTER/pg_reload_conf.png` | T-007 | 2026-09-08 06:49 | Shows successful PostgreSQL configuration reload after the HBA modification. | Reviewed; no unnecessary sensitive data |
| E-009 | `AFTER/HBA_Restriction_Rule_Set.png` | T-008 | 2026-09-08 06:53 | Captures the restrictive HBA rule configured during the hardening test. | Reviewed; no unnecessary sensitive data |
| E-010 | `AFTER/Active_HBA_Rule.png` | T-009 | 2026-09-08 06:56 | Shows the active HBA rule after the hardening configuration was applied. | Reviewed; no unnecessary sensitive data |
| E-011 | `AFTER/Current_State.png` | T-010 | 2026-09-08 07:18 | Captures the resulting PostgreSQL state after the main hardening procedure. | Reviewed; no unnecessary sensitive data |
| E-012 | `AFTER/Active_HBA_Rule_2.png` | T-011 | 2026-09-08 09:04 | Additional verification of the active HBA configuration and absence of configuration errors. | Reviewed; no unnecessary sensitive data |
| E-013 | `Network Validation/Windows_to_Kali_Source_IP.png` | T-012 | 2026-09-08 09:33 | Demonstrates successful direct-LAN Windows-to-Kali PostgreSQL access and confirms PostgreSQL observed the Windows client as `WINDOWS_HOST`. | Reviewed; lab network address only |
| E-014 | `Network Validation/Windows_to_Kali_Old_Password_Rejected.png` | T-013 | 2026-09-08 09:41 | Demonstrates rejection of the previous known PostgreSQL credential during direct-LAN authentication. | Reviewed; password value not retained in evidence |
| E-015 | `Network Validation/Windows_to_Kali_HBA_Denied.png` | T-014 | 2026-09-08 10:49 | Demonstrates HBA denial of the Windows client when the temporary rule was restricted to the Kali host address. | Reviewed; lab network address only |

## Evidence Classification

### Primary Experiment - Containerized PostgreSQL

Evidence E-001 through E-012 support examination of the Docker PostgreSQL environment before and after authentication hardening. These records cover the baseline configuration, remote exposure, SCRAM verification, HBA configuration, PostgreSQL configuration reload, and final PostgreSQL state.

The screenshot showing the actual credential replacement command was intentionally excluded from the repository because it contained sensitive authentication material. The credential change itself is documented in the test procedure and report without reproducing the secret.

### Supplementary Network Validation

Evidence E-013 through E-015 supports a separate direct-LAN validation using PostgreSQL on Kali Linux. This supplementary test was used to establish that a direct PostgreSQL connection preserves the actual Windows source IP and to demonstrate that `pg_hba.conf` can deny an otherwise reachable client based on source address.

## Evidence Integrity

Evidence files were collected during testing and retained in the project evidence directory. File names identify the test context and distinguish baseline, post-hardening, and supplementary network-validation evidence.

SHA-256 hashes for the retained evidence files are stored in:

```text
Evidence/evidence-hashes.txt