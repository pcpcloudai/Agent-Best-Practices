# ADR-003: Risk-Based Security for Azure Container Registry

- **Status:** Accepted
- **Scope:** Customer side of the shared responsibility model (Microsoft platform responsibilities are out of scope)
- **Architecture:** ![Architecture](https://raw.githubusercontent.com/4min4yed/Cloud-Computing/fc591153d0187e625ecd6bd286037032b5ab60d0/Microsoft%20Azure/Images/ADR_ACR_ARCH.jpg)

## Context

Agents and their workloads rely heavily on containers and OCI artifacts stored in Azure Container Registry (ACR). Compromising ACR compromises the entire deployment chain: a tampered image, a leaked push credential, or a deleted repository can affect every environment that consumes it. "ACR is secure by default" is the first assumption to reject : Microsoft's shared responsibility model leaves a substantial attack surface on our side.

The main risk this ADR is about is artifact integrity across the lifecycle: an attacker who can push, mutate, or replace an image, or who holds a credential with broad registry access, must not be able to get a malicious or unauthorized artifact deployed, nor to destroy the last trusted copy.

This ADR covers our side of the shared responsibility model only; Microsoft's platform-side responsibilities are out of scope.

## Methodology

The adopted methodology of producing this architecture is based on a risk management methodology:

- identifying the operational procedures (what ACR does?) and assets involved in those procedures (ACR does it with what/to what?) as well as classifying them based on their security properties,
- then the risks related to those aspects (what could go wrong ?)
- and following the SWIFT (Structured What-If Technique) to uncover more potential risks,
- and finally addressing the identified risks with appropriate security controls in the architecture, which follows a zero-trust approach.

## Decision Drivers

- Zero-trust, least-privilege access: no identity gets more registry access than its job requires.
- No long-lived credentials. Every authentication path is identity-based and short-lived.
- Artifact protection applies across its whole lifecycle: build, publish, store, deploy.
- Recovery from destructive events must be possible independent of the primary registry.
- Cost and operational complexity must stay proportionate to the risk.

## Options Considered

### Network Exposure

1. **Private Link + dedicated data endpoints, public access disabled**: \
registry reachable only through a private endpoint in an approved VNet; eliminates the public attack surface. \
**Chosen** because it shrinks the attack surface the most.

2. **Public access with IP allowlist** : simpler, but leaves the registry publicly resolvable. \
**Rejected** : expands attack surface; violates driver 1. Reconsider only if a service genuinely cannot be placed in a VNet.

3. **Trusted-services-only exception** : public endpoint limited to Azure trusted services. \
**Deferred** : side feature; reconsider only if an Azure service must reach a private endpoint.

### Authentication

1. **Microsoft Entra ID with managed identities / workload identity federation (OIDC)** : managed identity for Azure-hosted workloads, federated credentials for GitHub Actions and Azure  DevOps. No stored secrets, short-lived tokens. \
**Chosen** : eliminates long-lived credentials on every path; satisfies driver 2

2. **Service principal with client secret** : universally supported but secrets are long-lived and leak into logs and code. \
**Rejected** : violates driver Reconsider only for a legacy integration that cannot federate, with fixed rotation and a named owner.

3. **ACR admin account** : shared registry-wide credential with no accountability. \
**Rejected** : violates driver 2; disabled by default in this decision.

### Authorization

1. **ABAC repository permissions (`Container Registry Repository Reader / Writer / Contributor`)** : lea st privilege at repository granularity; separates pull, push, and delete. \
**Chosen** : most granular control; satisfies driver 1.

2. **Legacy RBAC at registry scope** : simpler, but any push role holder can push to every repository. \
**Rejected** : violates driver 1.

### Artifact Integrity

1. **Artifact signing + verification at admission**: images are signed at build with Notation, and signatures are verified at deploy by an admission controller (Ratify on AKS). \
Notation is Microsoft's supported signing tool for ACR and integrates with Azure Key Vault for key custody. \
Ratify runs as a cluster add-on and rejects images whose signature is missing, expired, or from an untrusted signer. \
Signing alone proves who signed; admission verification is what actually blocks an unsigned image from deploying. \
Short-lived signing certificates are used with timestamping so signatures remain verifiable even after the certificate expires.

**Chosen** : signing alone does not stop an unsigned image from deploying; admission verification is the only enforceable control for driver 3.

2. **Docker Content Trust (DCT)** : cannot be enabled on new registries after 31 May 2026 and is removed 31 March 2028. \
**Rejected** : not a future proof solution.

3. **Tag immutability only** : tags are mutable references and prove nothing about authenticity. \
**Rejected** : cannot satisfy driver 3 without signature verification.

### Encryption at Rest

1. **Microsoft-managed keys** : platform-managed default, no key-lifecycle burden. \
**Chosen** : sufficient absent a regulatory/contractual key-control requirement; defers complexity without weakening the drivers.

2. **Customer-managed keys** : Key Vault / HSM-backed keys; adds availability, rotation, and recovery dependencies and is creation-time-only on Premium SKU. \
**Deferred** : reconsider if a regulation, customer, or organizational policy requires it.

## Decision

Option set chosen: private networking, identity-based authentication, repository-scoped ABAC authorization, deployment-time signature verification, digest-pinned immutable artifacts, and independent recovery : all as the default, defined in IaC.

- **Network exposure.** Every ACR instance has public network access set to Disabled and is reachable only through a Private Link private endpoint in an approved VNet. \
Private endpoints enable the dedicated data endpoints (`myregistry.<region>.data.azurecr.io`) \
all pull and push paths use the private endpoint. \
Public `*.azurecr.io` and `*.blob.core.windows.net` data paths are not reachable from the internet.
- **Authentication.** Every authentication path uses Microsoft Entra ID. Azure-hosted workloads use managed identity; GitHub Actions and Azure DevOps use workload identity federation (OIDC). 
- **Authorization.** Registries are created or migrated with ABAC Repository Permissions.The only permitted data-plane roles, assigned at the narrowest repository scope via ABAC conditions:
  - `Container Registry Repository Reader` : pullers.
  - `Container Registry Repository Writer` : publishers.
  - `Container Registry Repository Contributor` : break-glass and repository administration only, never normal workloads.
  - Legacy `AcrPull` / `AcrPush` / `AcrDelete` are not assigned to new identities; existing assignments are migrated before the ABAC cutover, not after.
  - *Break-glass with PIM*. The Container Registry Repository Contributor role is never assigned as permanent active role. It is assigned as eligible through Microsoft Entra Privileged Identity Management (PIM), with a maximum activation window, mandatory justification... with activation requiring MFA . and every activation and use is logged to the same Log Analytics workspace as registry diagnostics, with alerts on any activation outside a declared incident.
- **Artifacts Management.** Pull consumers get `Reader` on only the repository or prefix they consume, with no write/delete. \
Publishers get `Writer` on only the repositories they own, with no delete and no control-plane rights. \
Deletion is performed only by a separate, non-pipeline identity holding `Contributor` on the target repository, with approval.
- **Transport.** TLS 1.2+ for all registry communication; plaintext HTTP is blocked by client configuration and network policy. Key Vault and signing-service connections use TLS with managed identity or federated credentials; no signing or key-access secret traverses an unencrypted channel.
- **Encryption at rest.** Microsoft-managed keys remain enabled as the default. The architecture preserves the ability to adopt CMK without redesign if a regulatory or contractual requirement appears.
- **Artifact identification.** Production deployments reference images by digest (`<repository>@sha256:<digest>`), never by mutable tag. \
Production repositories are locked against:
- tag mutation using ACR repository attributes. \
- Deletion by only allowing Container Registry Repository Contributor role at the target repository, held only by the approved deleter identity, and is  protected by CanNotDelete resource locks and ACR soft-delete.
- **Supply chain.** Microsoft Defender for Cloud image assessment is enabled on every registry. \
Production promotion is blocked when an image violates the organization's vulnerability threshold.
- **Monitoring.** Every registry authentication and artifact data and control plane logs to a Log Analytics workspace. \
Alerts (each with a named owner) get triggered: authentication failure from unexpected identities, regions, networks..; push/delete activity from  unapproved publisher/deleter set...
Retention meets the incident response requirements and does not rely on ACR default retention.
- **Recovery.** A redundant registry acts as a vault outside the primary's blast radius: an automated pipeline backs up critical production images, signatures... to another registry. \
Restore tests run on a defined schedule to verify integrity and availability. \
Geo-replication is for regional availability only and is never treated as backup.
The vault registry has its own separate Contributor identity.



## Consequences

### Gains

- No secret utilization in ACR shrinks credential-leak blast radius.
- An external network scan finds no reachable registry endpoint.
- A compromised push alone cannot deploy a malicious image, because admission rejects artifacts with  unsigned or untrusted-signer.
- A compromised pull identity cannot push, and a compromised push identity cannot delete production repositories.
- Every mutation is tied to an Entra principal and a source network, with alerts on out-of-pattern events.
- A deleted or ransomware-affected registry does not eliminate the last trusted copy of production artifacts.

### Trade-offs

- More configuration complexity: private DNS, VNet peering, per-identity access.
- The ABAC migration breaks existing `AcrPull`/`AcrPush` assignments and dependent pipelines : hence the migrate-before-cutover requirement.
- Signing and admission add build-time dependencies: certificate lifecycle, admission-controller installation, policy maintenance.
- Backups adds storage cost and a recurring test overhead.
- The contributor supposedly responsible for deletion only, has write access as well, which contradicts with the least privilege AC, but ACR has no delete only policy/role.

### Revisit Triggers

We would revisit this decision if:
- A required service cannot be placed in a VNet
- ABAC repository roles prove unavailable or unstable in a required region;
- A regulation or customer contract requires CMK

## How We Will Check It

- Azure Policy reports any registry with public access enabled, key/admin authentication enabled, or `roleAssignmentMode` not set to `AbacRepositoryPermissions`.
- Changes to the ACR definition, role assignments, or admission policy go through pull-request review and show up in Activity Logs.
- successful tests: access with the admin account fails; a pull from outside the approved VNet fails; a pull identity attempting a push gets an authorization error; an unsigned image is rejected at admission; a tag-mutation or delete attempt outside the approved deleter identity is denied.
- Recovery test: the scheduled restore from the air-gapped vault succeeds and the restored digest matches the production digest.
- Alert test: a simulated unusual push/access... attempt generate alerts to the named owner.
- Contributor test: a permanent Contributor assignment attempt is denied by policy; a PIM activation without justification or MFA fails; an activation outside a declared incident fires an alert to the named owner.
