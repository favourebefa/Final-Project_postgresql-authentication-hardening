# Before-and-After Comparison

## Experimental Comparison

| Control Area | Baseline | Hardened State | Security Effect |
|---|---|---|---|
| Credential | Previously known credential remained valid | Credential rotated | Previously known credential could no longer authenticate |
| Authentication | SCRAM-SHA-256 | SCRAM-SHA-256 retained | Strong authentication mechanism preserved |
| HBA | Broad host rule | Source-restricted rule | Remote access scope reduced |
| PostgreSQL listening | `*` | Verified existing listening configuration | Service remained reachable through the intended test path |
| Docker port | 5432 published | 5432 remained published for authorized testing | Port exposure remained available but access was controlled through authentication and HBA |
| Source address | PostgreSQL observed Docker gateway | Verified Docker-observed source incorporated into HBA | Prevented incorrect source-based rule configuration |
| Authorized access | Successful using known credential | Successful using replacement credential | Legitimate access preserved |
| Previously known credential | Successful | Authentication failed | Credential compromise scenario mitigated |
| HBA mismatch | Not tested as a baseline denial condition | Denial demonstrated | Network-level restriction validated |

## Key Security Change

The most important change was not the introduction of SCRAM-SHA-256 because SCRAM-SHA-256 was already present in the baseline configuration.

The primary security improvement resulted from changing the known credential and restricting the set of source addresses permitted by the HBA configuration.

The authentication mechanism was retained as a control while credential validity and network authorization were strengthened.

## Security Control Relationship

    SCRAM-SHA-256
        |
        +-- Protects authentication exchange

    Credential rotation
        |
        +-- Invalidates previously known credential

    pg_hba.conf restriction
        |
        +-- Limits permitted connection sources

    Controlled port exposure
        |
        +-- Limits reachable service surface

Together these controls provide layered protection.