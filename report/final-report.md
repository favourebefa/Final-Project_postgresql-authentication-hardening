# List of Tables

| Table | Title |
|---|---|
| Table 1 | Primary Docker Environment |
| Table 2 | Experimental Variables |
| Table 3 | Test Sequence |
| Table 4 | Before-and-After Comparison |
| Table 5 | Test Matrix Results |
| Table 6 | Risk Register |
| Table 7 | Control Effectiveness Assessment |
| Table 8 | Recommended Improvement Roadmap |
| Table 9 | Primary and Supplementary Network Comparison |

# List of Figures

| Figure | Title |
|---|---|
| Figure 1 | Baseline PostgreSQL HBA Configuration |
| Figure 2 | Baseline PostgreSQL Listening Configuration |
| Figure 3 | PostgreSQL Port Exposure |
| Figure 4 | Baseline PostgreSQL Authentication Verification |
| Figure 5 | Baseline Remote PostgreSQL Access |
| Figure 6 | Credential Rotation |
| Figure 7 | Post-Hardening SCRAM-SHA-256 Verification |
| Figure 8 | PostgreSQL Port After Hardening |
| Figure 9 | PostgreSQL Configuration Reload |
| Figure 10 | Initial HBA Restriction |
| Figure 11 | Active HBA Rule |
| Figure 12 | Authorized Post-Hardening Connection |
| Figure 13 | Final Active HBA Rule |
| Figure 14 | Direct-LAN Source Address Validation |
| Figure 15 | Direct-LAN Previously Known Credential Rejection |
| Figure 16 | Direct-LAN HBA Denial |

# Chapter 2: Background and Related Work

## 2.1 PostgreSQL Authentication

PostgreSQL provides authentication controls that determine whether a client is permitted to establish a database connection. Authentication is particularly important when a PostgreSQL service is reachable over a network because a reachable service may receive connection attempts from clients outside the database host.

PostgreSQL authentication is controlled in part through the Host-Based Authentication (HBA) configuration. The HBA configuration specifies the databases and users to which a rule applies, the client addresses covered by the rule, and the authentication method that PostgreSQL should use.

Authentication should therefore be considered as part of a broader access-control process. A client must first satisfy the applicable connection policy before successful authentication can result in a database session.

## 2.2 Host-Based Authentication and `pg_hba.conf`

PostgreSQL uses the `pg_hba.conf` file to define host-based access-control rules. Each rule can specify a connection type, database, user, client address or address range, and authentication method.

For example, a host rule can conceptually be represented as:

`host database user address authentication-method`

This structure allows database administrators to restrict connections according to the source network while also specifying how credentials must be authenticated.

An important characteristic of HBA processing is that PostgreSQL evaluates the applicable rules for a connection. Consequently, the placement and specificity of rules are important when implementing access restrictions.

In this project, HBA rules were examined before and after hardening. The baseline container configuration included a general host rule allowing connections from all addresses while requiring SCRAM-SHA-256. The hardening process replaced this broad source-address allowance with a more restrictive rule based on the source address observed by PostgreSQL.

The experiment demonstrated that the address observed by PostgreSQL was significant to the effectiveness of the HBA restriction. When the external Kali Linux client connected through the Docker Desktop published port, PostgreSQL observed the Docker bridge gateway address rather than the original Kali LAN address.

## 2.3 SCRAM-SHA-256 Authentication

SCRAM-SHA-256 is a password authentication mechanism supported by PostgreSQL. It provides a stronger authentication mechanism than older password authentication approaches and is designed to avoid sending the user's password directly to the server during authentication.

The use of SCRAM-SHA-256 does not, however, eliminate the need for secure credential management. If an attacker knows a valid password, the authentication mechanism can still correctly authenticate that attacker as the associated database user.

This distinction is important to the present research. The baseline PostgreSQL instance was already using SCRAM-SHA-256. Therefore, the experiment does not evaluate a transition from a weak authentication mechanism to SCRAM-SHA-256. Instead, it evaluates whether credential rotation and restrictive HBA source controls improve security while retaining SCRAM-SHA-256.

The PostgreSQL verifier was also checked during the experiment to confirm that the `postgres` role continued to use SCRAM-SHA-256 after the credential change.

## 2.4 Credential Security and Default Credentials

Credentials that are known, predictable, reused, or distributed beyond their intended audience can create a significant authentication risk. A database service may have technically strong authentication while still being vulnerable to unauthorized access if an attacker possesses a valid credential.

The baseline phase of this project deliberately used a previously known laboratory credential to establish whether the exposed PostgreSQL service could be accessed remotely. The successful connection demonstrated that network reachability combined with knowledge of the valid credential was sufficient to establish a PostgreSQL session.

The hardening phase changed the known credential. A subsequent attempt using the previously known credential was rejected, demonstrating the security value of credential rotation in the tested scenario.

The experiment therefore treats credential security and authentication protocol strength as separate but complementary controls.

## 2.5 Network Exposure of PostgreSQL

PostgreSQL can be configured to listen on network interfaces rather than only on the local loopback interface. A service listening on network interfaces can accept connections from remote clients when the network path and PostgreSQL authentication rules permit them.

The baseline Docker configuration published PostgreSQL port 5432 from the container to the host. The PostgreSQL server was also configured to listen on all available addresses.

This combination made the database service reachable from the authorized Kali Linux test environment through the Windows host's LAN address. The baseline remote connection test confirmed that a client could establish a PostgreSQL session using the known laboratory credential.

Network exposure is therefore an important part of the threat model considered in this project. Authentication controls are more important when a database service is reachable beyond its local host, but network exposure should ideally also be minimized through appropriate firewall, container-network, and service-binding controls.

## 2.6 Containerized PostgreSQL

Containerization packages an application and its dependencies into an isolated runtime environment. Docker is commonly used to deploy multi-service application stacks in which components such as application servers, databases, and caches run in separate containers.

The primary environment used in this study is based on a Docker stack containing PostgreSQL and Redis. PostgreSQL 15 runs in the `medusa-postgres` container and exposes port 5432 through Docker Desktop.

Container networking introduces an additional consideration when applying source-address restrictions. The original client may connect to the host's published port, after which Docker networking can translate or route the connection before it reaches the database container.

Consequently, an HBA rule based only on the external client's IP address may not behave as expected if PostgreSQL observes a different source address.

## 2.7 Docker Desktop Networking and Source Address Translation

The primary experiment was conducted using Docker Desktop on a Windows host. The PostgreSQL container was attached to a Docker bridge network.

During the baseline remote connection, the external Kali Linux client used the Windows host LAN address to reach the published PostgreSQL port. However, PostgreSQL reported the connecting client address as the Docker bridge gateway address.

This observation affected the implementation of the HBA restriction. An initial attempt to permit the Kali LAN subnet was not sufficient because PostgreSQL was not seeing the Kali LAN address as the connection source. The resulting connection attempt was rejected with a `no pg_hba.conf entry` error.

The HBA rule was subsequently adjusted to the source address observed by PostgreSQL. After this change, the authorized client could authenticate using the newly rotated credential, while the previously known credential was rejected as an authentication failure.

This behaviour demonstrates why access-control policies should be based on the actual network topology and the source address visible to the service enforcing the policy.

## 2.8 Authentication and Authorization as Layered Controls

Authentication and authorization address different security questions.

Authentication determines whether a presented identity can be verified. In this project, the PostgreSQL password and SCRAM-SHA-256 mechanism form part of the authentication process.

Authorization determines what an authenticated identity is permitted to do. PostgreSQL roles and database privileges provide authorization mechanisms, although detailed privilege escalation and role-permission testing are outside the scope of this project.

HBA rules operate at an earlier access-control stage by determining whether a connection from a particular source can use a particular authentication method for a specified database and user.

The project therefore evaluates a layered control model:

1. **Network reachability** — Can the client reach the PostgreSQL service?
2. **HBA policy** — Is the client's observed source address permitted?
3. **Authentication** — Does the client possess the required valid credential?
4. **Database authorization** — What can the authenticated role access or modify?

The experiment focuses primarily on the second and third layers while documenting the first as part of the system examination.

## 2.9 Related Security Principle: Defense in Depth

Defense in depth is the practice of applying multiple complementary security controls so that the failure or compromise of one control does not automatically result in complete system compromise.

The PostgreSQL configuration examined in this project illustrates this principle. SCRAM-SHA-256 strengthens password authentication, credential rotation invalidates a previously known credential, and restrictive HBA rules reduce the set of client sources that can authenticate.

These controls address different parts of the attack path. A strong authentication protocol cannot compensate for an exposed service combined with a known valid password, while a source restriction may reduce attack opportunities even when a credential is compromised.

The experiment therefore evaluates the combined effect of authentication and source-based access restrictions rather than treating SCRAM-SHA-256 as a standalone security solution.

## 2.10 Relevance to the Research Question

The background concepts described in this chapter directly support the research question.

The baseline test establishes whether a remotely reachable PostgreSQL service can be accessed using a known credential. The hardening phase then changes the credential and restricts the applicable HBA source address while retaining SCRAM-SHA-256.

The comparison between these states allows the study to determine whether the implemented controls prevent the previously successful unauthorized connection scenario.

The Docker networking observation also provides an important qualification: the effectiveness of an HBA source restriction depends on the address PostgreSQL actually observes. Therefore, authentication hardening must be evaluated within the specific network architecture in which the database is deployed.

## 2.11 Chapter Summary

This chapter established the technical background required to understand the experiment. PostgreSQL authentication, HBA rules, SCRAM-SHA-256, credential security, network exposure, and Docker networking were identified as the principal concepts relevant to the study.

The key distinction established is that strong authentication does not by itself prevent access using a valid known credential. Restrictive source-address controls and credential rotation provide additional security layers.

The next chapter defines the threat model, assets, risks, trust boundaries, and security controls used to evaluate the PostgreSQL configuration.

# Chapter 3: Threat Model, Risk and Control Context

## 3.1 Threat Model Overview

The threat model for this project focuses on an unauthorized remote client attempting to access a PostgreSQL database using a previously known credential. The scenario assumes that the attacker can reach the PostgreSQL service over the network and possesses knowledge of a credential that was valid before hardening.

The threat model is intentionally limited to the authorized laboratory environment. It does not model attacks against Internet-facing production systems or systems without explicit authorization.

The central security question is whether the combination of credential rotation, SCRAM-SHA-256 authentication, and restrictive Host-Based Authentication (HBA) rules can prevent the previously successful remote authentication scenario.

The simplified attack path is:

**Remote client → Network reachability → Published PostgreSQL port → HBA evaluation → Password authentication → PostgreSQL session**

The experiment evaluates how the security controls affect this path before and after hardening.

## 3.2 Protected Assets

The principal asset in this investigation is the PostgreSQL database service and the data accessible through the authenticated PostgreSQL role.

The relevant assets include:

- PostgreSQL database service.
- Database authentication credentials.
- PostgreSQL roles and identities.
- Database sessions established by authenticated clients.
- PostgreSQL configuration controlling remote access.
- Docker network path exposing PostgreSQL.
- Integrity and confidentiality of the database environment.

Although the laboratory environment uses fictional or test data, unauthorized database access would still represent a security failure because it would demonstrate that an untrusted client can cross the intended authentication boundary.

## 3.3 Threat Actor

The modeled threat actor is an unauthorized remote client with the following capabilities:

- Ability to reach the PostgreSQL service through the available network path.
- Knowledge of a previously valid database credential.
- Ability to initiate PostgreSQL connection attempts.
- Ability to observe whether authentication succeeds or fails.

The threat actor is not assumed to have administrator privileges on the Windows host, Docker environment, or PostgreSQL server.

The experiment does not attempt to determine how the credential was originally obtained. Credential compromise is treated as an assumed condition so that the effectiveness of the authentication controls can be evaluated.

## 3.4 Threat Scenario

The primary threat scenario consists of a remote client attempting to authenticate to PostgreSQL using a known credential.

During the baseline phase, the PostgreSQL service was reachable through the Windows host's published port 5432. The known laboratory credential successfully authenticated from the authorized Kali Linux test environment.

This established the baseline condition required for the experiment.

During the hardening phase, the known credential was replaced with a new laboratory credential. The PostgreSQL verifier continued to use SCRAM-SHA-256, and the HBA configuration was restricted according to the source address observed by PostgreSQL.

A subsequent authentication attempt using the previously known credential failed. A separate HBA restriction test also demonstrated that a client source not permitted by the active HBA rule was denied before successful authentication.

## 3.5 Trust Boundaries

Several trust boundaries are relevant to the experiment.

### 3.5.1 External Client to Windows Host

The first boundary exists between the remote test client and the Windows host. The client connects to the Windows host's LAN address on TCP port 5432.

The host therefore represents the network entry point through which the PostgreSQL service is exposed.

### 3.5.2 Windows Host to Docker Networking

The second boundary exists between the Windows host's published port and the Docker networking environment.

Docker Desktop forwards the published PostgreSQL port toward the PostgreSQL container. The networking path affects the source address visible to the database.

### 3.5.3 Docker Network to PostgreSQL Container

The third boundary exists between the Docker bridge network and the PostgreSQL container.

PostgreSQL evaluates incoming connections using its HBA configuration. The source address observed at this boundary is therefore important when implementing source-based access restrictions.

### 3.5.4 PostgreSQL Authentication Boundary

The final relevant boundary is the PostgreSQL authentication boundary. A connection that reaches PostgreSQL must satisfy the applicable HBA rule and successfully complete the configured authentication mechanism.

In this experiment, SCRAM-SHA-256 was the required password authentication mechanism for the tested remote host rule.

## 3.6 Security Risks

The experiment addresses several related security risks.

### Risk 1: Unauthorized Access Using a Known Credential

If a previously known PostgreSQL credential remains valid, an unauthorized client that can reach the database service may be able to authenticate successfully.

This risk was demonstrated during the baseline test when the known laboratory credential successfully established a remote PostgreSQL session.

### Risk 2: Excessively Broad HBA Source Rules

An HBA rule that permits all source addresses increases the range of clients that can reach the PostgreSQL authentication boundary.

The baseline host rule permitted connections from all addresses while requiring SCRAM-SHA-256. Although SCRAM provided password authentication protection, the source restriction was broad.

### Risk 3: Network Exposure of PostgreSQL

Publishing PostgreSQL port 5432 makes the database service reachable through the host network interface. If network-level restrictions are insufficient, unauthorized clients may be able to reach the PostgreSQL authentication service.

The project therefore treats service exposure as part of the overall attack surface.

### Risk 4: Incorrect Source Address Assumptions

A source-based HBA rule can be ineffective if it is written for an IP address that PostgreSQL does not actually observe.

The experiment demonstrated this risk through Docker Desktop networking. The Kali Linux client used the Windows host LAN address, but PostgreSQL observed the Docker bridge gateway address as the connection source.

An HBA rule written only for the original Kali LAN subnet therefore produced an HBA denial rather than allowing the intended authenticated connection.

## 3.7 Security Controls Evaluated

The project evaluates the following controls.

| Control | Purpose | Evaluation |
|---|---|---|
| Credential rotation | Invalidates the previously known credential | Tested before and after hardening |
| SCRAM-SHA-256 | Provides password authentication using the configured PostgreSQL mechanism | Verified before and after hardening |
| Restrictive HBA rule | Limits which observed client source can use the PostgreSQL host rule | Tested using permitted and denied source conditions |
| Network exposure examination | Determines whether PostgreSQL is reachable remotely | Tested through the published port |
| Source-IP verification | Determines which address PostgreSQL actually sees | Verified through `inet_client_addr()` |
| Configuration reload | Applies the updated HBA configuration | Verified after modification |

## 3.8 Control Relationships

The controls operate at different stages of the connection process.

The network configuration determines whether a client can reach the PostgreSQL service. Once a connection reaches PostgreSQL, the HBA configuration determines whether the observed source address is covered by an applicable rule.

If the connection is permitted by HBA, SCRAM-SHA-256 authentication then requires the client to provide the valid credential.

Credential rotation changes which credential is considered valid. Therefore, even if a client can reach the authentication service and is covered by the HBA rule, possession of the previously known credential should no longer be sufficient to authenticate.

The controls can therefore be represented as a layered sequence:

**Network reachability → HBA source restriction → SCRAM-SHA-256 authentication → Database session**

This layered model is central to the project's evaluation.

## 3.9 Baseline Security State

The baseline security state was established before making the hardening changes.

The PostgreSQL container was reachable through the host's published TCP port 5432. PostgreSQL was configured to listen on all addresses, and the general host HBA rule permitted connections from all addresses while requiring SCRAM-SHA-256.

The baseline remote test from Kali Linux successfully authenticated using the known laboratory credential. PostgreSQL reported the client address as the Docker bridge gateway rather than the original Kali LAN address.

This baseline established that the combination of network reachability and possession of the known credential was sufficient to establish a PostgreSQL session.

## 3.10 Hardened Security State

The hardened state introduced two principal changes.

First, the previously known PostgreSQL credential was replaced with a new laboratory credential. This was intended to invalidate authentication attempts using the old credential.

Second, the broad HBA source rule was replaced with a restrictive rule based on the source address actually observed by PostgreSQL in the Docker networking environment.

SCRAM-SHA-256 remained enabled after these changes.

The resulting configuration was then tested using both the new and previously known credentials. The new credential successfully authenticated through the permitted path, while the previous credential was rejected.

A supplementary direct-LAN test also demonstrated that HBA can deny a client based on its actual LAN source address when the database directly observes that address.

## 3.11 Risk Treatment

The project's approach to risk treatment is primarily **risk reduction**.

The known-credential risk is reduced by rotating the credential.

The broad network-source risk is reduced by restricting the HBA source address.

The authentication mechanism is retained as SCRAM-SHA-256 rather than weakened to an older authentication method.

The experiment does not claim that these controls eliminate all PostgreSQL security risks. Instead, they address the specific threat scenario defined by the research question.

## 3.12 Residual Risk

Even after hardening, residual risks remain.

A client that is legitimately within the permitted source boundary and possesses the current valid credential may still authenticate successfully.

Similarly, if an attacker compromises a trusted host, gains access to the permitted network path, or obtains the current credential, HBA source restrictions alone may not prevent authentication.

Docker networking can also change the source address observed by PostgreSQL. Consequently, future changes to the container networking architecture may require corresponding changes to the HBA policy.

Additional controls such as host firewalls, network segmentation, TLS configuration, least-privilege database roles, credential-management procedures, and monitoring may therefore be required for a production deployment.

These controls are outside the main experimental scope but represent important areas for future security improvement.

## 3.13 Threat Model Assumptions

The following assumptions apply to the experiment:

1. The PostgreSQL container and Windows host are authorized laboratory systems.
2. The test client is authorized to connect to the laboratory environment.
3. The baseline credential is treated as known to the modeled unauthorized client.
4. The attacker does not have administrative access to the Docker host.
5. The attacker does not modify the PostgreSQL configuration directly.
6. The network path used during testing remains available.
7. Test observations accurately represent the configuration at the time each test was performed.
8. The Docker networking behaviour observed during the experiment is specific to the tested Docker Desktop environment.

## 3.14 Chapter Summary

This chapter defined the threat model and security context for the project.

The primary threat is unauthorized remote PostgreSQL authentication using a previously known credential. The main security controls evaluated are credential rotation, SCRAM-SHA-256 authentication, restrictive HBA source rules, and verification of the network source observed by PostgreSQL.

The threat model also identified Docker Desktop networking as an important architectural factor because the database observed the Docker bridge gateway address rather than the original client LAN address.

The next chapter presents the methodology used to conduct the experiment, including the test environment, tools, variables, procedure, evidence handling, and ethical safeguards.

# Chapter 4: Methodology and Ethical Scope

## 4.1 Research Design

This project uses a controlled experimental design to evaluate PostgreSQL authentication hardening in a containerized environment.

The experiment follows a before-and-after approach. The baseline configuration is first examined and tested to establish the initial security state. Authentication hardening is then applied, after which the same relevant connection conditions are tested again.

The comparison focuses on observable authentication outcomes rather than theoretical assumptions. The principal outcome is whether a remote client using the previously known credential can establish a PostgreSQL session before and after hardening.

The experiment also examines whether HBA source restrictions operate as intended and whether the source address observed by PostgreSQL matches the address assumed when designing the access-control rule.

## 4.2 Research Environment

The primary experiment was conducted on an authorized local Docker Desktop environment running on a Windows host.

The main PostgreSQL service was:

- Container name: `medusa-postgres`
- PostgreSQL version: 15.x
- Container image: `postgres:15-alpine`
- Database: laboratory `medusa-store` database
- Published PostgreSQL port: TCP 5432
- Docker network: `week1_medusa-net`
- Network type: Docker bridge network

The Docker stack also contained Redis as a supporting service. Redis was not the subject of the authentication experiment.

The authorized test client was a Kali Linux system connected to the same local laboratory network.

The primary network path was:

**Kali Linux → Windows host LAN address → Docker published port 5432 → Docker network → PostgreSQL container**

A supplementary direct-LAN PostgreSQL test was conducted on the Kali Linux system. This secondary test was used to validate source-address-based HBA behaviour when PostgreSQL directly observed the client's LAN address.

## 4.3 Authorization and Ethical Scope

All testing was performed within an authorized local/classroom environment.

The systems used for testing were controlled laboratory systems, and the database credentials used during testing were laboratory credentials. No attempt was made to access an unrelated external system.

The experiment was designed to avoid unnecessary disruption. Testing consisted of controlled connection attempts, configuration inspection, authentication validation, and HBA access-control tests.

The project did not involve:

- Internet-wide scanning.
- Unauthorized system access.
- Password cracking.
- Credential harvesting.
- Denial-of-service activity.
- Exploitation of production systems.
- Collection of real user credentials.
- Collection of unnecessary personal information.

The project therefore remained within the authorized defensive-security testing boundary.

## 4.4 Tools and Technologies

The following tools and technologies were used:

| Tool / Technology | Purpose |
|---|---|
| Docker Desktop | Container runtime and container networking |
| PostgreSQL 15 | Primary database system under investigation |
| PostgreSQL `psql` | Database connection and configuration testing |
| Kali Linux | Authorized remote test client |
| Windows CMD / PowerShell | Docker administration and network validation |
| `Test-NetConnection` | TCP reachability validation for the supplementary test |
| PostgreSQL configuration tools | Authentication and HBA configuration inspection |
| `pg_isready` | PostgreSQL service health validation |
| `inet_client_addr()` | Verification of the source address observed by PostgreSQL |
| VS Code | Project documentation and repository editing |
| Git | Version control and project artifact management |

Tool versions were recorded where relevant to reproducibility. The primary database environment used PostgreSQL 15 through the `postgres:15-alpine` container image.

## 4.5 Experimental Variables

The experiment contains independent, dependent, and controlled variables.

### 4.5.1 Independent Variables

The principal independent variables are:

1. PostgreSQL credential state.
2. HBA source-address restriction.
3. Authentication configuration.

The credential state changes from the previously known laboratory credential to a new laboratory credential.

The HBA source restriction changes from a broad host rule to a rule based on the source address observed by PostgreSQL.

SCRAM-SHA-256 is retained as the authentication mechanism.

### 4.5.2 Dependent Variable

The primary dependent variable is the outcome of the PostgreSQL connection attempt.

Possible outcomes include:

- Successful PostgreSQL authentication.
- Password authentication failure.
- HBA rejection due to absence of an applicable rule.

The observed source address reported by PostgreSQL is also treated as an important measurement because it determines how the HBA restriction must be interpreted.

### 4.5.3 Controlled Variables

The following conditions were kept consistent where practical:

- PostgreSQL service.
- PostgreSQL container.
- Published database port.
- Database role used for authentication.
- Test client.
- General network environment.
- PostgreSQL authentication mechanism.
- Authorized testing scope.

Maintaining these conditions allows the before-and-after comparison to focus primarily on the hardening changes.

## 4.6 Baseline Procedure

The baseline phase was conducted before changing the PostgreSQL authentication configuration.

The following conditions were examined:

1. Current PostgreSQL HBA host rules.
2. PostgreSQL listening addresses.
3. Published Docker port.
4. PostgreSQL password authentication verifier.
5. Remote authentication using the previously known laboratory credential.
6. Source address observed by PostgreSQL.

The baseline connection was performed from Kali Linux using the Windows host's LAN address and TCP port 5432.

After successful authentication, the following PostgreSQL query was used to identify the authenticated identity, database, and client source:

`SELECT current_user, current_database(), inet_client_addr(), inet_client_port();`

This established the actual connection path visible from inside PostgreSQL.

## 4.7 Hardening Procedure

After the baseline state was recorded, the authentication configuration was hardened.

The hardening procedure consisted of:

1. Changing the known PostgreSQL credential.
2. Verifying that the PostgreSQL role continued to use SCRAM-SHA-256.
3. Inspecting the current HBA rules.
4. Applying a restrictive HBA source rule.
5. Reloading PostgreSQL configuration.
6. Verifying the active HBA rule.
7. Testing authentication using the new laboratory credential.
8. Testing authentication using the previously known credential.
9. Recording the resulting PostgreSQL responses.

The credential values themselves are intentionally omitted from this report and repository documentation.

## 4.8 HBA Source-Address Validation

An important methodological issue emerged during testing.

The external Kali Linux client used the Windows host LAN address to reach the published PostgreSQL port. However, PostgreSQL reported the Docker bridge gateway address as the connection source.

An initial HBA restriction based on the external LAN subnet therefore did not match the source address observed by PostgreSQL. The resulting connection was rejected because no applicable HBA rule existed for the observed source.

The HBA rule was then adjusted to the source address observed by PostgreSQL. The configuration was reloaded and tested again.

This process was important because it prevented an incorrect assumption about the network source from being treated as evidence that the authentication hardening itself had failed.

## 4.9 Supplementary Direct-LAN Validation

A supplementary experiment was conducted using PostgreSQL running directly on the Kali Linux system.

This test was not used as the primary containerized experiment. Its purpose was to determine whether HBA source-address restrictions behave as expected when PostgreSQL directly observes the client's LAN address.

The Kali PostgreSQL instance was configured to listen on the Kali LAN address. An HBA rule permitted the local laboratory subnet using SCRAM-SHA-256.

A Windows client then connected directly to the Kali PostgreSQL service. PostgreSQL reported the Windows host's actual LAN address as the client address.

Two additional validation conditions were tested:

1. The previously known credential was rejected after credential rotation.
2. A more restrictive HBA rule limited access to the PostgreSQL host itself, causing the Windows client's connection to be rejected because its source address was outside the permitted rule.

The supplementary test therefore provided supporting evidence for the interpretation of source-based HBA behaviour.

## 4.10 Test Sequence

The testing sequence was organized to preserve the before-and-after comparison.

| Phase | Test Activity | Purpose |
|---|---|---|
| Baseline | Inspect HBA configuration | Establish initial access-control state |
| Baseline | Inspect listening addresses | Determine network exposure |
| Baseline | Inspect published port | Confirm external reachability |
| Baseline | Verify password authentication mechanism | Establish authentication baseline |
| Baseline | Remote connection using known credential | Determine whether unauthorized scenario succeeds |
| Baseline | Query client source address | Determine observed network source |
| Hardening | Change credential | Invalidate previously known credential |
| Hardening | Verify SCRAM-SHA-256 | Confirm authentication mechanism remains enabled |
| Hardening | Apply HBA restriction | Reduce permitted source addresses |
| Hardening | Reload configuration | Activate HBA changes |
| Validation | Test new credential | Confirm intended authorized access |
| Validation | Test old credential | Determine whether previous access is prevented |
| Validation | Test denied source condition | Confirm HBA restriction |
| Supplementary | Direct-LAN source validation | Compare source visibility without Docker path |

## 4.11 Expected Results

The experiment was designed around the following expected outcomes.

### Baseline

The previously known credential was expected to authenticate successfully because it was valid and the baseline HBA host rule permitted the relevant connection.

### After Credential Rotation

The previously known credential was expected to fail authentication.

The new laboratory credential was expected to authenticate when the connection satisfied the applicable HBA rule.

### After HBA Restriction

A client whose observed source address did not match the permitted HBA rule was expected to receive an HBA rejection.

### Authentication Mechanism

SCRAM-SHA-256 was expected to remain the active password authentication mechanism after hardening.

These expectations provide testable conditions for evaluating the hypothesis.

## 4.12 Evidence Collection

Evidence was collected during the testing process rather than reconstructed solely from memory.

The evidence set includes screenshots covering:

- Baseline HBA configuration.
- PostgreSQL listening configuration.
- Published PostgreSQL port.
- Baseline authentication.
- Baseline remote access.
- Credential hardening.
- SCRAM-SHA-256 verification.
- HBA configuration changes.
- Configuration reload.
- Active HBA rules.
- Post-hardening state.
- Direct-LAN source validation.
- Old credential rejection.
- HBA denial.

Evidence was organized into the project's `Evidence/BEFORE/`, `Evidence/AFTER/`, and `Evidence/Network Validation/` directories.

Each evidence item was assigned a unique evidence identifier in `Evidence/evidence-index.md`.

The test matrix in `Evidence/test-matrix.md` maps individual tests to the corresponding evidence items.

## 4.13 Evidence Integrity and Data Protection

Evidence files were retained as screenshots of the controlled test environment.

Authentication secrets were not intentionally included in the final evidence index, test matrix, report, or repository documentation. Where screenshots could expose sensitive credential values, the project requires appropriate redaction before final submission.

The evidence index records the test or event associated with each evidence item and identifies its location and purpose.

Important evidence should be preserved without unnecessary modification so that the recorded result remains traceable to the original test event.

## 4.14 Reproducibility

The experiment was documented so that another authorized researcher can reproduce the main test sequence using the same type of local environment.

The repository contains:

- Project README.
- Capstone proposal.
- Hardened PostgreSQL policy.
- PostgreSQL authentication test procedure.
- Evidence index.
- Test matrix.
- Evidence screenshots.
- AI-use record.
- Final report and presentation artifacts.

The README documents prerequisites, test order, expected outcomes, evidence locations, safety boundaries, and known limitations.

Actual authentication secrets are intentionally excluded from reproducibility documentation. A researcher reproducing the experiment should create separate laboratory credentials rather than reuse credentials from this study.

## 4.15 Safety Controls and Stop Conditions

Testing was performed using controlled systems and limited connection attempts.

The following safety controls were applied:

1. Testing remained within the authorized local/classroom environment.
2. No external systems were targeted.
3. No password-cracking activity was performed.
4. No denial-of-service testing was performed.
5. Configuration changes were limited to the laboratory PostgreSQL systems.
6. The PostgreSQL HBA configuration was restored to the intended final laboratory state after temporary validation changes.
7. Temporary Docker resources created for network testing were removed after use.
8. Sensitive authentication values were excluded from project documentation.

Testing would have been stopped if:

- The activity affected an unauthorized system.
- Network traffic left the authorized laboratory scope.
- A production system was identified as the target.
- Testing caused unexpected service disruption.
- Sensitive real-world credentials or personal information were encountered.

## 4.16 Methodological Limitations

The methodology has several limitations.

First, the primary experiment was performed using Docker Desktop on Windows. The observed source-address translation may therefore differ from behaviour in a native Linux Docker environment, a virtual machine, or a production container orchestration platform.

Second, the experiment evaluates a specific PostgreSQL version and container configuration. Results may differ with other PostgreSQL versions, Docker networking modes, operating systems, or deployment architectures.

Third, the experiment focuses on password authentication and HBA source restrictions. It does not comprehensively evaluate database authorization privileges, application-layer authentication, TLS certificate authentication, or enterprise credential-management systems.

Fourth, the supplementary Kali PostgreSQL experiment is not equivalent to the primary containerized environment. It is used only to validate the interpretation of source-address-based HBA behaviour.

These limitations are considered when interpreting the results and determining the extent to which the findings can be generalized.

## 4.17 Chapter Summary

This chapter described the controlled methodology used to evaluate PostgreSQL authentication hardening.

The experiment used a before-and-after design consisting of baseline configuration examination, controlled remote authentication, credential rotation, HBA restriction, configuration validation, and post-hardening authentication tests.

The methodology also incorporated a supplementary direct-LAN test to examine source-address behaviour without the Docker Desktop networking path.

Evidence was collected during testing and organized using an evidence index and test matrix. Ethical boundaries, safety controls, data protection, reproducibility, and methodological limitations were also defined.

The next chapter examines the technical implementation and system configuration in greater detail.

# Chapter 5: Technical Implementation and System Examination

## 5.1 Overview

This chapter documents the technical environment examined during the PostgreSQL authentication-hardening experiment. It describes the containerized PostgreSQL deployment, network configuration, baseline authentication state, hardening changes, validation procedures, and supplementary direct-LAN experiment.

The primary system under investigation was the PostgreSQL 15 container running within the Week 1 Docker stack. The experiment was designed to establish the baseline exposure and authentication state before applying controlled security changes.

The technical implementation was divided into two stages:

1. **Primary containerized experiment:** PostgreSQL running inside Docker Desktop.
2. **Supplementary direct-LAN validation:** PostgreSQL running directly on Kali Linux to validate source-address-based HBA behaviour.

The primary experiment is the basis for the project's research question and hypothesis. The supplementary experiment provides additional context for interpreting the network behaviour observed in Docker Desktop.

## 5.2 Primary Docker Environment

The primary PostgreSQL service was deployed using the `postgres:15-alpine` Docker image.

The container was named `medusa-postgres` and was connected to the Docker bridge network used by the Week 1 application stack.

The relevant configuration included:

| Component | Configuration |
|---|---|
| Container | `medusa-postgres` |
| Database engine | PostgreSQL 15 |
| Image | `postgres:15-alpine` |
| Database | `medusa-store` |
| Database role | `postgres` |
| PostgreSQL port | 5432 |
| Host published port | 5432 |
| Docker network | `week1_medusa-net` |
| Network type | Bridge |
| PostgreSQL listening address | `*` |
| Authentication mechanism | SCRAM-SHA-256 |

The PostgreSQL container was healthy and running before the authentication tests were conducted.

The Redis service present in the same Docker stack was not part of the authentication experiment and was not used as an experimental variable.

## 5.3 Docker Network Configuration

The PostgreSQL container was connected to the Docker bridge network `week1_medusa-net`.

The network used the following relevant addressing:

| Network Property | Observed Value |
|---|---|
| Network type | Bridge |
| Subnet | `DOCKER_NETWORK` |
| Gateway | `DOCKER_GATEWAY` |
| PostgreSQL container address | `POSTGRES_CONTAINER` |
| Redis container address | `REDIS_CONTAINER` |

The Windows host exposed PostgreSQL through TCP port 5432.

The authorized Kali Linux test client used its LAN address to connect to the Windows host. This created the following connection path:

**Kali Linux → Windows host LAN interface → Docker published port → Docker bridge network → PostgreSQL container**

The distinction between the external client address and the address observed by PostgreSQL became an important finding during the experiment.

## 5.4 Baseline HBA Configuration

The initial PostgreSQL HBA configuration was examined before any hardening changes were made.

The relevant host rules included localhost entries using `trust` authentication and a general host rule covering all source addresses using SCRAM-SHA-256.

The principal baseline host rule was conceptually:

`host all all all scram-sha-256`

This meant that remote host connections matching the rule were required to authenticate using SCRAM-SHA-256, but the source-address field was not restricted to a specific client or network.

The baseline configuration therefore provided password authentication but did not impose a narrow source-address boundary for the general host rule.

The localhost and replication rules were retained as existing PostgreSQL configuration and were not the focus of the remote authentication experiment.

## 5.5 Baseline Listening Configuration

The PostgreSQL `listen_addresses` setting was examined during the baseline phase.

The observed value was:

`*`

This configuration means that PostgreSQL was listening for connections on all available network interfaces within the container.

The finding was relevant because a PostgreSQL service that listens beyond the local loopback interface can receive network connections when an appropriate network path exists.

The listening configuration was therefore considered together with the Docker port publication and HBA rules rather than treated as an isolated security control.

## 5.6 Baseline Port Exposure

Docker was examined to determine whether PostgreSQL port 5432 was published to the Windows host.

The observed Docker port mapping was:

`5432/tcp → 0.0.0.0:5432`

and:

`5432/tcp → [::]:5432`

This confirmed that the PostgreSQL container's port 5432 was published on the host.

The published port made the PostgreSQL service reachable from the authorized Kali Linux test environment through the Windows host LAN address.

## 5.7 Baseline Authentication Mechanism

The PostgreSQL password verifier for the `postgres` role was examined during the baseline phase.

The observed verifier type was:

`SCRAM-SHA-256`

This established that the baseline experiment was already using SCRAM-SHA-256 authentication.

This is an important methodological finding because the hardening experiment did not involve replacing a weaker password authentication mechanism with SCRAM-SHA-256. Instead, SCRAM-SHA-256 was retained while the known credential and HBA source policy were changed.

## 5.8 Baseline Remote Authentication Test

The baseline remote authentication test was performed from the authorized Kali Linux client.

The client connected to the Windows host LAN address on TCP port 5432 using the PostgreSQL `postgres` role and the previously known laboratory credential.

The connection succeeded.

After authentication, PostgreSQL was queried using:

`SELECT current_user, current_database(), inet_client_addr(), inet_client_port();`

The resulting session information confirmed:

- Authenticated user: `postgres`
- Database: `medusa-store`
- Observed client address: `DOCKER_GATEWAY`

The successful baseline connection established the initial experimental condition: a remotely reachable PostgreSQL service could be accessed using the previously known laboratory credential.

The result was recorded as baseline evidence.

## 5.9 Observed Docker Source Address

The source address reported by PostgreSQL was an important technical finding.

The Kali Linux client used the Windows host LAN address to reach PostgreSQL, but PostgreSQL reported:

`DOCKER_GATEWAY`

as the client address.

This address corresponded to the Docker bridge gateway.

The observation demonstrated that the address visible to PostgreSQL was not the same as the original Kali LAN address.

This distinction directly affected the HBA hardening process. An HBA rule written only for the external Kali subnet would not match the source address PostgreSQL actually evaluated.

## 5.10 Initial HBA Restriction Attempt

During the hardening process, an initial restrictive HBA rule was configured for the external LAN subnet.

The intended policy was to permit the authorized LAN network while requiring SCRAM-SHA-256 authentication.

However, when the connection was tested, PostgreSQL reported that there was no applicable HBA entry for the observed source address.

The relevant failure condition identified the client as:

`DOCKER_GATEWAY`

rather than the external Kali LAN address.

This result was important because it demonstrated that the HBA restriction itself was functioning, but the policy had been written against an address that PostgreSQL did not observe.

The finding led to refinement of the HBA policy rather than being treated as evidence of a password-authentication failure.

## 5.11 Final Docker HBA Restriction

The HBA rule was subsequently adjusted to the source address observed by PostgreSQL.

The final remote host rule was:

`host all all DOCKER_GATEWAY/32 scram-sha-256`

This rule restricted the applicable remote host connection to the observed Docker bridge gateway source while retaining SCRAM-SHA-256 authentication.

The `/32` notation restricts the rule to a single IPv4 address.

After modifying the HBA configuration, PostgreSQL's configuration was reloaded.

The active HBA rules were then queried to verify that the intended rule was loaded and that no configuration error was reported.

## 5.12 Credential Rotation

The previously known PostgreSQL credential was replaced with a new laboratory credential during the hardening phase.

The actual credential value is intentionally excluded from this report and repository documentation.

Credential rotation was necessary because a restrictive authentication mechanism cannot prevent authentication by a client that already possesses a valid credential and is permitted by the applicable HBA rule.

The purpose of this change was therefore to invalidate the previously known credential while preserving the SCRAM-SHA-256 authentication mechanism.

## 5.13 Post-Hardening SCRAM Verification

After credential rotation, the PostgreSQL password verifier was checked again.

The verifier remained:

`SCRAM-SHA-256`

This confirmed that the credential change did not weaken the configured password authentication mechanism.

The result also ensured that the comparison between the baseline and hardened states remained focused on credential validity and HBA source restrictions rather than a simultaneous change to the authentication protocol.

## 5.14 HBA Configuration Reload

After modifying the HBA configuration, PostgreSQL was instructed to reload its configuration.

The reload operation was verified before conducting the final connection tests.

The active HBA configuration was then queried using PostgreSQL's HBA inspection facilities.

The resulting configuration contained the intended restrictive remote host rule and did not report configuration errors.

This verification step was necessary to distinguish between a configuration file modification and an actually active PostgreSQL access-control policy.

## 5.15 Authorized Post-Hardening Connection

The authorized client was tested using the new laboratory credential.

The connection was successful after the HBA rule was adjusted to match the source address observed by PostgreSQL.

The successful session demonstrated that the hardened configuration still allowed the intended connection path when both conditions were satisfied:

1. The connection originated from the source permitted by the HBA rule.
2. The client supplied the current valid credential.

The resulting PostgreSQL session again reported the Docker bridge gateway as the client source.

This confirmed that the source-address observation remained consistent after hardening.

## 5.16 Previously Known Credential Test

The previously known credential was then tested against the hardened PostgreSQL service.

The connection failed with a PostgreSQL password authentication error.

This was different from the earlier HBA denial observed during the initial restrictive-rule attempt.

The final test therefore demonstrated that the previously known credential was no longer valid for authentication after credential rotation.

This result provides the principal evidence supporting the authentication-hardening hypothesis.

## 5.17 Final Docker Authentication State

The final Docker configuration combined the following controls:

- PostgreSQL remained reachable through the required laboratory network path.
- PostgreSQL continued to use SCRAM-SHA-256.
- The previously known credential had been replaced.
- The HBA source rule was restricted to the source address observed by PostgreSQL.
- The HBA configuration was reloaded and verified.
- The new credential successfully authenticated through the permitted path.
- The previously known credential was rejected.

The resulting security state therefore preserved authorized access while preventing the specific previously successful credential-based access scenario.

## 5.18 Supplementary Direct-LAN PostgreSQL Environment

A separate PostgreSQL 18.4 instance was configured on the Kali Linux system for supplementary network validation.

This system was not used as a replacement for the primary Docker experiment.

The supplementary PostgreSQL instance was configured to listen on the Kali LAN address:

`KALI_HOST`

The Windows host used the LAN address:

`WINDOWS_HOST`

The PostgreSQL HBA configuration included a SCRAM-SHA-256 rule for the authorized LAN subnet.

This configuration allowed the Windows host to connect directly to PostgreSQL running on Kali without passing through the Docker published-port path used in the primary experiment.

## 5.19 Direct-LAN Source Address Verification

The Windows host successfully connected to the supplementary Kali PostgreSQL service using the current laboratory credential.

PostgreSQL reported the client address as:

`WINDOWS_HOST`

This was significant because it showed that when PostgreSQL directly received the LAN connection, it observed the actual Windows client address.

The result provides a useful contrast with the Docker experiment, where PostgreSQL observed the Docker bridge gateway address.

The comparison demonstrates that HBA source-address policies must be designed according to the actual network path and address visibility of the deployment.

## 5.20 Direct-LAN Credential Validation

The previously known credential was also tested against the supplementary direct-LAN PostgreSQL service.

The authentication attempt failed with a password authentication error.

This result was consistent with the primary Docker experiment and provided additional evidence that credential rotation invalidated the previously known credential.

## 5.21 Direct-LAN HBA Restriction Validation

The supplementary PostgreSQL HBA configuration was temporarily changed to permit only the Kali PostgreSQL host address.

The Windows host then attempted to connect using the current valid credential.

The connection was rejected because the Windows client's source address was not covered by the restrictive HBA rule.

PostgreSQL reported that no applicable HBA entry existed for the Windows client's address.

The HBA rule was subsequently restored to the intended laboratory subnet configuration.

This supplementary test demonstrated that HBA source restrictions can prevent an otherwise valid credential from establishing a session when the client's source address falls outside the permitted policy.

## 5.22 Technical Comparison of the Two Network Paths

The two experiments produced different source-address observations because they used different network architectures.

| Characteristic | Primary Docker Test | Supplementary Direct-LAN Test |
|---|---|---|
| PostgreSQL location | Docker container | Kali Linux host |
| Client | Kali Linux | Windows host |
| Network path | Windows published port and Docker bridge | Direct LAN connection |
| PostgreSQL observed source | Docker bridge gateway | Windows LAN address |
| HBA source policy | Based on observed Docker source | Based on actual LAN source |
| Purpose | Primary research experiment | Supporting network validation |

The comparison shows why network architecture must be considered when applying source-based database access controls.

## 5.23 Technical Implementation Summary

The technical implementation consisted of controlled examination and modification of PostgreSQL authentication and network-access controls.

The baseline state demonstrated remote reachability and successful authentication using the previously known credential.

The hardening process changed the credential, retained SCRAM-SHA-256, and restricted the HBA rule according to the source address actually observed by PostgreSQL.

Post-hardening tests demonstrated that the new credential could authenticate through the permitted path while the previously known credential was rejected.

The supplementary direct-LAN test further demonstrated that HBA restrictions operate against the source address visible to PostgreSQL and that this address depends on the network architecture.

## 5.24 Chapter Summary

This chapter documented the technical implementation of the PostgreSQL authentication-hardening experiment.

The primary Docker environment exposed PostgreSQL through the Windows host's published port and used SCRAM-SHA-256 authentication. The baseline test confirmed successful remote access using the previously known credential.

During hardening, the credential was rotated and the HBA policy was restricted. An important networking observation showed that PostgreSQL saw the Docker bridge gateway rather than the original client LAN address. The HBA rule was therefore adjusted to match the observed source.

The final Docker tests showed that the current credential could authenticate through the permitted path while the previously known credential was rejected.

A supplementary direct-LAN experiment confirmed that PostgreSQL can observe the actual client LAN address when the Docker networking path is absent and demonstrated the operation of restrictive HBA rules under that condition.

The next chapter presents the experimental results and maps the observed outcomes to the project's evidence and test matrix.

# Chapter 6: Results and Evidence

## 6.1 Overview

This chapter presents the results obtained from the PostgreSQL authentication-hardening experiment. The results are organized according to the testing sequence and are linked to the evidence identifiers maintained in the project evidence index.

The primary experiment compares the PostgreSQL configuration and authentication behaviour before and after hardening. The supplementary direct-LAN experiment is presented separately because it was used to validate network-source behaviour outside the primary Docker networking path.

The results distinguish between three important outcomes:

1. Successful authentication.
2. Password authentication failure.
3. HBA rejection because no applicable `pg_hba.conf` rule matched the observed client source.

This distinction is necessary because these outcomes occur at different stages of the PostgreSQL connection process.

## 6.2 Baseline Results

The baseline phase established the initial state of the PostgreSQL container before authentication hardening.

The baseline evidence showed that PostgreSQL was configured to accept host connections using SCRAM-SHA-256 and that the service was exposed through the Docker-published PostgreSQL port.

The principal baseline observations were:

- PostgreSQL was running in the `medusa-postgres` container.
- PostgreSQL was listening on all addresses.
- TCP port 5432 was published by Docker.
- The `postgres` role used SCRAM-SHA-256.
- The general host HBA rule permitted all source addresses while requiring SCRAM-SHA-256.
- A remote connection from the authorized Kali Linux test environment succeeded using the previously known laboratory credential.
- PostgreSQL reported the Docker bridge gateway as the client source address.

These results established the baseline condition against which the hardened configuration was compared.

## 6.3 Baseline HBA Configuration

The baseline HBA configuration was captured as evidence item E-001.

The relevant host rules included localhost entries and a general host rule permitting connections from all source addresses with SCRAM-SHA-256 authentication.

The key baseline rule was:

`host all all all scram-sha-256`

The result demonstrates that the baseline configuration already required SCRAM-SHA-256 for the general host connection path.

However, the source-address field was unrestricted.

**Evidence:** E-001 — `BEFORE/Current_PostgreSQL_HBA_Configuration.png`

## 6.4 Baseline Listening Address

The PostgreSQL listening configuration was captured as evidence item E-002.

The observed `listen_addresses` value was:

`*`

This showed that PostgreSQL was listening on all available network interfaces within the container.

The observation was significant because it confirmed that PostgreSQL was configured to accept network connections beyond a loopback-only configuration.

**Evidence:** E-002 — `BEFORE/Listen_Addresses.png`

## 6.5 Baseline PostgreSQL Port Exposure

Docker port exposure was captured as evidence item E-003.

The PostgreSQL container exposed port 5432 through the Windows host using the published-port configuration.

The observed mappings were:

`5432/tcp → 0.0.0.0:5432`

and:

`5432/tcp → [::]:5432`

This confirmed that PostgreSQL was reachable through the Windows host's network interfaces.

**Evidence:** E-003 — `BEFORE/PostgreSQL_Expose_Port5432.png`

## 6.6 Baseline Authentication Mechanism

The baseline PostgreSQL password verifier was captured as evidence item E-004.

The verifier for the `postgres` role was:

`SCRAM-SHA-256`

This established that SCRAM-SHA-256 was already active before hardening.

This result is important to the interpretation of the project because the hardening experiment did not introduce SCRAM-SHA-256. Instead, SCRAM-SHA-256 remained enabled while the credential and HBA source policy were changed.

**Evidence:** E-004 — `BEFORE/Valid_Baselne_Postgresql.png`

## 6.7 Baseline Remote Authentication

A remote connection from the authorized Kali Linux client was successfully established during the baseline test.

After authentication, PostgreSQL returned the following session characteristics:

| Observation | Result |
|---|---|
| Current user | `postgres` |
| Current database | `medusa-store` |
| Client source observed by PostgreSQL | `DOCKER_GATEWAY` |

The successful session demonstrated that the previously known credential was sufficient to authenticate remotely through the exposed PostgreSQL service under the baseline configuration.

**Evidence:** E-005 — `BEFORE/Remote_Access_Evidence.png`

## 6.8 Baseline Source Address Finding

The baseline connection produced an important networking observation.

The Kali Linux client connected through the Windows host LAN address, but PostgreSQL reported the client source as:

`DOCKER_GATEWAY`

This was the Docker bridge gateway address.

The result showed that the source address visible to PostgreSQL differed from the original external client address.

This observation became important when implementing the restrictive HBA policy because PostgreSQL evaluates HBA rules against the address it observes at the database connection boundary.

**Evidence:** E-005 — `BEFORE/Remote_Access_Evidence.png`

## 6.9 Credential Rotation Result

During hardening, the previously known PostgreSQL credential was replaced with a new laboratory credential.

The actual credential values are intentionally excluded from this report and repository evidence. The screenshot containing the credential value was intentionally excluded from the final evidence set.

The credential rotation was validated through subsequent authentication testing: the current credential successfully authenticated through the permitted path, while the previously known credential was rejected.

The purpose of the change was to invalidate the previously known credential while retaining SCRAM-SHA-256 authentication.

**Evidence:** The credential-containing screenshot was not retained as submitted evidence; the rotation is validated by the subsequent post-hardening authentication results.

## 6.10 Post-Hardening SCRAM Verification

After credential rotation, the PostgreSQL password verifier was checked again.

The verifier remained:

`SCRAM-SHA-256`

The result showed that the authentication mechanism had not been weakened or replaced during hardening.

The finding supports the comparison between the baseline and hardened states because the authentication protocol remained constant while credential validity changed.

**Evidence:** E-006 — `AFTER/Password_Format_Check.png`

## 6.11 PostgreSQL Port State After Hardening

The PostgreSQL port exposure was checked after the hardening changes.

The service remained available through the expected laboratory port.

This was intentional because the experiment was designed to evaluate authentication and HBA controls rather than completely remove network reachability.

Maintaining the network path allowed the post-hardening authentication tests to determine whether the security controls could reject the previously successful credential while preserving intended access.

**Evidence:** E-007 — `AFTER/Postgres_Port.png`

## 6.12 HBA Configuration Reload

After the HBA configuration was modified, PostgreSQL was instructed to reload its configuration.

The reload was recorded as evidence item E-009.

The reload was necessary to ensure that the modified HBA policy was active before conducting the final authentication tests.

**Evidence:** E-008 — `AFTER/pg_reload_conf.png`

## 6.13 Initial HBA Restriction Result

The first restrictive HBA rule was based on the external laboratory LAN subnet.

When the remote connection was tested, PostgreSQL rejected the connection because the source address it observed did not match the configured HBA rule.

The PostgreSQL error identified the source as:

`DOCKER_GATEWAY`

and reported that no applicable `pg_hba.conf` entry existed.

This result demonstrated that the HBA control was enforcing the source restriction, but the initial policy did not correspond to the address visible to PostgreSQL.

The result was therefore treated as an HBA source-address mismatch rather than a password authentication failure.

**Evidence:** E-009 — `AFTER/HBA_Restriction_Rule_Set.png`

## 6.14 HBA Policy Refinement

Following the initial HBA denial, the rule was refined to match the source address actually observed by PostgreSQL.

The resulting host rule was:

`host all all DOCKER_GATEWAY/32 scram-sha-256`

The `/32` prefix restricted the rule to a single IPv4 source address.

This change allowed the HBA policy to reflect the actual Docker networking path observed during the experiment.

**Evidence:** E-009 — `AFTER/HBA_Restriction_Rule_Set.png`

## 6.15 Active HBA Verification

After the HBA configuration was reloaded, the active PostgreSQL HBA rules were queried.

The resulting configuration showed the restrictive source rule and no configuration errors.

This confirmed that the intended HBA policy was active rather than merely present in an edited configuration file.

**Evidence:** E-010 — `AFTER/Active_HBA_Rule.png`

## 6.16 Authorized Post-Hardening Authentication

The authorized Kali Linux client was tested using the new laboratory credential after the HBA policy had been adjusted to the source address observed by PostgreSQL.

The connection succeeded.

The PostgreSQL session again identified the client source as the Docker bridge gateway.

The successful connection demonstrated that the hardened configuration did not prevent the intended authorized test path when the client satisfied both the HBA source condition and the current credential requirement.

**Evidence:** E-011 — `AFTER/Current_State.png`

## 6.17 Previously Known Credential Rejection

The previously known credential was then tested against the hardened Docker PostgreSQL service.

The authentication attempt failed with:

`password authentication failed for user "postgres"`

This result is distinct from the earlier HBA denial.

In this final test, the source address matched the applicable HBA rule, allowing PostgreSQL to proceed to password authentication. The previously known credential was then rejected because it was no longer valid.

This is the principal post-hardening result for the credential component of the hypothesis.

## 6.18 Final Active HBA State

The final active HBA configuration was captured as evidence item E-013.

The relevant remote rule remained restricted to the source address observed by PostgreSQL and required SCRAM-SHA-256.

The final configuration therefore combined:

- A restricted source address.
- SCRAM-SHA-256 authentication.
- A rotated credential.

**Evidence:** E-012 — `AFTER/Active_HBA_Rule_2.png`

## 6.19 Supplementary Direct-LAN Results

The supplementary direct-LAN experiment was conducted separately from the primary Docker experiment.

PostgreSQL running directly on Kali Linux was configured to listen on the Kali LAN address.

The Windows host successfully connected to this PostgreSQL instance using the current laboratory credential.

PostgreSQL reported the Windows host's LAN address as:

`WINDOWS_HOST`

This differed from the source address observed in the Docker experiment.

The result confirmed that source-address visibility depends on the network architecture through which the connection reaches PostgreSQL.

**Evidence:** E-013 — `Network Validation/Windows_to_Kali_Source_IP.png`

## 6.20 Supplementary Credential Rejection

The previously known credential was tested against the supplementary direct-LAN PostgreSQL instance.

The authentication attempt failed with a password authentication error.

This provided supporting evidence that the credential rotation was effective beyond the primary Docker test.

The result was not used as a replacement for the primary experiment because the supplementary system used a different PostgreSQL deployment architecture.

**Evidence:** E-014 — `Network Validation/Windows_to_Kali_Old_Password_Rejected.png`

## 6.21 Supplementary HBA Denial

The supplementary PostgreSQL HBA configuration was temporarily restricted to the Kali PostgreSQL host address.

The Windows client then attempted to connect using the current valid credential.

PostgreSQL rejected the connection because the Windows client's source address was not covered by the restrictive HBA rule.

The resulting error identified the Windows host address and reported that no applicable HBA entry existed.

This demonstrated that HBA source restrictions can prevent a connection before password authentication succeeds.

The HBA configuration was subsequently restored to the intended laboratory subnet.

**Evidence:** E-015 — `Network Validation/Windows_to_Kali_HBA_Denied.png`

## 6.22 Before-and-After Comparison

The principal changes and results are summarized below.

| Security Property | Baseline | Hardened State | Result |
|---|---|---|---|
| PostgreSQL network exposure | Port 5432 published | Port 5432 retained for testing | Reachability preserved |
| `listen_addresses` | `*` | Retained | Network behaviour preserved |
| Password verifier | SCRAM-SHA-256 | SCRAM-SHA-256 | Authentication mechanism retained |
| Credential | Previously known credential valid | Previous credential invalidated | Security improved |
| General HBA source scope | Broad source allowance | Restricted observed source | Access boundary reduced |
| Known credential remote test | Successful | Rejected | Previous access path prevented |
| Current credential through permitted source | Not applicable to baseline comparison | Successful | Intended access preserved |
| Denied HBA source | Not tested as hardened condition | Rejected | HBA restriction effective |

## 6.23 Test Matrix Results

The following table summarizes the principal test results documented in the project test matrix.

| Test ID | Test Condition | Expected Result | Observed Result | Evidence |
|---|---|---|---|---|
| T-001 | Baseline remote connection using known credential | Connection succeeds | Connection succeeded | E-004, E-005 |
| T-002 | Inspect baseline HBA | Broad host rule identified | Broad SCRAM host rule observed | E-001 |
| T-003 | Inspect PostgreSQL listening address | Network listening identified | `*` observed | E-002 |
| T-004 | Inspect Docker port exposure | Port 5432 published | Port 5432 published | E-003, E-008 |
| T-005 | Rotate credential | Previous credential invalidated | Credential changed; credential-containing screenshot excluded from final evidence | Not retained |
| T-006 | Verify authentication mechanism | SCRAM retained | SCRAM-SHA-256 observed | E-006 |
| T-007 | Reload PostgreSQL configuration | HBA changes activated | Reload performed | E-008 |
| T-008 | Apply initial restrictive HBA rule | Unmatched source denied | HBA denial occurred | E-009 |
| T-009 | Verify active HBA | Restrictive rule active | Rule active without errors | E-010 |
| T-010 | Test current credential through permitted source | Authentication succeeds | Authentication succeeded | E-011 |
| T-011 | Verify final HBA | Final restrictive rule active | Rule active | E-012 |
| T-012 | Direct-LAN source validation | Actual client source observed | Windows source observed as `WINDOWS_HOST` | E-013 |
| T-013 | Direct-LAN old credential test | Old credential rejected | Authentication failed | E-014 |
| T-014 | Direct-LAN denied source test | HBA denies source | HBA denied connection | E-015 |

## 6.24 Key Findings

The results produced five principal findings.

### Finding 1: Baseline Remote Authentication Was Possible

The baseline test demonstrated that the previously known credential could be used to establish a remote PostgreSQL session through the published Docker port.

This established the vulnerable or undesired baseline condition required for the experiment.

### Finding 2: SCRAM-SHA-256 Was Already Enabled

The baseline PostgreSQL role was already using SCRAM-SHA-256.

Therefore, the experiment demonstrates that a strong password authentication mechanism alone does not prevent unauthorized access when a valid credential is known.

### Finding 3: Credential Rotation Prevented Reuse of the Previously Known Credential

After the credential was changed, the previously known credential produced a password authentication failure.

This indicates that credential rotation successfully invalidated the specific credential used during the baseline test.

### Finding 4: HBA Restrictions Depend on the Observed Source Address

The initial HBA restriction based on the external LAN source did not match the source address visible to PostgreSQL through Docker Desktop.

PostgreSQL observed the Docker bridge gateway address instead.

After the HBA policy was adjusted to the observed source, the intended authorized connection succeeded.

### Finding 5: HBA Can Deny Connections Before Authentication

The supplementary direct-LAN test demonstrated that a source address outside the permitted HBA range was rejected with a `no pg_hba.conf entry` error even when the current credential was valid.

This confirms that HBA source restrictions can provide an access-control boundary before successful password authentication.

## 6.25 Result Interpretation Boundary

The results support a specific conclusion rather than a claim of complete PostgreSQL security.

The experiment demonstrates that the combination of credential rotation and restrictive HBA rules prevented the specific previously successful remote credential scenario tested in the laboratory environment.

The results do not demonstrate that all unauthorized PostgreSQL access is impossible.

For example, a client that possesses the current credential and originates from a permitted source may still authenticate. Similarly, changes to Docker networking could alter the source address observed by PostgreSQL and therefore require corresponding changes to the HBA policy.

These boundaries are considered in the analysis and discussion chapter.

## 6.26 Chapter Summary

The experimental results show a clear difference between the baseline and hardened states.

Before hardening, the previously known credential successfully authenticated remotely through the Docker-published PostgreSQL port. PostgreSQL was already using SCRAM-SHA-256, but the general HBA host rule had a broad source scope.

After hardening, the credential was rotated and the HBA policy was restricted to the source address observed by PostgreSQL. The current credential successfully authenticated through the permitted path, while the previously known credential was rejected.

The supplementary direct-LAN experiment further demonstrated that HBA source restrictions can deny clients whose actual source address is outside the permitted policy.

These results provide the empirical basis for evaluating the research hypothesis in the next chapter.

# Chapter 7: Analysis and Discussion

## 7.1 Introduction

The results presented in Chapter 6 provide evidence for evaluating the effectiveness of PostgreSQL authentication hardening against the specific threat scenario defined in this project.

The analysis focuses on the research question:

**To what extent do SCRAM-SHA-256 authentication and restrictive `pg_hba.conf` rules prevent unauthorized remote PostgreSQL connections using a previously known credential in a containerized environment?**

The analysis compares the baseline and hardened states and distinguishes direct observations from interpretations and inferences. Particular attention is given to credential validity, HBA source restrictions, Docker networking, and preservation of authorized access.

## 7.2 Baseline Security Condition

The baseline experiment established that the PostgreSQL service was remotely reachable through the Docker-published port and that the previously known laboratory credential successfully authenticated.

This is a direct observation supported by the successful remote PostgreSQL session recorded during the baseline phase.

The baseline PostgreSQL configuration already used SCRAM-SHA-256 for the general host authentication rule. Therefore, the successful connection cannot reasonably be interpreted as evidence that SCRAM-SHA-256 itself was absent or misconfigured.

Instead, the result demonstrates an important security principle: a strong authentication protocol does not prevent authentication when a client possesses a valid credential.

### Observation

The known credential successfully established a remote PostgreSQL session.

### Interpretation

The authentication boundary accepted the credential because it was valid and the connection matched the applicable HBA rule.

### Inference

Credential validity is an independent security factor from the strength of the authentication protocol.

This distinction is central to answering the research question.

## 7.3 Effect of Credential Rotation

Credential rotation was one of the principal hardening controls.

After the previously known credential was replaced, an authentication attempt using that previous credential resulted in a password authentication failure.

This demonstrates that the credential used successfully during the baseline phase was no longer valid after hardening.

### Observation

The previously known credential was rejected after the credential change.

### Interpretation

PostgreSQL proceeded to password authentication for the tested source and rejected the credential because it was no longer valid.

### Inference

Credential rotation successfully removed the specific authentication path demonstrated during the baseline test.

The result therefore provides strong support for the credential component of the hardening hypothesis.

However, the result should not be generalized to all possible credentials or all possible attackers. The experiment only establishes that the previously known credential could no longer be used.

## 7.4 Role of SCRAM-SHA-256

SCRAM-SHA-256 remained active before and after hardening.

This provides an important control for the experiment because the authentication mechanism did not change when the credential was rotated.

The experiment can therefore attribute the change in the result primarily to the credential and HBA policy changes rather than to replacing one authentication protocol with another.

### Observation

The PostgreSQL verifier remained SCRAM-SHA-256 after the credential change.

### Interpretation

The password authentication mechanism remained consistent throughout the principal comparison.

### Inference

The failure of the previously known credential was not caused by replacing SCRAM-SHA-256 with another authentication mechanism.

The experiment therefore demonstrates that SCRAM-SHA-256 should be considered one component of a broader authentication-control strategy rather than a complete solution to known-credential risk.

## 7.5 Effect of the HBA Restriction

The HBA restriction provided a separate access-control mechanism.

The baseline HBA configuration contained a broad host rule that permitted all source addresses while requiring SCRAM-SHA-256.

During hardening, the source condition was restricted.

An initial attempt to restrict the policy to the external LAN subnet resulted in an HBA denial because PostgreSQL observed a different source address.

After the policy was adjusted to match the observed Docker source, the authorized connection succeeded.

### Observation

The initial restrictive rule produced a `no pg_hba.conf entry` error.

### Interpretation

The HBA rule was being enforced, but the configured source address did not match the address PostgreSQL observed.

### Inference

HBA source restrictions are dependent on the network path and the address visible at the PostgreSQL connection boundary.

This is a significant finding because it shows that an access-control policy can be technically restrictive but operationally incorrect if the network architecture is misunderstood.

## 7.6 Docker Networking as a Security-Relevant Factor

The Docker networking behaviour was one of the most significant findings in the project.

The Kali Linux client originated the connection from its LAN environment, but PostgreSQL reported the Docker bridge gateway as the client source.

This meant that PostgreSQL did not directly evaluate the original external client address for the Docker-published connection.

The result has practical security implications.

An administrator who assumes that PostgreSQL will see the external client IP may create an HBA rule that unintentionally denies authorized clients or fails to enforce the intended boundary.

The experiment therefore demonstrates that source-address restrictions must be designed using the actual network topology and the address visible to the service enforcing the rule.

## 7.7 Distinguishing HBA Denial from Authentication Failure

The experiment produced two different denial conditions.

The first was an HBA denial:

`no pg_hba.conf entry`

The second was a password authentication failure:

`password authentication failed for user "postgres"`

These errors represent different control stages.

An HBA denial means that PostgreSQL did not find an applicable host-based authentication rule for the connection.

A password authentication failure occurs after the connection has reached an applicable authentication rule but the presented credential was not accepted.

This distinction allowed the experiment to identify which control was responsible for each observed result.

The distinction also prevented the initial HBA source mismatch from being incorrectly interpreted as a failed credential test.

## 7.8 Preservation of Authorized Access

Security hardening should not only block unwanted access; it should preserve legitimate access where required.

After the HBA policy was adjusted to match the source address observed by PostgreSQL, the authorized test client successfully authenticated using the current laboratory credential.

This result demonstrates that the hardened configuration was not simply a complete denial of remote access.

### Observation

The current credential successfully authenticated through the permitted Docker source condition.

### Interpretation

The final HBA rule allowed the intended source and the current credential was accepted.

### Inference

The implemented controls were capable of reducing the unauthorized credential scenario while preserving the tested authorized access path.

This supports the practical usefulness of layered authentication controls.

## 7.9 Supplementary Direct-LAN Findings

The direct-LAN PostgreSQL experiment provided supporting evidence for the interpretation of the Docker networking result.

When PostgreSQL ran directly on Kali Linux, the Windows client connected to it without the Docker published-port path.

In that configuration, PostgreSQL observed the Windows host's actual LAN address.

When the HBA rule was temporarily restricted to the Kali PostgreSQL host address, the Windows client's connection was denied because its source address was outside the permitted rule.

This supplementary experiment demonstrates that HBA source restrictions operate according to the address visible to PostgreSQL.

It also shows that the difference between the Docker and direct-LAN results is attributable to the different network architectures rather than to inconsistent HBA behaviour.

## 7.10 Evaluation of the Research Question

The research question asks to what extent SCRAM-SHA-256 authentication and restrictive HBA rules prevent unauthorized remote PostgreSQL connections using a previously known credential.

The experiment indicates that these controls can prevent the specific unauthorized connection scenario tested, but their effectiveness depends on how the controls are implemented.

SCRAM-SHA-256 provides a strong password authentication mechanism, but it did not prevent the baseline connection because the credential was valid and known.

Credential rotation removed the validity of the previously known credential.

The restrictive HBA rule reduced the permitted source boundary, but its correct operation depended on identifying the source address actually observed by PostgreSQL.

The combined result is therefore stronger than any individual control considered alone.

The findings support the conclusion that authentication hardening is most effective when credential management and source-based access controls are implemented together.

## 7.11 Hypothesis Evaluation

The project defined two hypotheses.

### Null Hypothesis (H0)

**Changing the known PostgreSQL credential and applying restrictive `pg_hba.conf` rules will not materially prevent unauthorized remote PostgreSQL connections in the tested containerized environment.**

### Alternative Hypothesis (H1)

**Changing the known PostgreSQL credential while retaining SCRAM-SHA-256 authentication and applying restrictive `pg_hba.conf` rules will prevent unauthorized remote PostgreSQL connections using the previously known credential in the tested containerized environment.**

The baseline test demonstrated successful authentication using the previously known credential.

After hardening, the previously known credential was rejected, while the current credential was accepted through the permitted source condition.

The evidence therefore supports **H1 for the specific tested threat scenario** and provides insufficient support for H0.

The conclusion must remain bounded by the experimental conditions. The results do not establish that all unauthorized access is impossible under all network configurations.

## 7.12 Control Effectiveness Assessment

The controls can be assessed individually and collectively.

| Control | Effectiveness | Analysis |
|---|---|---|
| Credential rotation | High for the tested known credential | The previous credential was rejected after rotation |
| SCRAM-SHA-256 | Important but insufficient alone | Strong authentication remained enabled, but a valid known credential authenticated during baseline |
| Restrictive HBA | Effective when correctly configured | Denied sources outside the permitted policy |
| Source-IP verification | Essential for correct HBA design | Revealed the Docker bridge source seen by PostgreSQL |
| Configuration reload and verification | Effective operational control | Ensured modified HBA rules were active |
| Combined controls | Effective for tested scenario | Preserved intended access while preventing the previously successful credential scenario |

The assessment is specific to the experiment and should not be interpreted as a universal security rating for PostgreSQL.

## 7.13 Defense-in-Depth Interpretation

The results provide evidence for a layered security approach.

The network path determines whether the PostgreSQL service is reachable.

HBA determines whether the observed source is permitted.

SCRAM-SHA-256 determines how the password credential is authenticated.

Credential rotation determines whether the previously known password remains valid.

Each control therefore addresses a different aspect of the threat.

If only SCRAM-SHA-256 had been retained without changing the known credential, the baseline experiment indicates that the known credential would remain usable.

If only the credential had been changed without considering source restrictions, the service would still remain broadly reachable through the published port.

The combination provides stronger protection against the specific threat scenario.

## 7.14 Security Boundary and Residual Risk

The hardened configuration reduced the demonstrated risk but did not eliminate all possible access paths.

The final Docker HBA rule was based on the source address observed by PostgreSQL. A client that can legitimately reach the service through the same permitted source condition and possesses the current valid credential may still authenticate.

Additionally, Docker networking behaviour can change if the deployment architecture changes.

For example, changing from Docker Desktop to a native Linux Docker environment, changing network modes, introducing a reverse proxy, or modifying routing could change the source address observed by PostgreSQL.

The HBA configuration would then need to be reassessed.

Residual risk therefore remains around credential compromise, trusted-source compromise, network architecture changes, and excessive database privileges.

## 7.15 Comparison With the Initial Security Objective

The project objective was not to make PostgreSQL completely unreachable. Instead, it was to determine whether authentication hardening could prevent the previously successful remote access scenario.

The results demonstrate that this objective was achieved within the tested environment.

The baseline access path was successfully reproduced.

The authentication credential was then rotated.

The HBA policy was restricted based on the observed source.

The current credential remained usable through the permitted path.

The previously known credential was rejected.

A supplementary source-restriction test also demonstrated HBA denial for an unpermitted client source.

The experimental sequence therefore provides a coherent before-and-after demonstration of the security controls.

## 7.16 Observation, Interpretation, Inference and Limitation

To maintain analytical discipline, the principal findings can be separated into four categories.

### Direct Observations

- PostgreSQL used SCRAM-SHA-256.
- PostgreSQL listened on all addresses.
- Docker published TCP port 5432.
- The known credential authenticated during the baseline test.
- PostgreSQL observed `DOCKER_GATEWAY` as the Docker test client's source.
- The previously known credential failed after credential rotation.
- The current credential succeeded through the permitted source.
- An unpermitted source received an HBA denial in the supplementary test.
- Direct-LAN PostgreSQL observed the Windows client's LAN address.

### Interpretations

- The baseline configuration allowed remote authentication using the previously known laboratory credential.
- Credential rotation invalidated the previous authentication path.
- HBA source restrictions were being enforced.
- The initial HBA mismatch resulted from the difference between the external client address and the address observed inside Docker.
- The final HBA rule matched the observed source condition.

### Inferences

- Strong password authentication should be combined with effective credential management.
- Source-based access controls must account for network address translation and routing.
- Defense in depth provides stronger protection than relying on a single authentication control.

### Limitations

- The experiment was conducted in one Docker Desktop environment.
- The primary Docker test involved a specific bridge-network topology.
- The experiment used laboratory credentials and controlled systems.
- The study did not test all possible PostgreSQL authentication mechanisms or privilege configurations.
- The results cannot be assumed to represent every containerized PostgreSQL deployment.

## 7.17 Chapter Summary

The analysis demonstrates that the hardening controls were effective against the specific previously successful credential-based remote access scenario.

The baseline test showed that a known credential could authenticate remotely even though SCRAM-SHA-256 was already enabled.

Credential rotation invalidated that credential.

Restrictive HBA rules provided an additional source-based access boundary, although their correct implementation depended on identifying the source address actually visible to PostgreSQL.

The Docker networking result was particularly significant because PostgreSQL observed the Docker bridge gateway rather than the original external client address.

The combined controls therefore supported the alternative hypothesis for the tested scenario while leaving identifiable residual risks.

The next chapter assesses the security risk and control effectiveness in greater detail and translates the experimental findings into a formal control assessment.

# Chapter 8: Risk and Control Assessment

## 8.1 Introduction

This chapter assesses the security risks identified during the PostgreSQL authentication-hardening experiment and evaluates the effectiveness of the controls implemented.

The assessment is based on the observed results from the primary Docker experiment and the supplementary direct-LAN validation.

The assessment does not assign a universal security rating to PostgreSQL. Instead, it evaluates the controls against the specific threat scenario investigated by this project: an unauthorized remote client attempting to use a previously known PostgreSQL credential.

## 8.2 Risk Assessment Approach

Risk in this project is considered in terms of:

- The asset potentially affected.
- The threat or unwanted event.
- The vulnerability or weakness enabling the event.
- The likelihood of the event under the tested conditions.
- The potential impact.
- The control applied to reduce the risk.
- The residual risk remaining after the control is implemented.

Because this was a controlled laboratory experiment rather than a production risk assessment, likelihood and impact are qualitative rather than based on organizational financial or operational metrics.

The assessment therefore uses the following qualitative categories:

- **Low:** Limited security consequence under the tested conditions.
- **Medium:** Meaningful security exposure requiring corrective controls.
- **High:** Significant exposure that could result in unauthorized access or compromise of an important asset.

## 8.3 Risk 1 — Unauthorized Database Access Using a Known Credential

### Description

A PostgreSQL service that is remotely reachable and accepts a known valid credential may allow an unauthorized client to establish a database session.

### Baseline Evidence

The baseline remote connection successfully authenticated using the previously known laboratory credential.

This demonstrates that the threat condition was achievable in the baseline environment.

### Potential Impact

Successful unauthorized database authentication could allow an attacker to access information available to the compromised database role and potentially perform actions permitted by that role.

The precise impact would depend on the privileges assigned to the role and the sensitivity of the accessible data.

### Baseline Risk Rating

**High**

The rating reflects the fact that the tested condition resulted in successful remote database authentication using a credential assumed to be known by an unauthorized client.

### Control Applied

Credential rotation was applied to invalidate the previously known credential.

SCRAM-SHA-256 remained enabled.

### Residual Risk

The previously known credential no longer authenticated after rotation. However, a client possessing the current valid credential may still authenticate if it also satisfies the applicable HBA rule.

Therefore, credential rotation reduces the specific demonstrated risk but does not eliminate all credential-related risks.

## 8.4 Risk 2 — Broad HBA Source Access

### Description

The baseline HBA host rule permitted connections from all source addresses while requiring SCRAM-SHA-256.

Although authentication was required, the source-address boundary was broad.

### Potential Impact

A broad source rule increases the number of network locations from which clients may reach the PostgreSQL authentication boundary.

This can increase the opportunity for unauthorized authentication attempts when the database service is network reachable.

### Baseline Risk Rating

**Medium**

The rating reflects the broad source scope combined with the requirement for valid authentication.

### Control Applied

The HBA rule was restricted to the source address observed by PostgreSQL in the Docker networking environment.

The final rule required both source-address matching and SCRAM-SHA-256 authentication.

### Residual Risk

The effectiveness of the source restriction depends on the accuracy of the network architecture and the source address visible to PostgreSQL.

A change in Docker networking, routing, or deployment architecture could alter the observed source address and require the HBA policy to be reassessed.

## 8.5 Risk 3 — Network Exposure of PostgreSQL

### Description

The PostgreSQL container published TCP port 5432 through the Windows host.

The PostgreSQL service also listened on all available addresses within the container.

### Potential Impact

A network-reachable database service can receive connection attempts from clients that would not have access to a local-only database service.

The security impact depends on additional controls such as firewalls, network segmentation, HBA rules, and authentication.

### Baseline Risk Rating

**Medium**

The service was intentionally exposed for the laboratory experiment, and successful remote authentication was demonstrated.

### Control Applied

The project did not remove the published port because doing so would prevent the intended remote authentication experiment.

Instead, authentication and HBA controls were strengthened.

### Residual Risk

The database remains network reachable through the laboratory path.

In a production environment, database exposure should be minimized to only the networks and services that require access.

## 8.6 Risk 4 — Incorrect HBA Source Assumption

### Description

An HBA policy may be configured using an expected client address that differs from the address actually observed by PostgreSQL.

This occurred during the experiment because Docker Desktop networking caused PostgreSQL to observe the Docker bridge gateway address rather than the original Kali LAN address.

### Potential Impact

An incorrect source rule can produce two types of security or operational problems.

First, legitimate clients may be denied access because their actual observed source does not match the intended rule.

Second, administrators may misunderstand the actual network trust boundary and therefore implement an ineffective policy.

### Baseline Risk Rating

**Medium**

The risk primarily concerns incorrect security-policy implementation and operational misconfiguration.

### Control Applied

The source address was explicitly verified using PostgreSQL session information.

The HBA policy was then adjusted to match the observed source.

### Residual Risk

The policy remains dependent on the current network topology.

Any change to the Docker networking architecture should trigger reassessment of the source address and HBA rule.

## 8.7 Risk Register

The principal risks are summarized below.

| Risk ID | Risk | Baseline Rating | Control | Residual Risk |
|---|---|---|---|---|
| R-001 | Unauthorized access using known credential | High | Credential rotation + SCRAM-SHA-256 | Medium |
| R-002 | Broad HBA source access | Medium | Restrictive HBA source rule | Low–Medium |
| R-003 | Network exposure of PostgreSQL | Medium | Authentication and HBA controls | Medium |
| R-004 | Incorrect HBA source assumption | Medium | Source-IP verification | Low–Medium |
| R-005 | Credential compromise after hardening | Medium | Credential rotation and secure credential management | Medium |
| R-006 | Network architecture changes invalidate HBA assumptions | Medium | Configuration review and source validation | Medium |

The residual ratings are qualitative assessments specific to the laboratory scenario and are not intended to represent a formal enterprise risk score.

## 8.8 Control 1 — Credential Rotation

Credential rotation was effective against the specific known-credential scenario.

The baseline credential successfully authenticated.

After the credential was changed, the previously known credential failed authentication.

### Control Effectiveness

**Effective**

### Evidence

The baseline remote authentication and post-hardening authentication results provide the before-and-after comparison.

### Security Benefit

The control invalidates credentials that are known or suspected to have been exposed.

### Limitation

Credential rotation does not prevent authentication by an attacker who obtains the new credential.

It should therefore be combined with secure credential storage, appropriate credential lifecycle management, and access restrictions.

## 8.9 Control 2 — SCRAM-SHA-256

SCRAM-SHA-256 was active in both the baseline and hardened configurations.

### Control Effectiveness

**Effective as an authentication mechanism, but insufficient as a standalone control**

The baseline result demonstrates why this distinction is important. A known valid credential successfully authenticated even though SCRAM-SHA-256 was already in use.

### Security Benefit

SCRAM-SHA-256 provides a stronger password authentication mechanism than legacy password approaches and helps protect the authentication exchange.

### Limitation

A strong authentication protocol cannot compensate for possession of a valid credential.

The protocol therefore needs to be combined with credential management and access-control restrictions.

## 8.10 Control 3 — Restrictive HBA Rule

The HBA restriction was effective when the rule was configured according to the source address actually observed by PostgreSQL.

The supplementary direct-LAN experiment also demonstrated that an unpermitted source could be rejected even when the current credential was valid.

### Control Effectiveness

**Effective when correctly configured**

### Security Benefit

The control limits the network sources that can reach the relevant PostgreSQL authentication rule.

### Limitation

The control is dependent on the actual network architecture.

A rule based on an incorrect source assumption can unintentionally deny legitimate clients or fail to represent the intended trust boundary.

## 8.11 Control 4 — Source Address Verification

The use of `inet_client_addr()` provided a practical method for determining the source address PostgreSQL observed.

### Control Effectiveness

**Effective as a validation mechanism**

### Security Benefit

It prevented the HBA policy from being based solely on an assumption about the external client's IP address.

### Limitation

The observed source is specific to the network path used during the test. It should be reassessed when the network architecture changes.

## 8.12 Control 5 — Configuration Verification

The project verified the active HBA configuration after modification and reload.

This distinction between editing a configuration file and confirming the active PostgreSQL configuration is important.

### Control Effectiveness

**Effective**

### Security Benefit

Configuration verification reduces the risk of interpreting an un-applied configuration change as an active security control.

### Limitation

Configuration verification only establishes the state at the time of testing. Continuous configuration monitoring would be required in a production environment.

## 8.13 Defense-in-Depth Assessment

The controls demonstrated greater effectiveness when considered together.

The security sequence can be represented as:

**Network reachability → HBA source restriction → SCRAM-SHA-256 authentication → Credential validity → Database authorization**

Each stage addresses a different condition.

The baseline experiment showed that a known credential could cross the authentication boundary when the service was reachable and the broad HBA rule permitted the connection.

Credential rotation removed the validity of that specific credential.

HBA restriction reduced the permitted source boundary.

SCRAM-SHA-256 remained the authentication mechanism.

This layered approach therefore reduced the specific attack path more effectively than reliance on SCRAM-SHA-256 alone.

## 8.14 Control Dependency

The experiment also identified dependencies between controls.

The HBA restriction depended on accurate identification of the source address observed by PostgreSQL.

The source address depended on the network architecture.

The credential control depended on the credential actually being rotated and not subsequently exposed.

The authentication mechanism depended on PostgreSQL maintaining the intended SCRAM-SHA-256 configuration.

This demonstrates that security controls should not be evaluated entirely in isolation.

## 8.15 Risk Reduction Achieved

The principal demonstrated risk reduction was the removal of the previously successful authentication path.

Before hardening:

**Known credential + reachable service + applicable HBA rule = successful PostgreSQL session**

After hardening:

**Previously known credential + reachable service + applicable HBA rule = password authentication failure**

Separately:

**Current credential + unpermitted source = HBA denial**

And for the intended authorized path:

**Current credential + permitted source = successful PostgreSQL session**

This represents a meaningful reduction in the specific threat scenario while preserving the intended laboratory access path.

## 8.16 Residual Risk Assessment

The hardened environment still contains residual risks.

### Credential Risk

If the current credential is compromised, an attacker may still authenticate if the source satisfies the HBA rule.

### Trusted-Source Risk

If an attacker gains control of a host or network path that appears as an allowed source to PostgreSQL, HBA source restrictions may not provide sufficient protection.

### Configuration Drift

Changes to Docker networking or PostgreSQL configuration could invalidate the tested security assumptions.

### Excessive Privilege

The experiment did not comprehensively evaluate PostgreSQL role privileges. A successfully authenticated role may therefore have permissions that exceed the minimum necessary for a particular application.

### Network Exposure

The database remains reachable through the published port in the laboratory environment.

## 8.17 Risk Treatment Recommendations

Based on the assessment, the following treatment priorities are recommended:

1. Rotate credentials whenever exposure is suspected.
2. Maintain SCRAM-SHA-256 or an appropriate modern authentication mechanism.
3. Restrict HBA rules to only the required database users, databases, and network sources.
4. Verify the source address observed by PostgreSQL before implementing source-based restrictions.
5. Minimize direct exposure of PostgreSQL to unnecessary networks.
6. Apply host-level firewall restrictions in addition to PostgreSQL HBA controls.
7. Use least-privilege database roles.
8. Review HBA configuration whenever container networking changes.
9. Store credentials securely rather than embedding them in source code or public configuration.
10. Monitor authentication failures and unexpected connection attempts in production environments.

## 8.18 Control Assessment Summary

The overall control assessment is summarized below.

| Security Objective | Control | Assessment |
|---|---|---|
| Invalidate known credential | Credential rotation | Effective |
| Maintain strong password authentication | SCRAM-SHA-256 | Effective but not sufficient alone |
| Restrict remote sources | HBA source rule | Effective when correctly mapped |
| Verify actual network source | `inet_client_addr()` | Effective validation technique |
| Confirm configuration activation | HBA reload and inspection | Effective |
| Preserve intended access | Current credential through permitted source | Successful |
| Block previous access path | Previously known credential | Successfully rejected |
| Block unauthorized source | Restrictive HBA condition | Successfully demonstrated |

## 8.19 Overall Risk Conclusion

The control assessment indicates that the implemented hardening measures materially reduced the specific authentication risk investigated by the project.

The strongest evidence is the contrast between successful baseline authentication using the previously known credential and the subsequent password authentication failure after credential rotation.

The HBA tests provide additional evidence that source-based access restrictions can prevent clients outside the permitted boundary from reaching successful authentication.

However, the controls are not independent of the deployment architecture. Docker networking affected the source address visible to PostgreSQL, demonstrating that access-control policies must be validated against the actual network path.

The appropriate conclusion is therefore that the implemented controls were effective against the tested threat scenario, while residual risks remain around credential compromise, trusted-source compromise, configuration changes, network exposure, and database privileges.

## 8.20 Chapter Summary

This chapter assessed the principal risks and controls identified during the experiment.

The highest baseline risk was unauthorized PostgreSQL access using a known valid credential. Credential rotation successfully invalidated that credential.

SCRAM-SHA-256 remained active throughout the experiment and provided the intended password authentication mechanism, but the baseline result demonstrated that strong authentication alone does not prevent use of a valid credential.

Restrictive HBA rules provided an additional source-based security boundary. Their effectiveness depended on correctly identifying the source address visible to PostgreSQL.

Overall, the combination of credential rotation, SCRAM-SHA-256, restrictive HBA rules, and source-address verification reduced the specific unauthorized-access scenario tested while preserving the intended authorized connection path.

The next chapter presents recommendations and an improvement plan based on these findings.


# Chapter 9: Recommendations and Improvement Plan

## 9.1 Introduction

The experimental results and risk assessment demonstrate that PostgreSQL authentication security depends on multiple complementary controls.

The baseline environment allowed remote authentication using a previously known credential. After hardening, credential rotation prevented reuse of that credential, while restrictive HBA rules reduced the permitted source boundary.

The following recommendations are derived from these findings and are intended to improve the security and maintainability of PostgreSQL deployments in containerized environments.

The recommendations are divided into immediate improvements, medium-term improvements, and longer-term improvements.

## 9.2 Recommendation 1 — Rotate Exposed or Previously Known Credentials

The first priority should be to replace any PostgreSQL credential that is known, shared unnecessarily, exposed, or suspected of compromise.

The experiment demonstrated that credential rotation successfully invalidated the previously known credential.

### Recommended Action

Database credentials should be changed when:

- A credential is suspected to have been exposed.
- A credential has been shared outside its intended scope.
- A deployment is transferred between environments.
- A security incident occurs.
- Credentials have reached their defined rotation period.

### Rationale

A strong authentication mechanism cannot prevent an attacker from authenticating when the attacker possesses a valid credential.

Credential lifecycle management is therefore an essential complement to SCRAM-SHA-256.

### Priority

**High**

## 9.3 Recommendation 2 — Maintain SCRAM-SHA-256 Authentication

SCRAM-SHA-256 should remain enabled for password-based PostgreSQL authentication.

The experiment confirmed that SCRAM-SHA-256 was already active in the baseline environment and remained active after hardening.

### Recommended Action

Database administrators should use SCRAM-SHA-256 rather than weaker legacy password authentication mechanisms where password authentication is required.

Existing PostgreSQL roles should periodically be reviewed to ensure that their password verifiers use the intended authentication mechanism.

### Rationale

SCRAM-SHA-256 provides stronger password authentication, but the experiment demonstrates that it should not be treated as a standalone control.

### Priority

**High**

## 9.4 Recommendation 3 — Restrict HBA Rules to Required Sources

HBA rules should be written as narrowly as practical.

The baseline configuration used a broad host source condition. The hardened configuration restricted the relevant source condition.

### Recommended Action

Where possible, HBA rules should specify:

- The required database rather than all databases.
- The required role rather than all users.
- The smallest appropriate source network or host.
- The intended authentication mechanism.

For example, a production policy should avoid broad rules such as:

`host all all all scram-sha-256`

when a narrower rule can satisfy the application's requirements.

### Rationale

Reducing the source and identity scope limits the number of clients and identities that can reach the authentication boundary.

### Priority

**High**

## 9.5 Recommendation 4 — Validate the Source Address Before Writing HBA Restrictions

The experiment demonstrated that the client address expected by the administrator may differ from the address PostgreSQL actually observes.

### Recommended Action

Before implementing source-based HBA restrictions, administrators should:

1. Identify the complete network path.
2. Establish a controlled test connection.
3. Query the source address visible to PostgreSQL.
4. Compare that address with the intended client or network.
5. Implement the HBA rule.
6. Reload the configuration.
7. Test both permitted and denied conditions.

The PostgreSQL function:

`inet_client_addr()`

can be used during controlled validation to identify the source address observed by the server.

### Rationale

The experiment initially produced an HBA denial because PostgreSQL observed the Docker bridge gateway rather than the original client LAN address.

Source validation therefore reduces the risk of incorrect HBA implementation.

### Priority

**High**

## 9.6 Recommendation 5 — Minimize Database Network Exposure

PostgreSQL should not be exposed to networks that do not require database access.

The laboratory environment intentionally published port 5432 because remote connectivity was required for the experiment. In a production deployment, the database should instead be accessible only to the application services and administrative clients that require it.

### Recommended Action

Organizations should consider:

- Binding database ports only where required.
- Restricting access through host firewalls.
- Using private network segments.
- Avoiding unnecessary public exposure.
- Separating application and database networks where practical.
- Restricting administrative access to approved management networks.

### Rationale

Reducing network exposure reduces the number of potential clients that can reach the authentication boundary.

### Priority

**High**

## 9.7 Recommendation 6 — Use Host Firewall Controls in Addition to HBA

HBA should not be treated as the only network-access control.

### Recommended Action

A host firewall or equivalent network-control mechanism should restrict access to PostgreSQL port 5432 to approved sources.

HBA should then provide a second PostgreSQL-level access-control layer.

### Rationale

Defense in depth reduces dependence on a single security control.

If an HBA rule is incorrectly configured or modified, an additional network-level restriction can still reduce exposure.

### Priority

**High**

## 9.8 Recommendation 7 — Review HBA Rules After Network Architecture Changes

The experiment demonstrated that network architecture directly affects HBA source-address interpretation.

### Recommended Action

An HBA review should be included whenever changes are made to:

- Docker networks.
- Container network modes.
- Host networking.
- Routing.
- NAT configuration.
- Reverse proxies.
- Load balancers.
- Network segmentation.
- Container orchestration platforms.

The source address observed by PostgreSQL should be revalidated after significant network changes.

### Rationale

A previously correct HBA rule may become incorrect after a networking change.

### Priority

**Medium–High**

## 9.9 Recommendation 8 — Apply Least Privilege to PostgreSQL Roles

Authentication confirms identity but does not determine whether the authenticated role has excessive privileges.

### Recommended Action

Application services should use dedicated PostgreSQL roles with only the permissions required for their functions.

The default administrative `postgres` role should not be used by application services where a lower-privileged role can perform the required operations.

Administrative roles should be restricted to administrative activities.

### Rationale

If an application credential is compromised, least privilege reduces the potential impact of the compromise.

### Priority

**High**

## 9.10 Recommendation 9 — Separate Application and Administrative Credentials

Different services and administrative users should not unnecessarily share the same PostgreSQL credentials.

### Recommended Action

Organizations should create separate database roles for:

- Application services.
- Database administrators.
- Monitoring systems.
- Backup services.
- Development environments.
- Testing environments.

Credentials should be independently managed and rotated.

### Rationale

Credential separation reduces the blast radius of a credential compromise.

### Priority

**High**

## 9.11 Recommendation 10 — Avoid Storing Credentials in Source Code

Database credentials should not be embedded directly in source code or committed to public repositories.

### Recommended Action

Use an appropriate secrets-management mechanism or protected environment configuration for deployment credentials.

Repositories should also be checked for accidentally committed secrets before publication.

### Rationale

Source repositories can be copied, shared, or exposed independently of the database infrastructure.

Keeping credentials outside source code reduces the likelihood of accidental credential disclosure.

### Priority

**High**

## 9.12 Recommendation 11 — Implement TLS for Sensitive Deployments

The primary experiment demonstrated password authentication behaviour but did not comprehensively evaluate PostgreSQL TLS configuration.

### Recommended Action

Production PostgreSQL deployments carrying sensitive traffic should evaluate TLS configuration and certificate validation appropriate to their architecture.

Where client certificate authentication is appropriate, it should be considered as an additional authentication option.

### Rationale

Encryption protects database traffic in transit and can provide stronger trust relationships between clients and servers.

### Priority

**Medium–High**

## 9.13 Recommendation 12 — Monitor PostgreSQL Authentication Activity

Authentication failures can provide useful indicators of credential misuse or unauthorized connection attempts.

### Recommended Action

Production deployments should monitor:

- Failed authentication attempts.
- Repeated attempts against administrative accounts.
- Unexpected source addresses.
- Configuration changes.
- Database role changes.
- Unusual connection patterns.

Relevant PostgreSQL logs should be integrated with centralized monitoring or SIEM infrastructure where appropriate.

### Rationale

Preventive controls reduce the likelihood of unauthorized access, while monitoring improves the ability to detect attempted or successful misuse.

### Priority

**Medium–High**

## 9.14 Recommendation 13 — Validate Security Configuration After Deployment

Security configuration should be tested after deployment rather than assumed to be correct because configuration files contain the expected values.

### Recommended Action

A deployment validation process should include:

1. Checking PostgreSQL listening configuration.
2. Checking published network ports.
3. Reviewing active HBA rules.
4. Confirming authentication mechanisms.
5. Testing an authorized connection.
6. Testing an unauthorized source.
7. Testing an invalid or revoked credential.
8. Recording the results.

### Rationale

The experiment demonstrated that the active PostgreSQL configuration and the expected network behaviour must be verified directly.

### Priority

**High**

## 9.15 Recommendation 14 — Maintain Configuration Documentation

PostgreSQL authentication and network policies should be documented alongside the infrastructure configuration.

### Recommended Action

Documentation should identify:

- Approved database clients.
- Approved network sources.
- Database roles.
- Authentication mechanisms.
- HBA rules.
- Network architecture.
- Credential-management procedures.
- Configuration-review requirements.

### Rationale

Clear documentation reduces configuration drift and makes future troubleshooting and security reviews more reliable.

### Priority

**Medium**

## 9.16 Recommended Improvement Roadmap

The recommendations can be organized into a phased improvement plan.

| Timeframe | Improvement | Priority |
|---|---|---|
| Immediate | Rotate known or exposed credentials | High |
| Immediate | Maintain SCRAM-SHA-256 | High |
| Immediate | Restrict HBA source rules | High |
| Immediate | Verify PostgreSQL source addresses | High |
| Immediate | Remove unnecessary database exposure | High |
| Immediate | Apply least-privilege database roles | High |
| Short term | Add host/network firewall restrictions | High |
| Short term | Separate application and administrative credentials | High |
| Short term | Establish configuration validation procedures | High |
| Short term | Review HBA rules after network changes | Medium–High |
| Medium term | Implement centralized authentication monitoring | Medium–High |
| Medium term | Evaluate TLS and certificate-based authentication | Medium–High |
| Medium term | Implement stronger secrets management | High |
| Long term | Automate configuration and security validation | Medium |
| Long term | Integrate database security monitoring with SIEM | Medium |

## 9.17 Recommended Secure Configuration Model

Based on the findings, a more secure containerized PostgreSQL deployment should follow a layered model.

### Layer 1 — Network Restriction

Only approved application and administrative networks should be able to reach PostgreSQL.

### Layer 2 — Docker Network Segmentation

Database containers should communicate over dedicated internal networks where appropriate, rather than unnecessarily exposing the database to external networks.

### Layer 3 — HBA Restriction

HBA rules should specify the narrowest practical combination of database, role, and source network.

### Layer 4 — Strong Authentication

SCRAM-SHA-256 should be maintained for password-based authentication.

### Layer 5 — Credential Management

Credentials should be unique, securely stored, rotated when necessary, and excluded from source repositories.

### Layer 6 — Least Privilege

Database roles should have only the permissions necessary to perform their intended functions.

### Layer 7 — Monitoring

Authentication and configuration events should be logged and monitored for suspicious behaviour.

This model provides defense in depth and reduces dependence on any individual control.

## 9.18 Improvement Plan for the Tested Environment

For the specific laboratory environment used in this project, the following actions are recommended:

1. Retain the final SCRAM-SHA-256 authentication configuration.
2. Keep the laboratory credential separate from repository documentation.
3. Retain the restrictive HBA rule appropriate to the observed Docker networking path.
4. Revalidate the observed source address if the Docker network is changed.
5. Avoid publishing PostgreSQL beyond the network required for testing.
6. Use a dedicated non-administrative database role for application testing where possible.
7. Maintain the evidence and test matrix for future configuration changes.
8. Repeat the authentication tests after significant infrastructure changes.
9. Remove temporary testing resources when they are no longer required.
10. Preserve only the evidence necessary to demonstrate the security findings.

## 9.19 Expected Benefits

Implementing these recommendations would provide several security benefits.

### Reduced Credential Risk

Credential rotation and secure secrets management reduce the risk associated with known or exposed passwords.

### Reduced Network Exposure

Network and HBA restrictions reduce the number of clients capable of reaching the database authentication boundary.

### Stronger Authentication

SCRAM-SHA-256 maintains a modern password authentication mechanism.

### Reduced Blast Radius

Least-privilege roles and credential separation limit the impact of compromised credentials.

### Improved Detection

Authentication monitoring can identify suspicious activity that preventive controls do not block.

### Improved Configuration Reliability

Source-address validation and configuration review reduce the likelihood of incorrect HBA policies after network changes.

## 9.20 Recommendation Prioritization

The highest-priority improvements are those that directly address the demonstrated threat scenario.

The first priority is credential management because the baseline experiment showed that possession of a valid credential was sufficient to authenticate remotely.

The second priority is source restriction because the baseline HBA policy had a broad source scope.

The third priority is network exposure reduction because limiting who can reach PostgreSQL reduces the opportunities for authentication attempts.

These controls should be implemented together rather than viewed as substitutes for one another.

## 9.21 Chapter Summary

This chapter translated the experimental findings into practical security recommendations.

The highest-priority recommendations are to rotate known or exposed credentials, retain SCRAM-SHA-256, restrict HBA rules, verify the source address visible to PostgreSQL, minimize network exposure, apply least privilege, and protect database credentials through appropriate secrets-management practices.

The experiment demonstrated that the effectiveness of HBA restrictions depends on the actual network architecture. Consequently, HBA policies should be reviewed whenever Docker networking or other routing components change.

The recommended improvement plan applies defense in depth by combining network restriction, HBA controls, strong authentication, credential management, least privilege, and monitoring.

The next chapter discusses the limitations of the study and identifies opportunities for future research.


# Chapter 10: Limitations and Future Work

## 10.1 Introduction

This chapter identifies the limitations of the experimental design and defines areas for future investigation.

The findings of this project are based on a controlled laboratory environment and should therefore be interpreted within the specific technical and networking conditions under which the experiment was conducted.

Identifying these limitations is important because it prevents the results from being generalized beyond what the collected evidence supports.

## 10.2 Docker Desktop Environment

The primary experiment was conducted using Docker Desktop on a Windows host.

Docker Desktop introduces a networking architecture that differs from a native Linux Docker deployment. In the tested environment, PostgreSQL observed the Docker bridge gateway as the client source rather than the original external client LAN address.

The results may therefore differ in environments using native Linux Docker, alternative container networking modes, Kubernetes, cloud container platforms, or different routing architectures.

## 10.3 Limited PostgreSQL Version Coverage

The primary experiment used PostgreSQL 15 through the `postgres:15-alpine` container image.

The supplementary direct-LAN validation used PostgreSQL 18.4 on Kali Linux.

The study does not establish that identical behaviour will occur across every PostgreSQL release.

Future research could repeat the experiment across multiple supported PostgreSQL versions to determine whether authentication and HBA behaviour remains consistent.

## 10.4 Limited Authentication Scope

The project focused on password authentication using SCRAM-SHA-256.

Other PostgreSQL authentication mechanisms were not comprehensively tested.

Future work could compare SCRAM-SHA-256 with additional authentication approaches, including certificate-based authentication, depending on the deployment requirements.

Such an investigation could evaluate whether stronger client identity verification provides additional protection beyond password authentication.

## 10.5 Limited Authorization Testing

The project primarily examined authentication and HBA access control.

It did not comprehensively test the privileges available to the `postgres` role or evaluate a complete least-privilege role design.

As a result, the experiment demonstrates whether a client can establish a database session but does not establish the full security impact of every action that an authenticated role could perform.

Future research should evaluate dedicated application roles and PostgreSQL privileges to determine how least privilege affects the impact of credential compromise.

## 10.6 Limited Network Topologies

The experiment examined two principal network paths:

1. A Docker Desktop published-port path.
2. A direct LAN connection to PostgreSQL running on Kali Linux.

Other network architectures were not tested.

Future work could examine:

- Native Linux Docker bridge networking.
- Docker host networking.
- Routed container networking.
- Kubernetes networking.
- Reverse proxies.
- Load balancers.
- Network firewalls.
- VPN-based administrative access.
- Cloud-hosted PostgreSQL environments.

Comparing these environments would provide a broader understanding of how source-address visibility affects HBA policy design.

## 10.7 Controlled Credential Scenario

The experiment assumed that the previously known credential was available to the modeled unauthorized client.

The project did not investigate how the credential was obtained.

No password cracking, credential harvesting, or credential theft was performed.

This limitation was intentional because the objective was to evaluate the effectiveness of authentication hardening rather than demonstrate methods for obtaining credentials.

Future work could use simulated credential-compromise scenarios to evaluate credential lifecycle controls without exposing real credentials.

## 10.8 Limited Duration of Observation

The experiment represents the PostgreSQL security state at the time of testing.

It does not evaluate long-term configuration drift, repeated administrative changes, or continuous authentication activity.

Future research could introduce continuous monitoring and repeated validation to determine whether the hardened configuration remains effective over time.

## 10.9 No Production Workload

The experiment was conducted in a controlled laboratory environment rather than a production application environment.

Consequently, the study does not measure the effect of the hardening controls on production application availability, database performance, or operational workload.

Future research could evaluate the same controls in a representative staging environment containing realistic application traffic.

## 10.10 No Comprehensive Firewall Assessment

The project examined PostgreSQL and Docker-level access controls but did not conduct a comprehensive host firewall or enterprise network firewall assessment.

The published PostgreSQL port remained available because remote connectivity was required for the experiment.

A production security assessment should evaluate PostgreSQL HBA together with host-level and network-level firewall controls.

Future work could measure how combining firewall rules with HBA restrictions affects the attack surface.

## 10.11 Evidence Limitations

The evidence collected for the project consists primarily of screenshots and configuration outputs captured during the testing process.

Although these provide direct evidence of the observed states, they represent specific test moments rather than continuous system telemetry.

Future work could improve evidence collection by incorporating:

- Timestamped PostgreSQL logs.
- Centralized SIEM events.
- Network packet captures.
- Automated configuration snapshots.
- Cryptographic hashes for important evidence files.
- Automated test reports.

These additions could strengthen reproducibility and evidence integrity.

## 10.12 Generalizability

The findings support conclusions about the specific laboratory configuration tested.

The experiment demonstrates that credential rotation and restrictive HBA rules can prevent the tested previously known credential scenario when correctly implemented.

However, the results should not be interpreted as proof that all containerized PostgreSQL deployments will behave identically.

Network topology, PostgreSQL version, container runtime, routing, firewall configuration, and authentication architecture can all affect the outcome.

The conclusions are therefore bounded by the environment and conditions documented in this report.

## 10.13 Future Research Direction 1 — Automated Security Validation

A useful extension would be to automate the authentication-hardening validation process.

An automated test suite could verify:

1. PostgreSQL listening configuration.
2. Published database ports.
3. Active HBA rules.
4. Authentication mechanisms.
5. Authorized connection behaviour.
6. Rejected credential behaviour.
7. Rejected source-address behaviour.

This would allow configuration changes to be tested consistently after deployment.

## 10.14 Future Research Direction 2 — Infrastructure-as-Code Security Controls

Future work could integrate PostgreSQL security controls directly into infrastructure-as-code workflows.

For example, automated checks could verify that:

- PostgreSQL ports are not unnecessarily exposed.
- HBA rules do not contain unnecessarily broad source ranges.
- Application roles follow least privilege.
- Credentials are not hard-coded.
- Secure authentication mechanisms are enabled.

This could move the security controls from manual validation into repeatable DevSecOps processes.

## 10.15 Future Research Direction 3 — SIEM Integration

Future research could examine how PostgreSQL authentication events can be integrated with a SIEM platform.

The objective would be to detect:

- Repeated failed authentication attempts.
- Unexpected source addresses.
- Attempts against administrative accounts.
- Configuration changes.
- Successful authentication from unusual sources.

This would extend the project from preventive controls into detection and response.

## 10.16 Future Research Direction 4 — Comparative Container Networking

The Docker source-address finding provides a strong basis for comparative networking research.

Future work could repeat the same authentication experiment under different container networking architectures and record the source address observed by PostgreSQL in each case.

The resulting comparison could determine how:

- Bridge networking.
- Host networking.
- Routed networking.
- Kubernetes networking.
- Cloud container networking.

affect PostgreSQL HBA policy design.

## 10.17 Future Research Direction 5 — Least-Privilege Evaluation

A future experiment could replace the administrative PostgreSQL role used for laboratory validation with a dedicated application role.

The research could then compare the potential impact of credential compromise under:

- Administrative privileges.
- Application-level privileges.
- Read-only privileges.
- Restricted database-specific privileges.

This would extend the current research from authentication prevention to post-authentication risk reduction.

## 10.18 Future Research Direction 6 — TLS and Certificate Authentication

A further extension could evaluate encrypted PostgreSQL connections and certificate-based client authentication.

Such a study could examine whether combining:

**Network restriction + HBA + SCRAM-SHA-256 + TLS + client certificates**

provides measurable security improvements compared with password authentication alone.

This would provide a broader assessment of PostgreSQL authentication architectures.

## 10.19 Chapter Summary

This project has several limitations, primarily arising from its controlled laboratory environment, specific Docker Desktop networking architecture, limited PostgreSQL version coverage, and focus on authentication rather than comprehensive authorization.

The most significant limitation is that the source address observed by PostgreSQL depends on the network architecture. The result obtained through Docker Desktop may therefore differ from other containerized deployments.

Despite these limitations, the experiment provides a reproducible demonstration that credential rotation and correctly configured HBA restrictions can prevent the specific previously successful credential-based access scenario tested.

Future work could extend the research to additional container networking models, PostgreSQL versions, authentication mechanisms, least-privilege configurations, firewall controls, SIEM monitoring, and automated DevSecOps validation.



# Conclusion

This project evaluated the extent to which SCRAM-SHA-256 authentication and restrictive PostgreSQL `pg_hba.conf` rules can prevent unauthorized remote database connections using a previously known credential in a containerized environment.

The investigation used a controlled before-and-after experimental design. The primary experiment was conducted against a PostgreSQL 15 container running within the authorized Docker Desktop environment.

The baseline results demonstrated that PostgreSQL was network reachable through the Docker-published PostgreSQL port and that the previously known laboratory credential successfully established a remote database session. Importantly, SCRAM-SHA-256 was already enabled during the baseline phase. This demonstrated that a strong password authentication mechanism does not by itself prevent unauthorized access when an attacker possesses a valid credential.

The hardening phase introduced credential rotation while retaining SCRAM-SHA-256. The previously known credential was subsequently rejected with a password authentication failure. This demonstrated that credential rotation successfully invalidated the specific authentication path that had been demonstrated during the baseline test.

The investigation also demonstrated the importance of source-address-aware HBA configuration. The external Kali Linux client reached PostgreSQL through the Windows host and Docker networking, but PostgreSQL observed the Docker bridge gateway address rather than the original client LAN address. An initial HBA restriction based on the external LAN subnet therefore produced an HBA denial. After the rule was adjusted to match the source address observed by PostgreSQL, the authorized current credential successfully authenticated.

The supplementary direct-LAN experiment provided additional validation of this finding. When PostgreSQL directly received a connection from the Windows host, it observed the Windows host's actual LAN address. A restrictive HBA rule could then deny the connection when that source was outside the permitted policy.

The results support the alternative hypothesis for the specific threat scenario tested:

**Changing the known PostgreSQL credential while retaining SCRAM-SHA-256 authentication and applying restrictive `pg_hba.conf` rules prevented unauthorized remote PostgreSQL connections using the previously known credential in the tested containerized environment.**

The findings do not demonstrate that all unauthorized access to PostgreSQL is impossible. A client possessing the current valid credential and satisfying the permitted HBA source condition may still authenticate. Changes to Docker networking may also change the source address observed by PostgreSQL. Additional controls such as network firewalls, network segmentation, least-privilege roles, secure secrets management, TLS, and authentication monitoring would therefore be appropriate in a production environment.

Overall, the project demonstrates that PostgreSQL authentication hardening is most effective when implemented as a layered security strategy. SCRAM-SHA-256 provides strong password authentication, credential rotation invalidates previously known credentials, and restrictive HBA rules reduce the permitted source boundary. The effectiveness of these controls depends on correct implementation and an accurate understanding of the underlying network architecture.

The project therefore concludes that authentication hardening materially reduced the specific unauthorized-access risk investigated while preserving the intended authorized connection path within the laboratory environment.



# References

1. PostgreSQL Global Development Group. *PostgreSQL Documentation: Client Authentication*. PostgreSQL Documentation.

2. PostgreSQL Global Development Group. *PostgreSQL Documentation: The pg_hba.conf File*. PostgreSQL Documentation.

3. PostgreSQL Global Development Group. *PostgreSQL Documentation: Authentication Methods*. PostgreSQL Documentation.

4. PostgreSQL Global Development Group. *PostgreSQL Documentation: Secure TCP/IP Connections with SSL*. PostgreSQL Documentation.

5. Docker. *Docker Documentation: Networking Overview*. Docker Documentation.

6. Docker. *Docker Documentation: Bridge Network Driver*. Docker Documentation.

7. OWASP Foundation. *OWASP Authentication Cheat Sheet*. OWASP.

8. OWASP Foundation. *OWASP Network Segmentation Cheat Sheet*. OWASP.

9. National Institute of Standards and Technology. *Digital Identity Guidelines: Authentication and Lifecycle Management*. NIST Special Publication 800-63B.

10. Center for Internet Security. *CIS Benchmarks*. Center for Internet Security.

11. PostgreSQL Global Development Group. *PostgreSQL Documentation: Database Roles*. PostgreSQL Documentation.

12. PostgreSQL Global Development Group. *PostgreSQL Documentation: Database Access Privileges*. PostgreSQL Documentation.


# Appendices

## Appendix A — Evidence Index

The evidence index provides the traceability relationship between each collected evidence item and the corresponding experimental test.

The complete evidence index is maintained in:

`Evidence/evidence-index.md`

The evidence set contains 15 retained evidence items covering the baseline Docker configuration, post-hardening configuration, and supplementary direct-LAN validation.

### Evidence Categories

| Category | Evidence IDs | Purpose |
|---|---|---|
| Baseline | E-001 to E-005 | Establish initial PostgreSQL configuration and successful remote access |
| Hardening | E-006 to E-012 | Document SCRAM verification, HBA changes, reload, and final state |
| Network Validation | E-013 to E-015 | Validate direct-LAN source visibility, credential rejection, and HBA denial |

The evidence does not intentionally contain actual authentication secrets in the final submission.

---

## Appendix B — Test Matrix

The complete test matrix is maintained in:

`Evidence/test-matrix.md`

The matrix links each test condition to its expected result, observed result, evidence identifier, and interpretation.

The primary test sequence covers:

- Baseline remote authentication.
- Baseline HBA configuration.
- PostgreSQL listening configuration.
- Docker port exposure.
- Credential rotation.
- SCRAM-SHA-256 verification.
- HBA configuration reload.
- HBA restriction.
- Active HBA verification.
- Authorized post-hardening authentication.
- Final HBA verification.
- Supplementary direct-LAN source validation.
- Previously known credential rejection.
- Supplementary HBA denial.

---

## Appendix C — Authentication Test Procedure

The complete reproducible test procedure is maintained in:

`tests/postgres-auth-tests.md`

The procedure documents the test sequence, commands, expected results, observed results, evidence references, and interpretation guidelines.

Authentication secrets are intentionally excluded from the documented procedure.

---

## Appendix D — Hardened PostgreSQL Security Policy

The implemented security policy is maintained in:

`src/hardened-postgresql-policy.md`

The policy defines the intended authentication mechanism, credential requirements, HBA source restrictions, validation requirements, and security boundaries for the laboratory environment.

---

## Appendix E — Project Repository Structure

The final repository is organized as follows:

```text
week1/
├── README.md
├── AI-use-record.md
├── capstone-proposal.md
├── docker-compose.yml
├── .gitignore
│
├── Evidence/
│   ├── BEFORE/
│   ├── AFTER/
│   ├── Network Validation/
│   ├── evidence-hashes.txt
│   ├── evidence-index.md
│   └── test-matrix.md
│
├── analysis/
│   ├── before-after-comparison.md
│   ├── control-effectiveness.md
│   ├── findings-summary.md
│   └── results-analysis.md
│
├── diagrams/
│   ├── 01-System Architecture.png
│   ├── 02-BeforeAfter Authentication Flow.png
│   └── 03-Layered Security Controls.png
│
├── report/
│   ├── final-report.md
│   └── My Final Presentation.pptx
│
├── src/
│   └── hardened-postgresql-policy.md
│
└── tests/
    └── postgres-auth-tests.md
```

The credential-containing screenshot used during the local procedure is intentionally excluded from the repository and final submission.

