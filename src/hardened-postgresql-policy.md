# Hardened PostgreSQL Authentication Policy

## Purpose

This document defines the intended PostgreSQL authentication-hardening state evaluated by the final project.

The policy is designed to reduce unauthorized remote PostgreSQL authentication using previously known credentials while retaining the SCRAM-SHA-256 authentication mechanism.

## Authentication Requirements

### Password Authentication

PostgreSQL password authentication must use SCRAM-SHA-256.

The PostgreSQL role under test must not continue to accept the previously known baseline credential after credential rotation.

### Credential Rotation

The previously known baseline credential must be replaced with a new authorized lab credential.

Actual credential values must not be stored in this policy document, source repository, report, or other project documentation.

## Host-Based Authentication

Remote PostgreSQL access must be controlled through pg_hba.conf.

The HBA rule must:

1. Specify the intended database and user scope.
2. Restrict the permitted source address or network range.
3. Require scram-sha-256.
4. Avoid unnecessarily broad remote access rules where the deployment architecture permits a more restrictive rule.

The general policy form is:

host all all AUTHORIZED_SOURCE_RANGE scram-sha-256

The actual source range must be selected according to the network architecture of the deployment.

## Docker Networking Consideration

For the primary Docker Desktop environment, PostgreSQL observed the published remote connection source as the Docker bridge gateway rather than the original client address.

The observed Docker source address during testing was DOCKER_GATEWAY.

Therefore, an HBA source-address rule for the containerized service must be based on the address PostgreSQL actually observes in that network path.

This value is specific to the test environment and must not automatically be treated as a universal production configuration.

## Supplementary Direct-LAN Validation

A separate PostgreSQL instance on Kali Linux was used to validate source-address behavior without the Docker Desktop networking layer.

In the direct-LAN test:

- Kali PostgreSQL address: KALI_HOST
- Windows client address observed by PostgreSQL: WINDOWS_HOST
- Authentication method: scram-sha-256

The direct-LAN HBA test temporarily restricted access to KALI_HOST/32.

The Windows client was then denied because its source address was outside the permitted address.

The temporary restriction was subsequently restored to AUTHORIZED_LAN_RANGE.

## Validation Requirements

A hardened configuration should demonstrate:

- Successful authentication using the authorized new credential.
- Rejection of the previous credential.
- Continued use of SCRAM-SHA-256.
- Enforcement of the intended HBA source-address restriction.
- Successful PostgreSQL configuration reload.
- No HBA configuration errors.

## Security Boundary

This policy applies only to the authorized project test environment.

It is not intended to prescribe a universal production PostgreSQL configuration.

Production source ranges, database scopes, roles, TLS requirements, network segmentation, and other controls must be determined according to the actual deployment architecture and security requirements.

## Evidence

Validation evidence is stored under:

Evidence/BEFORE/
Evidence/AFTER/
Evidence/Network Validation/

The evidence index is Evidence/evidence-index.md.

The test matrix is Evidence/test-matrix.md.