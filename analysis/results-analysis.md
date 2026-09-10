# Results Analysis

## 1. Purpose of the Analysis

This analysis evaluates the results obtained from the PostgreSQL authentication-hardening experiment. The analysis focuses on whether credential rotation, continued use of SCRAM-SHA-256, and restrictive `pg_hba.conf` rules reduced the possibility of unauthorized remote authentication using a previously known credential.

The Docker PostgreSQL environment was treated as the primary experimental environment. The Kali PostgreSQL environment was used only as supplementary direct-LAN validation.

## 2. Baseline Condition

The baseline Docker PostgreSQL environment permitted a remote connection using the previously known laboratory credential.

The PostgreSQL HBA configuration showed a broad host rule using `scram-sha-256`. Therefore, the baseline already used SCRAM-SHA-256 rather than a weaker password authentication method.

The baseline also showed that PostgreSQL was listening on all available addresses and that port 5432 was published by Docker.

These conditions established the initial remote-access path:

    Kali test client
          |
          v
    Windows host :5432
          |
          v
    Docker published port
          |
          v
    PostgreSQL container
          |
          v
    SCRAM-SHA-256 authentication

The baseline therefore demonstrated that the use of SCRAM-SHA-256 did not by itself prevent authentication when the credential remained valid and the HBA configuration permitted the connection.

## 3. Credential Rotation Result

During hardening, the PostgreSQL credential was changed.

The previously known credential was subsequently tested and was rejected by PostgreSQL with a password-authentication failure.

The replacement credential successfully authenticated to the authorized PostgreSQL service.

This provides direct evidence that credential rotation invalidated the previously known authentication secret while preserving legitimate access for the authorized user.

## 4. SCRAM-SHA-256 Result

The PostgreSQL password verifier was checked after credential rotation.

The result continued to indicate `SCRAM-SHA-256`.

This is significant because the hardening process did not require replacing SCRAM-SHA-256 with a weaker mechanism. Instead, the authentication mechanism remained constant while the credential itself was rotated.

Therefore, the experiment evaluates SCRAM-SHA-256 as a retained authentication control rather than as a newly introduced control.

## 5. HBA Restriction Result

The project demonstrated that HBA rules can provide an additional access-control boundary.

An initial restrictive rule based on the Kali client's LAN address produced an HBA denial because PostgreSQL did not observe the original Kali address. Instead, the Docker networking path caused PostgreSQL to observe the connection as originating from the Docker gateway address.

The final HBA configuration was therefore aligned with the source address actually observed by PostgreSQL.

After the adjustment, the authorized replacement credential successfully authenticated.

This result demonstrates that HBA restrictions must account for the actual network architecture. An otherwise correct source-address rule can deny legitimate traffic if the database server observes a translated or intermediate address.

## 6. Previously Known Credential Result

The previously known credential was tested after the final hardening changes.

PostgreSQL returned:

    password authentication failed for user

This result is different from an HBA denial.

The HBA rule had permitted the connection source to reach the authentication stage, but the previously known credential was no longer valid because it had been rotated.

This distinction is important to the interpretation of the experiment:

    HBA restriction
        controls whether the connection source is permitted.

    Credential rotation
        controls whether the previously known credential remains valid.

    SCRAM-SHA-256
        protects the authentication exchange.

These controls therefore address different parts of the authentication and access-control process.

## 7. Supplementary Direct-LAN Result

The supplementary Kali PostgreSQL environment provided a different network path.

In the direct-LAN test, PostgreSQL observed the Windows test client's LAN address rather than a Docker gateway address.

This demonstrated the difference between the source address observed through the Docker networking layer and the source address observed by a PostgreSQL server receiving a direct LAN connection.

The supplementary test therefore supported the interpretation that source-address behavior is dependent on network architecture.

## 8. Supplementary HBA Denial

The direct-LAN PostgreSQL HBA rule was temporarily restricted so that the Windows client's source address was excluded.

The resulting connection attempt was rejected with a `no pg_hba.conf entry` error.

This demonstrated that HBA source-address restrictions can prevent a client from reaching successful authentication when the connection does not satisfy the configured access rule.

## 9. Before-and-After Security State

The overall change can be summarized as follows:

| Security Property | Before Hardening | After Hardening |
|---|---|---|
| Known credential | Valid | Invalidated through rotation |
| Authentication mechanism | SCRAM-SHA-256 | SCRAM-SHA-256 retained |
| Remote PostgreSQL access | Permitted | Restricted according to HBA |
| HBA host rule | Broad | Source-restricted |
| Authorized replacement credential | Not applicable | Successful |
| Previously known credential | Successful | Rejected |
| Docker-observed source | Docker gateway address | Verified and incorporated into HBA |
| Configuration validation | Baseline inspection | Reload and active-rule verification |

## 10. Interpretation

The results support the project hypothesis within the defined laboratory environment.

Credential rotation demonstrated a direct reduction in the risk associated with a previously known credential because the former credential could no longer authenticate.

Restrictive HBA rules provided an additional network-level control by determining which source addresses were permitted to attempt PostgreSQL authentication.

SCRAM-SHA-256 remained enabled throughout the hardening process and therefore provided continuity of the stronger authentication mechanism.

However, the experiment does not establish that SCRAM-SHA-256 alone prevents unauthorized access. The baseline demonstrates the opposite condition: a valid known credential could authenticate while SCRAM-SHA-256 was already enabled.

The strongest interpretation is therefore that PostgreSQL security is improved through layered controls rather than dependence on a single authentication mechanism.

## 11. Observation, Interpretation and Inference

### Observation

The baseline remote connection succeeded using the previously known laboratory credential.

### Interpretation

The credential remained valid and the baseline HBA configuration permitted the connection.

### Observation

After credential rotation, the previously known credential failed authentication.

### Interpretation

Credential rotation invalidated the former credential.

### Observation

An HBA rule based on the original Kali LAN address produced an HBA denial in the Docker environment.

### Interpretation

PostgreSQL observed a different source address because of the Docker networking path.

### Observation

A final HBA rule based on the verified Docker-observed source permitted the authorized connection.

### Interpretation

The HBA rule was aligned with the actual source address presented to PostgreSQL.

### Inference

Combining credential rotation with restrictive HBA rules provides stronger protection against unauthorized remote PostgreSQL access than relying on SCRAM-SHA-256 alone.

## 12. Overall Finding

The experiment indicates that the combination of credential rotation, SCRAM-SHA-256, restrictive HBA rules, and controlled network exposure can substantially reduce the likelihood of successful unauthorized PostgreSQL authentication using a previously known credential.