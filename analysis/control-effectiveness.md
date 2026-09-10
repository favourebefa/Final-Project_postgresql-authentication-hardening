# Control Effectiveness Assessment

## 1. Purpose

This assessment evaluates the effectiveness of the security controls tested during the PostgreSQL authentication-hardening project.

The assessment considers the observed results rather than assuming that a control is effective simply because it was configured.

## 2. Control Assessment

| Control | Intended Function | Evidence of Operation | Effectiveness | Assessment |
|---|---|---|---|---|
| SCRAM-SHA-256 | Provide stronger password authentication than weaker legacy methods | SCRAM-SHA-256 was verified before and after hardening | Effective as an authentication mechanism | The control remained active throughout the experiment |
| Credential rotation | Invalidate a previously known credential | Previously known credential failed after rotation | Effective | Directly mitigated the tested known-credential condition |
| Restrictive `pg_hba.conf` | Limit which connection sources can authenticate | HBA mismatch produced a `no pg_hba.conf entry` denial | Effective when correctly configured | Provides an additional source-based access-control layer |
| Source-address validation | Identify the address PostgreSQL actually sees | `inet_client_addr()` was used during testing | Effective | Prevented incorrect assumptions about the Docker network path |
| Configuration reload | Apply updated PostgreSQL access-control rules | PostgreSQL configuration reload completed successfully | Effective | Required for the HBA changes to become active |
| Port exposure control | Reduce unnecessary network exposure | Port 5432 exposure was examined | Partially addressed | The project validated exposure but did not redesign the complete Docker port-publishing architecture |
| Evidence handling | Preserve traceability while protecting sensitive information | Evidence index and test matrix were created | Effective within project scope | Evidence was mapped to individual tests and sensitive credential values were excluded |

## 3. SCRAM-SHA-256 Assessment

SCRAM-SHA-256 was effective as the authentication mechanism used by the tested PostgreSQL connections.

However, the experiment demonstrates an important limitation: SCRAM-SHA-256 does not make a valid credential unusable.

The baseline connection succeeded while SCRAM-SHA-256 was already active. Therefore, an attacker who possesses a valid credential may still authenticate if the HBA configuration permits the connection.

SCRAM-SHA-256 should consequently be considered one layer within a broader authentication-control strategy.

## 4. Credential Rotation Assessment

Credential rotation was directly effective against the tested threat condition.

The previously known credential successfully authenticated during the baseline test but failed after the credential was changed.

This establishes a clear before-and-after security effect:

    Before rotation:
    Previously known credential → Successful authentication

    After rotation:
    Previously known credential → Authentication failure

Credential rotation therefore removed the specific authentication path represented by the previously known credential.

## 5. HBA Assessment

The HBA control was effective in restricting connection sources.

The supplementary direct-LAN experiment demonstrated that a client whose source address did not match the configured HBA rule received a `no pg_hba.conf entry` denial.

The primary Docker experiment also demonstrated that HBA configuration must account for the address actually observed by PostgreSQL.

This means HBA restrictions can be highly effective, but incorrect assumptions about network translation can unintentionally block legitimate users or services.

## 6. Layered Control Assessment

The controls tested in the project address different security properties:

| Layer | Control | Primary Security Property |
|---|---|---|
| Authentication protocol | SCRAM-SHA-256 | Protect authentication exchange |
| Credential lifecycle | Credential rotation | Invalidate compromised/known credentials |
| Network authorization | `pg_hba.conf` | Restrict permitted connection sources |
| Service exposure | Port/network configuration | Reduce reachable attack surface |
| Validation | Configuration and connection testing | Confirm intended security state |

The controls are therefore complementary rather than interchangeable.

## 7. Overall Effectiveness Rating

The overall control strategy is assessed as **effective within the tested laboratory scenario**.

This assessment does not mean that PostgreSQL is immune to unauthorized access. Instead, it indicates that the specific threat condition tested by the project—remote authentication using a previously known credential—was successfully mitigated by the combination of credential rotation and access-control restrictions.

The experiment also identified an implementation consideration: container networking can alter the source address observed by PostgreSQL. HBA restrictions must therefore be validated against the actual network path.

## 8. Residual Risk

Residual risk remains if:

- New credentials are subsequently compromised.
- HBA rules are incorrectly configured.
- PostgreSQL is unnecessarily exposed to untrusted networks.
- Credentials are reused across environments.
- Administrative privileges are broader than required.
- Network controls outside PostgreSQL permit unnecessary access to port 5432.
- Configuration changes are made without subsequent validation.

The tested controls reduce risk but do not eliminate the need for ongoing configuration management and monitoring.