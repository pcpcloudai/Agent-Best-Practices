# ADR-0002: Secure Azure Cosmos DB for agent conversation storage

## Context

Azure Cosmos DB is a fully managed, globally distributed NoSQL database that provides flexible JSON document storage, high availability, and low-latency access. For AI agent development, it can store structured data such as conversations, agent states, and memories, alongside document chunks and their vector embeddings. Its native vector search capabilities enable semantic retrieval, making it suitable for implementing RAG pipelines and persistent agent memory.

Given the importance of this data, an improperly secured Cosmos DB account can expose it to unauthorized access, data theft, modification, or deletion. Attackers could also exploit excessive permissions or compromised credentials to perform unauthorized operations, disrupt service availability, or manipulate stored documents and memories, potentially affecting the reliability of AI agents.

The architecture already establishes that external users access Cosmos DB exclusively through the backend API. However, this alone does not eliminate all security risks. This ADR evaluates additional measures to strengthen the database's security, reduce its exposure to potential attacks, and protect the confidentiality, integrity, and availability of its data.

## Decision Drivers

The selected security approach should satisfy the following requirements:

1. **Zero Trust:** Do not implicitly trust authenticated user sessions or application requests. A compromised external user session must not provide a direct path to Cosmos DB or grant access to database resources. All database operations must be mediated by the backend API and subject to authorization checks.

2. **Least-Privilege Access:** Grant each identity only the minimum permissions required to perform its responsibilities, limiting the potential impact of compromised credentials or components.

3. **Network Isolation:** Minimize database exposure by restricting access to trusted network paths and preventing unauthorized external connectivity.

4. **Credential Security:** Avoid long-lived credentials and static secrets wherever possible. Prefer short-lived, identity-based authentication mechanisms and secure credential management.

5. **Data Protection:** Protect stored data against unauthorized disclosure, modification, and deletion, both at rest and in transit.

6. **Resilience and Recoverability:** Ensure that data can be recovered following accidental deletion, malicious modifications, or destructive attacks.

7. **Auditability and Monitoring:** Maintain sufficient visibility into database access and administrative activities to detect suspicious behavior and support security investigations.

8. **Operational Complexity:** Favor solutions that can be integrated into the existing architecture, automated, and maintained without introducing unnecessary infrastructure or administrative overhead.

9. **Cost Efficiency:** Consider the additional infrastructure, licensing, storage, and operational costs introduced by each security measure while maintaining an appropriate level of protection.

## Options Considered

### 1. Network Security

#### Option 1: Public Endpoint with IP Firewall

Keep the public endpoint enabled and configure IP firewall rules to allow connections only from explicitly authorized IP addresses.

**Status: Rejected**

Although IP filtering restricts access to known addresses, the database remains exposed through a public endpoint. It also requires maintaining an accurate allowlist, which can become difficult when application infrastructure uses changing outbound IP addresses.

**Decision drivers violated:**

- **Network Isolation:** Does not eliminate public network exposure.
- **Operational Complexity:** Requires ongoing IP allowlist management.

#### Option 2: Private Endpoint with Public Network Access Disabled

Configure an Azure Private Endpoint to expose Cosmos DB through a private IP address within the application's Virtual Network and disable public network access.

**Status: Accepted**

This approach ensures that Cosmos DB is accessible only through approved private network connections, preventing direct access from the public internet. It reduces the database's network attack surface and restricts connectivity to trusted application infrastructure.

#### Option 3: Private Endpoint with Additional Network Segmentation

Build on Option 2 by introducing additional network segmentation through dedicated subnets, Network Security Groups (NSGs), and restrictive routing rules.

**Status: Deferred**

Although additional network segmentation provides defense in depth, it introduces more infrastructure configuration and maintenance. The Private Endpoint provides the required network isolation for the current architecture. Additional segmentation can be considered if the infrastructure grows or isolation requirements become more demanding.

---

### 2. Authentication

#### Option 1: Cosmos DB Account Keys

Authenticate using the primary or secondary Cosmos DB account keys.

**Status: Rejected**

Account keys provide broad access to the database account and are not scoped to individual application identities or specific data operations. If compromised, they can allow an attacker to perform unauthorized database operations.

**Decision drivers violated:**

- **Least-Privilege Access:** Account keys provide broader permissions than required for normal application operations.
- **Credential Security:** Long-lived credentials must be securely distributed, stored, and rotated.

#### Option 2: Microsoft Entra ID with Service Principal

Create a dedicated application identity in Microsoft Entra ID and authenticate using its client credentials.

**Status: Rejected**

Although this provides identity-based authentication, it requires managing client secrets or certificates, including their storage, distribution, expiration, and rotation. Since the backend runs on Azure infrastructure that supports Managed Identity, this approach introduces unnecessary credential-management overhead.

**Decision drivers violated:**

- **Credential Security:** Introduces credentials that can be exposed or compromised.
- **Operational Complexity:** Requires additional credential lifecycle and rotation management.

#### Option 3: Microsoft Entra ID with Managed Identity

Assign a system-assigned or user-assigned Managed Identity to the backend infrastructure and use it to authenticate to Cosmos DB.

**Status: Accepted**

Managed Identity eliminates the need to store or distribute database credentials. Azure manages the identity credentials and token lifecycle, allowing the backend to authenticate securely using Microsoft Entra ID.

---

### 3. Authorization

#### Option 1: Account-Level Keys

Use Cosmos DB account keys to authorize database operations.

**Status: Rejected**

Account keys do not provide the granular permission management required to enforce least-privilege access.

**Decision drivers violated:**

- **Least-Privilege Access:** Does not provide sufficiently granular authorization.
- **Credential Security:** Relies on long-lived credentials with broad access.

#### Option 2: Cosmos DB Native Data-Plane RBAC

Use Cosmos DB native role-based access control to assign a dedicated data-plane role to the backend's Managed Identity.

**Status: Accepted**

This allows permissions to be assigned at the account, database, or container scope. The backend receives only the permissions required for its operations, such as reading, creating, updating, and deleting documents.

This separates application data access from administrative operations and limits the impact of a compromised application identity.

#### Option 3: Fine-Grained Authorization with Separate Application Identities

Introduce multiple application identities with different permissions for specific application components or database operations.

**Status: Deferred**

Although this provides additional isolation between application components, it introduces unnecessary identity and permission management complexity for the current architecture, where a single backend is responsible for database access.

---

### 4. Data Encryption

#### Option 1: Microsoft-Managed Encryption Keys

Rely on Azure Cosmos DB's default encryption at rest using Microsoft-managed keys, together with TLS encryption for data in transit.

**Status: Accepted**

Cosmos DB automatically encrypts data at rest and supports encryption in transit. This provides baseline data protection without introducing additional key-management infrastructure.

#### Option 2: Customer-Managed Keys (CMK)

Use customer-managed encryption keys stored in Azure Key Vault to provide additional control over encryption key lifecycle and rotation.

**Status: Deferred**

Although CMK provides greater control over encryption keys, it introduces additional dependencies on Azure Key Vault and requires key lifecycle management. This additional complexity is not currently justified unless specific compliance or organizational requirements demand customer-controlled encryption keys.

#### Option 3: Customer-Managed Keys with Client-Side Encryption

Combine customer-managed keys with client-side encryption to encrypt selected sensitive fields before they are sent to Cosmos DB.

**Status: Deferred**

While this provides stronger protection for selected data fields, it introduces additional encryption and decryption responsibilities within the application. It can also restrict querying and indexing of encrypted properties, complicating document retrieval and AI agent workflows. This approach can be reconsidered if specific data confidentiality requirements justify the additional complexity.

---

### 5. Backup and Recovery

#### Option 1: Periodic Backup

Use Cosmos DB's periodic backup mode with a configured backup interval and retention period.

**Status: Rejected**

Periodic backups provide protection against data loss but offer less granular recovery capabilities than continuous backup. Recovery also requires restoring from an available backup rather than selecting an arbitrary point in time within the continuous backup retention window.

**Decision drivers violated:**

- **Resilience and Recoverability:** Provides less granular point-in-time recovery capabilities.

#### Option 2: Continuous Backup with Point-in-Time Recovery

Enable continuous backup to restore the database to a specific point in time within the configured retention period.

**Status: Accepted**

This provides granular recovery capabilities, allowing recovery from accidental deletion, malicious modifications, and other destructive events. It reduces potential data loss and provides greater flexibility when recovering from security incidents.

#### Option 3: Continuous Backup with Extended Retention and Tested Recovery Procedures

Combine continuous backup with an extended retention tier, documented recovery procedures, and regular restoration tests.

**Status: Deferred**

Extended retention and regular recovery testing provide additional resilience but introduce additional storage costs and operational responsibilities. The initial implementation will use continuous backup with an appropriate retention period. Extended retention and formal recovery testing can be introduced based on recovery requirements.

---

### 6. Monitoring and Auditing

#### Option 1: Basic Azure Monitor Metrics

Use Azure Monitor metrics to monitor database availability, throughput consumption, latency, and resource utilization.

**Status: Rejected**

Metrics provide operational visibility but are insufficient on their own to investigate individual database operations or detect suspicious access patterns.

**Decision drivers violated:**

- **Auditability and Monitoring:** Does not provide sufficient visibility into database access and administrative activities.

#### Option 2: Diagnostic Logging and Auditing

Enable Cosmos DB diagnostic settings and forward resource logs to Azure Monitor Log Analytics for centralized analysis and investigation.

**Status: Accepted**

This provides visibility into database operations and administrative activities, supporting security investigations, auditing, and the detection of unexpected access patterns.

#### Option 3: Advanced Threat Detection

Combine diagnostic logging, Azure Monitor alerts, and Microsoft Defender for Cosmos DB to detect suspicious activities and potential threats.

**Status: Deferred**

Advanced threat detection provides additional security visibility but introduces additional service costs and configuration requirements. It can be considered as a further security enhancement based on monitoring requirements and the organization's security budget.

---

### 7. Application-Level Security

#### Option 1: Basic Authentication and Authorization

Authenticate external users and validate their permissions before processing requests through the backend API.

**Status: Rejected**

Basic authentication and authorization alone do not provide sufficient protection against cross-user data access or compromised sessions if resource-level authorization is not consistently enforced.

**Decision drivers violated:**

- **Zero Trust:** An authenticated session must not automatically be trusted to access database resources.
- **Least-Privilege Access:** Does not sufficiently enforce resource-level access restrictions.

#### Option 2: Tenant Isolation and Resource-Level Authorization

Enforce ownership and tenant checks on every database operation, ensuring users can access only their own conversations, documents, and memories.

**Status: Rejected**

Although tenant isolation prevents unauthorized cross-user access, it does not independently address other application-level threats, such as injection attacks, excessive request rates, or insufficient auditability.

**Decision drivers violated:**

- **Zero Trust:** Does not provide sufficient defense against compromised sessions and malicious requests.
- **Auditability and Monitoring:** Does not independently provide sufficient visibility into application-level security events.

#### Option 3: Defense-in-Depth

Combine tenant isolation and resource-level authorization with strict input validation, parameterized queries, API rate limiting, and security audit trails.

**Status: Accepted**

This approach introduces multiple layers of application-level protection, reducing the likelihood that a single vulnerability or compromised session can lead to unauthorized database operations. It also helps mitigate injection attacks, resource abuse, and unauthorized access to other users' data.

This is the selected approach because it provides broader protection while maintaining the backend API as the only access path to Cosmos DB.

## Decision Summary

The following table summarizes the selected decisions for securing Azure Cosmos DB.

| Security Dimension             | Selected Decision                                    |
| ------------------------------ | ---------------------------------------------------- |
| **Network Security**           | Private Endpoint with Public Network Access Disabled |
| **Authentication**             | Microsoft Entra ID with Managed Identity             |
| **Authorization**              | Cosmos DB Native Data-Plane RBAC                     |
| **Data Encryption**            | Microsoft-Managed Encryption Keys                    |
| **Backup and Recovery**        | Continuous Backup with Point-in-Time Recovery        |
| **Monitoring and Auditing**    | Diagnostic Logging with Azure Monitor Log Analytics  |
| **Application-Level Security** | Defense-in-Depth                                     |

These decisions establish the selected security baseline for Azure Cosmos DB, addressing the security requirements identified in the decision drivers.

## Consequences

The selected security measures strengthen the confidentiality, integrity, and availability of agent conversation data. Private networking reduces the database's exposure to external threats, while Managed Identity and native RBAC eliminate static credentials and enforce least-privilege access. Encryption protects data at rest and in transit, continuous backup improves recoverability, and centralized diagnostic logging provides visibility into database operations and administrative activities. At the application level, defense-in-depth adds protection against unauthorized access, injection attacks, and resource abuse.

These measures also introduce some operational and architectural trade-offs. Private networking requires appropriate infrastructure configuration, RBAC requires careful permission management, and diagnostic logging introduces additional monitoring and log-management responsibilities. Furthermore, relying on Microsoft-managed encryption keys provides less control over key management, while continuous backup recovery remains subject to the configured retention period. Overall, the selected approach provides an appropriate balance between security, operational complexity, and cost for the current architecture.

## Confirmation

The implementation of the selected security measures will be verified through configuration reviews and targeted security tests:

- **Network Security:** Verify that public network access is disabled and that Cosmos DB is accessible through the configured Private Endpoint.
- **Authentication and Authorization:** Verify that the backend uses Managed Identity and that its assigned RBAC permissions follow the principle of least privilege.
- **Data Encryption:** Confirm that encryption at rest is enabled and that connections use TLS.
- **Backup and Recovery:** Verify that continuous backup is enabled and perform a test restoration to validate recovery capabilities.
- **Monitoring and Auditing:** Confirm that diagnostic logs are correctly forwarded to Log Analytics and that relevant database activities are captured.
- **Application-Level Security:** Use automated tests to verify tenant isolation, resource-level authorization, input validation, and protection against unauthorized operations.

Security configurations and validation tests should be reviewed before production deployment and periodically thereafter to ensure continued compliance with the selected security measures.
