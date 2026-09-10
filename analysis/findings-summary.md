# Findings Summary

## Primary Finding

The experiment found that a previously known PostgreSQL credential could successfully authenticate to the baseline containerized PostgreSQL service when the credential remained valid and the HBA configuration permitted the connection.

After credential rotation, the previously known credential was rejected while the authorized replacement credential remained usable.

## Finding 1 — SCRAM-SHA-256 Alone Was Not Sufficient

The baseline PostgreSQL configuration already used SCRAM-SHA-256.

Despite this, the previously known credential successfully authenticated.

The result demonstrates that a strong authentication mechanism does not compensate for a credential that is already known to an unauthorized party.

## Finding 2 — Credential Rotation Was Effective

Credential rotation invalidated the previously known credential.

The old credential failed during the post-hardening authentication test, while the replacement credential successfully authenticated.

This directly supports credential rotation as a mitigation for known-credential exploitation.

## Finding 3 — HBA Rules Provided Additional Protection

Restrictive HBA rules successfully denied connections when the observed source address did not satisfy the configured rule.

This demonstrates that HBA provides a network-level authorization control in addition to password authentication.

## Finding 4 — Docker Networking Affected HBA Configuration

The primary Docker experiment showed that PostgreSQL observed the remote connection as originating from the Docker gateway rather than the original Kali LAN address.

An HBA rule based solely on the original client address therefore produced an HBA denial.

The rule had to be aligned with the source address actually observed by PostgreSQL.

## Finding 5 — Direct-LAN Validation Confirmed Source-Address Behavior

The supplementary Kali PostgreSQL test showed that PostgreSQL could observe the Windows host's actual LAN address when the connection occurred directly over the LAN.

This provided useful validation of the difference between containerized and direct-LAN networking behavior.

## Finding 6 — Layered Controls Provided the Strongest Result

The experiment supports a layered security model:

    Strong authentication
            +
    Credential rotation
            +
    Source-based access control
            +
    Controlled network exposure
            =
    Reduced authentication risk

No single control should be treated as sufficient protection against all PostgreSQL access threats.

## Research Question Finding

The research question asked:

> To what extent do SCRAM-SHA-256 authentication and restrictive `pg_hba.conf` rules prevent unauthorized remote PostgreSQL connections using a previously known credential in a containerized environment?

The results indicate that SCRAM-SHA-256 provides a strong authentication mechanism but does not independently prevent use of a valid known credential. Credential rotation directly removes the validity of the previously known credential, while restrictive HBA rules limit which sources are permitted to reach PostgreSQL authentication.

Therefore, the strongest protection observed in the experiment resulted from combining authentication, credential lifecycle, and source-based access controls.

## Hypothesis Finding

The results support the hypothesis within the defined laboratory scope.

The previously known credential was successful under the baseline condition but failed after credential rotation. Restrictive HBA rules also demonstrated the ability to deny connections from excluded source addresses.

The conclusion is limited to the tested PostgreSQL configuration, Docker networking architecture, credentials, source addresses, and authorized classroom environment.