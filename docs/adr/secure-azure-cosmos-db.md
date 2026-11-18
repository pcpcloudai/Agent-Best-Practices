# ADR-0002: Secure Azure Cosmos DB for agent conversation storage

## Context

The Backend API (control plane) owns the data and the agents run in Session Pools (execution plane). Sessions are ephemeral, so conversation history can't live in them. It lives in Azure Cosmos DB.

That history can contain personal or confidential information. The agent sessions, on the other hand, run prompts we don't control and call tools on the user's behalf. If a session is hijacked, it should not be able to read other users' conversations. That is the main risk this ADR is about.

So the question is: how do we secure Cosmos DB so that only the control plane can reach the data, without having secrets to store and rotate?

## Decision drivers

* A compromised agent session must not be able to reach stored conversations.
* No long-lived credentials.
* As little network exposure as possible.
* We need to be able to audit who accessed what.

## Options considered

### Option 1: Public endpoint, IP firewall, account keys

This is the default setup and the easiest one. We reject it. The account keys give full access to everything, they can't be limited per identity, and they have to be stored and rotated somewhere. IP allow lists are also fragile when outbound addresses change. It goes against the "no long-lived secrets" goal.

### Option 2: Private Endpoint, Microsoft Entra ID RBAC, key authentication disabled

Cosmos DB is only reachable through a private IP inside our virtual network, and only identities with an explicit role can read or write data. There are no keys to leak. The cost is more setup: private DNS, role assignments in IaC, and a more annoying local development experience.

### Option 3: Option 2 plus customer-managed keys and client-side encryption

This gives more control over encryption keys and keeps some fields unreadable even for administrators. But it adds key lifecycle work, a risk of losing data if a key is lost or revoked, and it limits queries on encrypted fields.

## Decision

**Option 2**.

* Public network access is disabled. Cosmos DB is reached through a Private Endpoint and the matching private DNS zone.
* Key-based authentication is disabled. The Backend API connects with its Managed Identity and Microsoft Entra ID.
* The Backend API identity gets one built-in Cosmos DB data role (Reader or Contributor, whichever it really needs), scoped as narrowly as possible.
* Session Pool identities get no role on Cosmos DB, and they have no network route to it. When an agent needs conversation history, it asks the Backend API, which checks that the conversation belongs to the user.
* TLS 1.2 or higher is required. Encryption at rest stays on the default service-managed keys.
* Diagnostic logs go to Log Analytics, and Microsoft Defender for Azure Cosmos DB is turned on. We alert on authorization failures.
* Everything is defined in IaC, and Azure Policy flags accounts that have public access or key authentication enabled.

Option 3 stays available as an extension. If a regulation or a data classification asks for customer-managed keys or field-level encryption, we can add it on top of Option 2 without changing the rest.

## Consequences

The main gain is that a compromised session has nothing to steal: no key, no role, no route. Access is tied to identities and can be revoked, and the logs tell us who did what.

In return, we accept some friction. Private Endpoints cost money and need DNS to be set up correctly. Developers can't just paste a connection string anymore, so we need a documented way to work locally (for example, personal identities with a dev role, or a separate dev account with looser network rules). Cosmos DB data roles are also managed separately from the usual Azure RBAC, which means one more thing to keep in the IaC.

The Backend API also becomes the single door to conversation data, so it has to be monitored and sized with that in mind.

## How we will check it

* Azure Policy reports any account with public access or key authentication enabled.
* Changes to the Cosmos DB definition go through a pull request review.
* A few negative tests: access with a key fails, access from outside the network fails, and a session identity gets an authorization error.


