# AI Use Record

## Purpose

AI assistance was used as a supporting tool during the development, testing, documentation, and review of this cybersecurity final project.

AI-generated suggestions were treated as assistance rather than authoritative evidence. Technical claims and configuration changes were independently tested in the authorized local/classroom environment.

## AI Tool

- Tool: ChatGPT
- Use: Technical guidance, troubleshooting, documentation support, evidence organization, test-matrix development, and interpretation of observed results.

## AI-Assisted Activities

| Activity | Representative Prompt / Request | AI Output Summary | Verification / Independent Check | Result / Limitation |
|---|---|---|---|---|
| Project methodology | Asked for help structuring the PostgreSQL authentication-hardening experiment and aligning it with the project requirements. | Suggested a baseline → hardening → validation methodology and separation of primary and supplementary testing. | Compared the proposed structure with the project guide and implemented the tests in the authorized environment. | The methodology was adapted to the actual environment and observed results. |
| PostgreSQL authentication | Asked how PostgreSQL `pg_hba.conf` and `SCRAM-SHA-256` should be used in the experiment. | Explained HBA as the PostgreSQL host-based authentication control and described how source addresses and authentication methods affect connections. | Verified configuration using PostgreSQL commands and `pg_hba_file_rules`. | AI explanations were checked against actual PostgreSQL behavior. |
| Docker networking | Asked why PostgreSQL in Docker observed `DOCKER_GATEWAY` instead of the original client address. | Explained that Docker Desktop networking/NAT can cause PostgreSQL to observe the Docker bridge gateway as the connection source. | Queried `inet_client_addr()` from the PostgreSQL session and performed supplementary direct-LAN testing. | The observed Docker source address confirmed the networking limitation. |
| Credential hardening | Requested guidance for changing the PostgreSQL credential and verifying the authentication mechanism. | Provided commands for changing the PostgreSQL role credential and checking the password verifier. | Executed the commands against the authorized PostgreSQL environment. | The role verifier remained `SCRAM-SHA-256`. |
| HBA restriction testing | Requested a controlled method to demonstrate HBA-based source-address denial. | Suggested temporarily restricting the allowed source address and testing from Windows. | Changed the Kali HBA rule, reloaded PostgreSQL, and attempted the authorized connection from Windows. | PostgreSQL returned `no pg_hba.conf entry` for the Windows source address. The temporary rule was subsequently restored. |
| Evidence organization | Asked for help organizing screenshots and mapping them to project tests. | Suggested separate BEFORE, AFTER, and Network Validation evidence categories and an evidence index. | Verified the actual evidence files and directory contents using Windows CMD. | Evidence was organized into the project evidence structure. |
| Test matrix | Requested assistance creating a test matrix from the completed experiments. | Produced test IDs, expected results, observed results, evidence references, and interpretations. | Compared the matrix against actual commands, outputs, and captured evidence. | Matrix entries were adjusted to distinguish primary Docker results from supplementary Kali validation. |
| Repository documentation | Requested help creating the README and evidence documentation. | Generated documentation covering scope, environment, test procedure, evidence, limitations, and safety boundaries. | Reviewed the content against the project requirements and actual test environment. | Documentation was adapted to avoid unsupported claims and disclosure of credentials. |

## Important Technical Refinements

AI assistance initially suggested that source-address HBA testing could directly use the Windows LAN address in the Docker PostgreSQL configuration.

Testing showed that Docker Desktop networking caused PostgreSQL to observe the published connection as originating from the Docker bridge gateway (`DOCKER_GATEWAY`).

The methodology was therefore refined:

1. The Docker PostgreSQL environment remained the primary experiment.
2. Docker-observed source addressing was documented as a limitation.
3. A separate direct-LAN PostgreSQL instance on Kali was used for supplementary source-IP and HBA validation.
4. The supplementary test confirmed that PostgreSQL could observe the actual Windows source address (`WINDOWS_HOST`) without the Docker networking layer.
5. The temporary Kali HBA restriction was restored after testing.

This refinement demonstrates that AI-generated technical guidance was not accepted without independent validation.

## Verification Approach

Technical recommendations from AI were checked through:

- PostgreSQL command-line output.
- PostgreSQL configuration inspection.
- `pg_hba_file_rules` queries.
- Authentication success/failure responses.
- `inet_client_addr()` results.
- Docker networking inspection.
- Direct Windows-to-Kali connectivity tests.
- Captured screenshots.

## AI Limitations

AI assistance can provide technically plausible commands or explanations that may not match a specific environment.

For this project, differences in:

- Docker Desktop networking,
- operating-system networking,
- PostgreSQL versions,
- PostgreSQL configuration,
- container networking,

required independent testing and adjustment.

Therefore, observed command output and captured evidence were treated as the authoritative basis for project findings.

## Data Protection

Passwords, private keys, access tokens, API keys, and other secrets were not intentionally included in this AI-use record.

Lab-only network addresses may appear where necessary to explain the experimental results.

## Final Statement

AI was used as a technical support and documentation aid. The final experimental conclusions were based on independently executed tests and observed evidence from the authorized project environment rather than on AI output alone.