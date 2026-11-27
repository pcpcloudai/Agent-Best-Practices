---
status: Proposed
date: 2026-11-27
deciders: owner

---

# Architecture Decision Record about Azure storage account

## Context and Problem Statement

Our AI agent platform runs agents in Azure Container Apps Session Pools. Sessions are throwaway: one can be ended, replaced or scaled at any time. That means anything we need to keep, such as uploaded files, long-term memory, audit records and workflow checkpoints, cannot live inside a session. Our architecture guidance says this kind of object data belongs in an Azure Storage account.

The settings we pick for that storage account (the account type, how the data is copied, who can reach it and how they sign in) matter for security, uptime and cost. Some of them are also awkward to change once the account is in use.

So the question is: what kind of storage account should we use, and who should be allowed to talk to it?

## Decision Drivers

- Keep the control plane and the execution plane apart. The backend owns the data, and the agent sessions only do the work.
- Treat sessions as untrusted. Agents handle untrusted prompts and run tools, so they should not hold keys or broad access to our data.
- Keep the data private. Nothing should be open to the internet, and we want no shared keys or passwords.
- Keep it available. If one datacenter in the region has a problem, we still need our files.
- Make mistakes recoverable. If a file is deleted or overwritten by accident, we should be able to get it back.
- Keep the cost sensible, and keep dev, test and production set up the same way.

## Considered Options

1. One Standard general-purpose v2 storage account with zone-redundant storage (ZRS) and private endpoints. Only the backend API can use it, signing in with its managed identity. Sessions get no access of their own.
2. The same storage account, but each agent session gets its own access (its own identity with storage permissions) and reads and writes files directly.
3. A storage account with a public endpoint, where sessions use account keys or connection strings.

## Decision Outcome

We chose option 1: a Standard general-purpose v2 account with ZRS and private endpoints, used only by the backend API through its managed identity. It fits our rule that the backend keeps ownership of data and sessions stay disposable and low-trust, and it needs no keys.

When a session needs a file, the backend either passes the content to the session or gives it a short-lived, read-only link for that one file. We create the link with Entra ID credentials, not with the account key.

Settings every storage account should start with:

- Account type: Standard general-purpose v2
- Copies of data: ZRS, or GZRS if we later need a second region for disaster recovery
- Sign-in: Microsoft Entra ID with role-based access for the backend's managed identity, and account key access switched off
- Network: public access turned off, with access only through private endpoints
- Security basics: HTTPS only, TLS 1.2 or newer, and no anonymous access to blobs
- Layout: files stored under the owning user and conversation, so ownership is clear and easy to audit
- Recovery: soft delete and blob versioning turned on
- Cost: lifecycle rules that move old data to cheaper tiers and delete what we no longer need
- Monitoring: diagnostic logs sent to Log Analytics

### Consequences

What we gain:

- Sessions never hold storage keys, so a compromised or misbehaving agent cannot reach our files directly.
- Data stays safe when a session is thrown away, and stays available if a single zone goes down.
- Ownership and access checks happen in one place, the backend, which makes auditing simpler.
- Azure Policy can make sure every account follows the same rules.

What it costs us:

- The backend becomes the path for all file access, so it needs to handle the load and stay reliable.
- Private endpoints cost money and need some network and DNS setup.
- ZRS is pricier than LRS, and it won't help if a whole region goes down.

### Confirmation

We will check that this decision is actually followed by:

- Using Azure Policy to flag or block account keys, public access and redundancy settings we haven't approved
- Checking that no session identity has a storage role assigned
- Reviewing the deployment code against this ADR
- Doing a design review before the first production rollout

## Pros and Cons of the Options

### Option 1: Backend-only access, ZRS, private endpoints, Entra ID

- Good: it keeps ownership and governance in the control plane, as our architecture guidance asks.
- Good: it is the smallest blast radius if a session is compromised.
- Bad: all file traffic goes through the backend, which adds some load and effort.

### Option 2: Each session accesses storage directly

- Good: it is simple for the agent, with no backend hop for files.
- Bad: every session needs storage permissions, which goes against least privilege.
- Bad: it is harder to prove who accessed what, and a bad session can reach more data.

### Option 3: Public endpoint with keys or connection strings

- Good: it is the quickest and cheapest to get running.
- Bad: account keys are long-lived secrets, and sessions would carry them.
- Bad: a public endpoint gives attackers more to aim at.

## More Information

- Architecture guidance: Architecting AI Agents with Azure Container Apps Session Pools (in this repository)
- ADR templates: https://adr.github.io/
- ADR process: https://docs.aws.amazon.com/prescriptive-guidance/latest/architectural-decision-records/adr-process.html
- MADR template: https://medium.com/olzzio/the-markdown-adr-madr-template-explained-and-distilled-b67603ec95bb
- Azure Storage documentation: https://learn.microsoft.com/en-us/azure/storage/