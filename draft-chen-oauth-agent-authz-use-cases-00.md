---
stand_alone: true
ipr: trust200902
cat: std
submissiontype: IETF
area: Security
wg: OAuth Working Group

docname: draft-chen-oauth-agent-authz-use-cases-00

title: Agent Authorization use cases and gap analysis
abbrev: Authorization use cases
lang: en
kw:
  - OAuth
  - Agent
  - AI
  - Artificial Intelligence
  - Authorization
  - Delegation
  - Revocation
  - Gap Analysis

author:
- name: Meiling Chen
  org: China Mobile
  city: BeiJing
  country: China
  email: chenmeiling@chinamobile.com
- name: Jia Chen
  org: China Mobile
  city: BeiJing
  country: China
  email: chenjia@chinamobile.com
- name: Jiankang Yao
  org: CNNIC
  city: BeiJing
  country: China
  email: yaojk@cnnic.cn
  
informative:
  RFC2119:
  RFC8174:
  RFC7009:
  RFC6749:
  RFC8693:
  RFC9396:
  I-D.song-oauth-ai-agent-collaborate-authz:


--- abstract

This document provides a systematic analysis of these emerging agent-based use cases. It categorizes them into distinct scenarios, details their specific authorization requirements, and performs a comprehensive gap analysis against the existing OAuth 2.0 framework [RFC6749] and its common extensions. The analysis identifies fundamental mismatches, the goal of this document is to articulate these gaps clearly, providing a foundation for future work on new extensions within the OAuth Working Group to address the authorization needs of the next generation of ai agents.

--- middle

# Introduction

The OAuth 2.0 Authorization Framework [RFC6749] has become the de facto standard for delegated authorization on the internet. Its success is rooted in a well-defined model where a resource owner grants a third-party client limited access to their resources without sharing their credentials. This model primarily assumes a user-present, interactive flow where a static set of permissions (scopes) is approved upfront.

However, the landscape is rapidly evolving with the advent of sophisticated AI-driven "Agents". These are not simple clients but autonomous or semi-autonomous entities that perform complex, multi-step tasks on behalf of a user or another system. Their operational characteristics include:

*   **Delegated Autonomy:** Agents act with authority delegated from a principal (user or system) over extended periods.
*   **Complex Task Decomposition:** An agent might take a high-level instruction (e.g., "plan my business trip") and decompose it into numerous discrete actions involving multiple resource servers.
*   **Asynchronous & Long-Running Operations:** Tasks may run for hours, days, or indefinitely, often without direct, real-time supervision from the principal.
*   **Dynamic & Emergent Needs:** The exact permissions required may not be known at the start of a task but emerge as the agent plans and executes its steps.
*   **Composition & Chaining:** Agents may delegate sub-tasks to other agents, forming a chain of authority.

This document does not propose new solutions or protocols. Instead, its purpose is to:
*   Define the key actors and concepts in agent-based authorization scenarios.
*   Describe a set of core use cases that exemplify the new challenges.
*   Conduct a gap analysis for each use case, identifying where the current OAuth 2.0 framework and its extensions fall short.
*   Summarize the key security considerations.

By clearly articulating these gaps, this document aims to provide a shared understanding of the problem space and stimulate focused work on developing interoperable solutions for the next generation of delegated authorization.

# Terminology

In addition to the terms defined in [RFC6749], this document uses the following terms:

**User (Resource Owner):**
: An entity capable of granting access to a protected resource. In the context of this document, the user is the person who delegates authority to an agent. Conforms to the definition of "Resource Owner" in [RFC6749].

**Authorization Server (AS):**
: The server that authenticates the User and issues access tokens to the Agent after obtaining the User's authorization. Conforms to the definition in [RFC6749].

**Resource Server (RS):**
: The server hosting the protected resources, capable of accepting and responding to protected resource requests using access tokens. Conforms to the definition in [RFC6749].

**Delegation Chain:**
: A sequence of delegation events, where one actor grants a subset of its authority to another. For example: User -> Agent A -> Agent B.

**Intent:**
: A high-level description of a goal or task that the user wants the agent to accomplish (e.g., "book me a flight to Hawaii for next week").

**Tool:**
: A specific API or function that an agent can call to perform an action (e.g., a `search_flights` API, a `send_email` function).

#  Core Use Cases and Gap Analysis

This section explores different categories of use cases, providing a concrete example for each, and analyzes the gaps in the existing OAuth 2.x framework. For each use case, we examine what can be achieved with existing tools and, more importantly, what is missing.

## Category 1: Personal & Consumer Scenarios

### Use Case 1: Personal Digital Assistant

*   **Scenario Description:** A user gives a high-level, natural language command to their AI assistant. The assistant must decompose this command into multiple steps and interact with various services to fulfill the request.

*   **Example:** Alice tells her AI assistant, *"Help me plan a picnic for this Saturday."* The assistant needs to:
    1.  Check Alice's calendar for availability (requires `calendar.read`).
    2.  Check the weather forecast for Saturday (requires `weather.read`).
    3.  If the weather is good, suggest nearby parks (requires `maps.search`).
    4.  After Alice chooses a park, book a picnic spot if required (requires `parks.book`).
    5.  Finally, add the event to Alice's calendar (requires `calendar.write`).

*   **Authorization Requirements:**
    *   **Intent-to-Permission Translation:** The system must translate the high-level intent ("plan a picnic") into a series of specific, just-in-time permission requests.
    *   **Interactive, Multi-Step Consent:** The agent needs to "talk back" to the user for intermediate decisions (e.g., "The weather looks great! I found three parks nearby. Which one do you prefer?"). Authorization for the next step (booking) should only be granted after the user's choice.
    *   **Task-Level Revocation:** Alice should be able to say, *"Cancel the picnic planning,"* and have all permissions related to this specific task instantly revoked, without affecting other tasks the agent might be performing.

*   **Gap Analysis:**
    *   **What Works (Partially):** The OAuth Authorization Code flow can be used to get the initial permissions (like `calendar.read`). Refresh Tokens can keep the agent's session alive.
    *   **What's Missing (The Gap):**
        *   **Fundamental Paradigm Mismatch:** OAuth is a **"pre-authorization"** model. It asks the user to approve a static list of scopes (`calendar.read`, `weather.read`, `maps.search`, etc.) all at once at the beginning. It cannot understand the *intent* ("plan a picnic") and dynamically grant permissions as the task unfolds. This is the core gap.
        *   **No Standardized Interactive Flow:** There is no standard OAuth mechanism for an agent to "pause" a task, ask the user for a decision, and then use that decision to request a new, specific permission. This logic is currently left to complex, proprietary application-layer implementations.
        *   **Impractical Revocation:** OAuth Token Revocation ([RFC7009]) revokes a single token. To achieve task-level revocation, the application would need to manage a complex mapping of tasks to tokens. There is no standard way to say "revoke everything related to 'picnic planning'."

### Use Case 2: Smart Home & Automation

*   **Scenario Description:** A central home hub agent manages various IoT devices based on pre-defined rules or real-time events.

*   **Example:** Bob sets up a "Good Morning" routine. When his alarm goes off at 7 AM on a weekday, the home hub agent is authorized to:
    1.  Slowly turn on the bedroom lights (device: `bedroom_light`, capability: `brightness`).
    2.  Set the thermostat to 70°F (device: `thermostat`, capability: `set_temperature`).
    3.  Start the coffee maker (device: `coffee_maker`, capability: `on_off`).

*   **Authorization Requirements:**
    *   **Persistent Delegation:** After a one-time setup, the agent must be able to perform these actions daily without Bob's intervention.
    *   **Fine-Grained Device & Capability Permissions:** The agent should be authorized to control the `brightness` of the `bedroom_light`, but not, for example, to unlock the `front_door`.
    *   **Conditional or Event-Driven Authorization:** The permissions should only be usable when specific conditions are met (e.g., `time=7AM` AND `day=weekday`).
    *   **Bulk Revocation:** When Bob sells his house, he needs a simple way to revoke all permissions for all devices from his home hub with a single action.

*   **Gap Analysis:**
    *   **What Works (Partially):** Refresh Tokens are well-suited for persistent delegation. Scopes can be defined with high granularity (e.g., `device:bedroom_light:brightness`).

    *   **What's Missing (The Gap):**
        *   **"Scope Explosion" and Usability:** In a home with hundreds of devices and capabilities, presenting the user with a list of thousands of scopes to approve is unmanageable. The OAuth consent screen was not designed for this scale.
        *   **No Standardized Policy Enforcement:** OAuth grants *what* an agent can do, but not *when* or *under what conditions*. The logic to enforce `time=7AM` is outside the protocol and must be custom-built into the agent or the device's resource server, leading to inconsistent and non-interoperable implementations.
        *   **No Standardized Bulk Revocation:** [RFC7009] is for revoking one token at a time. There is no standard API to "revoke all tokens for user Bob" or "revoke all tokens issued to the home hub client." This is a critical administrative and security gap, forcing reliance on proprietary AS-specific APIs.

### Use Case 3: Agent as User's Full Proxy to Access Third-Party Tools

*   **Scenario Description:** A user authorizes an agent to use a third-party service on their behalf. The service, however, was designed for human interaction and has no built-in awareness of automated agents, leading to potential misuse or policy violations.

*   **Example:**  Charlie authorizes his personal research assistant agent to use his subscription to an academic paper database. The agent, acting as Charlie, begins downloading hundreds of papers for a literature review. The database service detects this high-frequency activity, flags it as a potential DDoS attack or data scraping, and temporarily locks Charlie's account, disrupting both the agent's task and Charlie's own access.
    

*   **Authorization Requirements:**
    *   **Agent-User Differentiation:** : The third-party service must be able to reliably distinguish between requests coming directly from Charlie and requests made by his agent.
    *   **Agent-Specific Policies:** The service needs a way to apply different policies (e.g., stricter rate limits, restricted API access) to the agent without impacting the human user's normal access rights.
    *   **Delegated Authority with Constraints:** The authorization given to the agent should be a constrained subset of the user's full permissions (e.g., "can search and download, but no more than 100 papers per hour").

*   **Gap Analysis:**
    *   **What (partially) works:**  OAuth allows a user to delegate access to a client (the agent). The scope parameter can limit which APIs the agent can call.
    
        *   **What Works (Partially):** OAuth allows a user to delegate access to a client (the agent). The `scope` parameter can limit which APIs the agent can call.
    *   **What's Missing (The Gap):**
        *   **No Standard Agent Identifier:** There is no standard OAuth claim or parameter that explicitly signals "this request is from an automated agent." Resource servers are left to guess based on non-standard signals like User-Agent strings or unusual traffic patterns.
        *   **Inability to Express Constraints:** The standard `scope` mechanism is binary (permission is granted or not). It cannot express or enforce nuanced constraints like rate limits, data volume caps, or time-of-day restrictions as part of the authorization grant itself.
        *   **Confused Deputy Risk:** Without clear differentiation, the service provider cannot tell if high-volume activity is malicious (account takeover) or a legitimate but overly aggressive agent. Their only recourse is often to block the user's account entirely.
 

### Use Case 4: Agent as User's Proxy to Access Operating System Resources

*   **Scenario Description:** An agent running on a user's device (e.g., a laptop or smartphone) needs to perform tasks that require access to local resources like files, applications, or system settings.
 
*   **Example:** Diana asks her desktop agent, *"Clean up my downloads folder and free up disk space."* To do this, the agent needs permission to read and delete files within `/home/diana/downloads`. However, to be "helpful," the agent might also try to clear system-wide caches, which requires elevated, system-level permissions.
 
*   **Authorization Requirements:**
    *   **Fine-Grained Resource Access:** The agent should be granted access only to the specific files or settings needed for a task (e.g., a single folder), not the user's entire home directory or full system access.
    *   **Task-Scoped Permissions:** Permissions should be granted for the duration of a specific task ("clean downloads folder") and automatically revoked upon completion.
    *   **Secure Privilege Escalation:** If a task requires elevated permissions, there must be a secure, user-approved mechanism for the agent to request them just-in-time, rather than running with high privileges constantly.
 
*   **Gap Analysis:**
    *   **What Works (Partially):** Modern operating systems have their own permission models (e.g., app sandboxing, runtime permission prompts).
    *   **What's Missing (The Gap):**
        *   **Architectural Mismatch:** OAuth is a framework for network-based delegated authorization. It is not designed to manage local, process-level OS permissions. There is a fundamental mismatch between the two models.
        *   **Lack of a Standard Bridge:** There is no standard protocol to connect an agent's high-level intent (often determined in the cloud) to a secure, fine-grained grant of local OS permissions on the device.
        *   **Over-Privileging by Default:** Due to the lack of a standard bridge, developers often resort to a dangerous workaround: the agent's local component is installed with broad, persistent permissions (e.g., running with the user's full rights or even as a system service). This completely bypasses the principle of least privilege and creates a significant security risk.
 

## Category 2: Enterprise & Business Process Scenarios

### Use Case 5: Complex Business Process Automation

*   **Scenario Description:** A task is passed through a chain of specialized agents, each performing one step of a larger business process.

*   **Example:** An automated insurance claim process:
    1.  **Agent A (Intake):** Receives a claim from a customer and is authorized to read the customer's policy. It passes the claim to the next agent.
    2.  **Agent B (Verification):** Receives the claim from Agent A. It is authorized to access external databases to verify the details of the incident. It then passes the verified claim to Agent C.
    3.  **Agent C (Adjudication):** Receives the verified claim from Agent B. It is authorized to run a risk assessment and approve or deny the claim. If approved, it delegates the final action to Agent D.
    4.  **Agent D (Payment):** Receives the approved claim from Agent C. It is authorized *only* to execute a payment to the customer's bank account for the approved amount.

*   **Authorization Requirements:**
    *   **Verifiable Delegation Chain:** The final agent (Payment Agent D) and the resource server (the bank's API) must be able to cryptographically verify the entire authorization path: `Customer -> Agent A -> Agent B -> Agent C -> Agent D`.
    *   **Principle of Least Privilege at Each Step:** Agent B should have no payment authority, and Agent D should have no access to the customer's policy details. The permissions must be strictly constrained at each step in the chain.
    *   **Auditable Context:** The entire process must be tied to a single, auditable `claim_id` that is securely passed along the chain.

*   **Gap Analysis:**
    *   **What Works (Partially):** OAuth 2.0 Token Exchange [RFC8693] introduces the `act` (actor) claim, which provides a primitive to show that one agent is acting on behalf of another. This is a foundational building block.
    *   **What's Missing (The Gap):**
        *   **No Native Support for Chains:** A standard OAuth token represents a simple, two-party delegation (User -> Agent A). It cannot natively represent a multi-step chain (`User -> A -> B -> C`). While the `act` claim from [RFC8693] helps, there is no standard for how to nest these claims to create a verifiable, multi-hop chain. This is a major architectural gap.
        *   **Lack of Standardized Context Passing:** There is no standard field in an OAuth token to carry the `claim_id` securely through the process. Developers resort to custom claims in a JWT, which harms interoperability.

### Use Case 6: Coordinated Task Group

*   **Scenario Description:** A coordinating agent decomposes a user's request into subtasks and assembles a group of specialized sub-agents to execute them. Unlike the delegation chain in Use Case 5, where each agent derives its authority from the previous agent hop by hop, here every member's authority stems from a single grant obtained centrally by the coordinator [I-D.song-oauth-ai-agent-collaborate-authz]. The execution order of subtasks (sequential, parallel, or mixed) is independent of this authorization structure.

*   **Example:** A user asks a health assistant for real-time health advice. A leading agent resolves the request into three subtasks: collecting the user's health data, predicting the user's health status, and generating advice. It selects a sub-agent for each subtask, each needing access to different resource servers (a wearable data API, a prediction service, a guideline repository). Sub-agents may all be selected upfront, or incrementally during execution as intermediate results reveal which further subtasks are needed.

*   **Authorization Requirements:**
    *   **One Grant, Many Members:** The coordinator must be able to obtain authorization once, on behalf of the whole group, rather than each member running its own flow against the authorization server and the user.
    *   **Member-Level Least Privilege:** Each member must be confined to its own subject-audience-scope binding within the group's overall authority.
    *   **Late Binding of Members:** Members selected mid-execution must be able to receive authority under the existing grant without a full re-authorization round trip.
    *   **Group Lifecycle:** When the task completes or the coordinator is compromised, the entire group's authority must be terminable as one unit.

*   **Gap Analysis:**
    *   **What Works (Partially):** The Client Credentials grant and Token Exchange [RFC8693] allow individual agents to obtain tokens, and Rich Authorization Requests [RFC9396] can express fine-grained permissions per request.
    *   **What's Missing (The Gap):**
        *   **Per-Member Authorization Does Not Scale:** N members individually authenticating and obtaining tokens for what is logically one user grant means N authorization server interactions and potentially N consent prompts for a single request. There is no standard way for one party to request authorization on behalf of an enumerated set of clients.
        *   **No Group Construct in the Token Model:** Tokens describe one client acting for one subject. There is no standard representation of group membership, nor of per-member subject-audience-scope bindings under a common grant, that a resource server could verify.
        *   **No Late Binding of Members:** Members that cannot be enumerated at grant time have no standard way to be admitted to the group's authority, for example via a verifiable task assignment bound to the original grant.
        *   **No Group Lifecycle Management:** There is no standard way to terminate a group's authority atomically, nor an enforced rule that a member's effective permissions remain a subset of the group's.

### Use Case 7: Automated DNSSEC DS Record Maintenance Agent

*   **Scenario Description:** This case follows the multi-agent RRR (Registrant-Registrar-Registry) model defined in `draft-ietf-dnsop-ds-automation-09`. A corporate domain owner deploys chained agents to fully automate DS record updates during DNSSEC KSK rollovers, zone bootstrapping, and zone deletion, relying on CDS/CDNSKEY signals between child authoritative zones and parent registry systems.

*   **Example:** The whole workflow runs unattended under a one-time delegated permission from the domain registrant, forming a delegation chain: **Registrant --> Registrar Agent --> Registry Agent.**
    - **Zone Agent:** Publishes consistent CDS/CDNSKEY records on all name servers after key rollover.
    - **Registrar Agent:** Pulls and validates CDS data, forwards DS update requests to the registry.
    - **Registry Agent:** Applies DS records to the parent zone and sends operation notifications.

*   **Authorization Requirements:**
    - Cryptographically verifiable multi-hop delegation chain for a full audit trail.
    - Fine-grained permission limited to a fixed set of domains, only allowing DS updating without modifying other DNS records.
    - Support task-specific revocation: revoke all DS automation permissions without affecting other agent workflows.
    - Standard audit context identifier passed across all agent hops for compliance logging.

*   **Gap Analysis:**
    *   **What Works (Partially):** Client Credentials can assign identities to registry/registrar agents. RFC 8693 token exchange supports simple single-hop agent delegation.

    *   **What's Missing (The Gap)
        *   **Current DS automation does not use OAuth. OAuth only supports two-party delegation and lacks standard multi-hop RRR delegation syntax; proprietary JWT claims hurt interoperability.
        *   **OAuth has no standardized task/bulk revocation. Securing DS automation via OAuth would require repetitive single-token revocation in incidents.
        *   **No reserved OAuth JWT claim for cross-agent audit IDs, breaking consistent compliance logging for DS automation.

## Category 3: Security & Administrative Scenarios

### Use Case 8: Automated Security Incident Response

*   **Scenario Description:** A security agent detects a security threat and must take immediate, automated action to contain it.

*   **Example:** A security system detects that an employee's laptop has been compromised by malware. An automated security agent is triggered to:
    1.  Immediately revoke all active login sessions for that employee across all company applications (e.g., email, code repository, HR system).
    2.  Isolate the compromised laptop from the corporate network.
    3.  Temporarily suspend the user's account to prevent further unauthorized access.

*   **Authorization Requirements:**
    *   **Privileged, System-Level Authority:** The security agent needs broad, pre-approved authority to perform high-impact administrative actions.
    *   **Global Token Revocation API:** The agent must be able to make a single API call to the Authorization Server to "immediately revoke all access and refresh tokens associated with user ID `employee-123`."

*   **Gap Analysis:**
    *   **What Works (Partially):** The OAuth Client Credentials grant is suitable for giving the security agent its own system-level identity and authority.
    *   **What's Missing (The Gap):**
        *   **Critically Inadequate Revocation API:** This is the most significant gap for this use case. The one-token-at-a-time revocation endpoint in [RFC7009] is completely insufficient for a security incident. The need to make potentially thousands of individual API calls to revoke tokens is too slow and unreliable during an active attack. The lack of a standardized bulk revocation API is a major operational and security failure point.

# Summary of Major Gaps

The use cases above highlight several fundamental gaps between the needs of AI agents and the capabilities of the standard OAuth 2.x framework:

1.  **From 'Pre-Approval' to 'Continuous Dialogue': A Paradigm Shift.** OAuth's model is to get all permissions upfront. Agents need a continuous, interactive authorization model where permissions are granted dynamically and just-in-time as a task evolves from a high-level intent.

2.  **Lack of a Standardized Interactive Channel.** The framework has no built-in mechanism for an agent to "pause" and securely ask the user for an intermediate decision or to respond to a real-time authorization challenge from a resource server.

3.  **Inability to Represent Delegation Chains.** Standard tokens cannot securely represent a multi-step delegation chain (`User -> Agent A -> Agent B`). This is a critical blocker for automating complex, multi-agent business processes.

4.  **Insufficient Revocation Mechanisms.** The single-token revocation API is inadequate. The lack of standardized APIs for task-level and bulk (per-user or per-client) revocation is a major operational and security deficiency.

5.  **Authorization Is Modeled Per-Client, Not Per-Group.** OAuth has no notion of a set of clients acting as one task group under a single grant: no group membership representation, no admission of late-selected members, and no group-level lifecycle or atomic revocation.

# Security Considerations

As we design new authorization mechanisms for agents, security must be the primary concern. The autonomy of agents amplifies the risk of any vulnerability.

*   **Risk of Over-Privileging:** The current lack of dynamic authorization tempts developers to request broad, long-lived permissions ("god tokens"), dramatically increasing the damage if an agent is compromised. Future solutions must make it easy to follow the Principle of Least Privilege.
*   **Delegation Chain Vulnerabilities:** Without a standard for secure delegation chains, custom implementations are prone to "Confused Deputy" attacks, where an agent is tricked into misusing its authority.
*   **Revocation Timeliness:** In a world of powerful, autonomous agents, the ability to instantly and completely revoke all permissions for a compromised user or agent is not a "nice-to-have"; it is an absolute necessity.
*   **Non-Repudiation:** For enterprise and B2B scenarios, actions taken by agents must be cryptographically auditable and non-repudiable, creating a strong digital paper trail.
*   **Coordinator as a Single Point of Authority:** In coordinated task groups, the leading agent both requests authorization on behalf of all members and assigns their tasks. If the coordinator is compromised, the blast radius covers the entire group; its authority to act as an applier for others must therefore be separately authenticated, authorized, and auditable.

# IANA Considerations

This document has no IANA actions.

# Acknowledgements

The analysis and use cases in this document are derived from observations of emerging AI agent technologies and their application trends across various industries. Thanks are due to the OAuth community for their past and ongoing efforts in building a secure and interoperable authorization framework, upon which this work is built.

--- back

# References

[RFC6749]
: Hardt, J., Ed., "The OAuth 2.0 Authorization Framework", RFC 6749, DOI 10.17487/RFC6749, October 2012, <https://www.rfc-editor.org/info/rfc6749>.

[RFC7009]
: Lodderstedt, T., Ed., Dronia, S., and M. Scurtescu, "OAuth 2.0 Token Revocation", RFC 7009, DOI 10.17487/RFC7009, August 2013, <https://www.rfc-editor.org/info/rfc7009>.

[RFC8693]
: Jones, M., Nadalin, A., Campbell, B., Ed., Bradley, J., and C. Mortimore, "OAuth 2.0 Token Exchange", RFC 8693, DOI 10.17487/RFC8693, January 2020, <https://www.rfc-editor.org/info/rfc8693>.

[RFC9396]
: Lodderstedt, T., Richer, J., and B. Campbell, "OAuth 2.0 Rich Authorization Requests", RFC 9396, DOI 10.17487/RFC9396, May 2023, <https://www.rfc-editor.org/info/rfc9396>.

[I-D.song-oauth-ai-agent-collaborate-authz]
: Song, Y., Li, L., Jiang, Y., and F. Liu, "OAuth2.0 Extension for Multi-AI Agent Collaboration", Work in Progress, Internet-Draft, draft-song-oauth-ai-agent-collaborate-authz-01, 28 February 2026, <https://datatracker.ietf.org/doc/html/draft-song-oauth-ai-agent-collaborate-authz-01>.
