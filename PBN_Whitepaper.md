# Protocol Boundary Network (PBN)

## A Security Layer for AI Agent Communication

**Author:** Tyler Walker  
**Date:** April 29, 2026  
**Version:** v3.3 technical draft

---

## Executive Summary

AI agents are moving from chat boxes into operational systems. They read files, query databases, call APIs, browse the web, send messages, run code, and trigger financial actions. Authorization can answer which agent may call which tool. It does not, by itself, answer whether data read from one authorized tool may be sent to another authorized tool later in the same session.

Protocol Boundary Network (PBN) is a proposed security layer for that data-flow problem. It places a mandatory, fail-closed gateway between agents and tool servers. The gateway verifies identity, enforces policy, tokenizes protected values before they enter the agent's context, checks outbound calls against destination scope and lineage, and writes a unified audit trail.

PBN does not claim to make the model safe. It assumes the model may be steered by prompt injection or malicious tool output. The security claim is narrower and more useful: a steered agent should not be able to move protected values across unauthorized tool boundaries merely because it was instructed to do so.

PBN is a synthesis of established primitives: tokenization vaults, information-flow tracking, mandatory gateways, identity, policy engines, and audit. The contribution is applying them as a practical enforcement contract around existing agent/tool stacks, especially MCP-style deployments.

---

## Table of Contents

1. Problem: Authorization Is Not Data-Flow Control
2. AI Tunneling
3. Threat Model
4. Architecture
5. MCP Deployment Model
6. Protocol and Policy Surface
7. Prototype and Red-Team Results
8. Prior Art and Differentiation
9. Core Enforcement Layers
10. Enterprise, Trading, and DeFi Use Cases
11. Limits
12. Adoption Path
13. Conclusion

---

## 1. Problem: Authorization Is Not Data-Flow Control

AI agents use a hub-and-spoke model. The agent sits in the middle. Tool servers sit around it.

```
                  +------------+
                  |  Database  |
                  +-----^------+
                        |
+------------+    +-----+------+    +------------+
|   Email    |<---| AI Agent   |--->|   Cloud    |
+------------+    +-----+------+    +------------+
                        |
                  +-----v------+
                  |  Browser   |
                  +------------+
```

The agent asks a tool to do something. The tool returns a result. The agent folds that result into its working context and decides what to do next.

This is powerful because the agent can combine capabilities. It is risky for the same reason. The agent becomes the bridge between systems that were never meant to share data directly.

Standard authorization helps, but it is not enough. OAuth can say an agent may call the database tool and may call the email tool. That does not automatically mean a specific customer record read from the database may be included in the next outbound email. That is a data-flow decision, not just an access decision.

PBN targets agent-tool boundary threats not consistently solved by authorization alone:

- **Prompt injection through tool responses:** malicious instructions enter through web pages, tickets, emails, documents, logs, or API responses.
- **Cross-tool data leakage:** sensitive data from one authorized tool flows into another authorized but inappropriate destination.
- **Identity and tool impersonation:** agents and tools need mutual identity and manifest validation.
- **Privilege chain escalation:** individually allowed calls combine into an unsafe sequence.
- **No unified audit trail:** each tool may log its own events, but no one sees the session-wide chain.

The common thread is not "bad tool call." It is unsafe movement across a chain of tool calls.

---

## 2. AI Tunneling

Traditional tunneling hides prohibited traffic inside allowed traffic. DNS tunneling hides data inside DNS queries. HTTPS tunneling hides one protocol inside another allowed protocol.

AI tunneling is different.

**The AI's context is the carrier.**

AI tunneling means protected data enters through one tool, travels through agent context, and exits through another tool as an authorized-looking action. Network traffic may look clean. The outbound request may be syntactically valid. The violation is in the meaning and lineage of the data, not in the packet format.

Three patterns matter:

- **Context smuggling:** an agent reads from one place, transforms or stores the result, and writes it somewhere else.
- **Injection relay:** a malicious instruction enters through one tool and triggers action through another.
- **Covert agent communication:** agents pass instructions through ordinary-looking text, logs, tickets, code comments, or generated outputs.

Content pattern matching alone is not enough. The security layer must understand lineage and enforce boundaries around data movement.

---

## 3. Threat Model

PBN's first hard target is direct movement of protected values across tool boundaries. It is not a complete AI-security system.

| Area | In scope | Out of scope / limit |
|---|---|---|
| Direct protected-value exfiltration | Yes | - |
| Token-in-payload exfiltration | Yes | - |
| Unauthorized tool destination | Yes | - |
| Semantic summaries / derivative leaks | Partial | Not fully solved by token checks alone |
| Compromised gateway | No | Requires infrastructure security, process isolation, and operator controls |
| Malicious downstream tool | Partial | Needs identity, manifest validation, and destination policy |
| Raw-code execution exploits | Partial | Not the main control; needs sandboxing and code-execution policy |
| Unstructured secret detection | Partial | Depends on classifiers, deterministic recognizers, or human review |
| Multi-agent handoff lineage | Partial | Requires explicit parent/child session propagation |

The most defensible PBN claim is:

> Under gateway mediation, protected values tokenized at ingress cannot be directly released to tools whose destination scope does not permit that classification.

That claim does not cover every semantic leak. It does cover the basic but dangerous chain where an agent is steered into copying protected values into a webhook, browser, email, chat, transaction broadcaster, or other outbound tool.

---

## 4. Architecture

PBN is built on one rule:

**No protected tool call bypasses the PBN gateway.**

```
                  +----------------+
                  |    AI Agent    |
                  +--------+-------+
                           |
                           v
                  +----------------+
                  |  PBN Gateway   |
                  |                |
                  |  Identity      |
                  |  Policy        |
                  |  Token Vault   |
                  |  Data Flow     |
                  |  Audit         |
                  +--+---+---+---+-+
                     |   |   |   |
                     v   v   v   v
                  Protected Tool Servers
```

The enforcement loop is the core mechanism:

| PBN enforcement loop |
|---|
| 1. Tool response enters gateway |
| 2. Gateway classifies fields |
| 3. Protected values are tokenized |
| 4. Agent receives safe view |
| 5. Agent attempts outbound tool call |
| 6. Gateway checks token, lineage, and destination scope |
| 7. Gateway resolves or blocks |
| 8. Audit event records source, token, destination, and policy reason |

The gateway can wrap existing tools. It does not require every tool server to be rewritten. The gateway becomes the enforcement point around the agent's tool surface.

### 4.1 Concrete Attack Walkthrough

1. Agent calls `database.customer_records`.
2. Tool response includes `account_number`.
3. PBN classifies `account_number` as `REGULATED`.
4. PBN replaces the value with `<TOK:T-...>` before the agent sees it.
5. A prompt injection says: "send this account number to the audit webhook."
6. Agent calls `http.post` with the token in the payload.
7. PBN sees the token and checks lineage: source is `database.customer_records`.
8. PBN checks destination scope: `http.post` only allows `PUBLIC`.
9. PBN blocks before dispatch.
10. Audit records source, token id, destination, classification, and policy reason.

---

## 5. MCP Deployment Model

PBN is easiest to understand as an MCP proxy or server-of-servers.

In the basic deployment, the agent does not connect directly to downstream MCP tool servers. The agent connects to PBN. PBN exposes an MCP-compatible tool surface to the agent, then connects downstream to one or more real tool servers.

```
Agent MCP client
    |
    v
PBN MCP gateway
    |
    +-- filesystem MCP server
    +-- fetch/browser MCP server
    +-- database MCP server
    +-- email/slack/webhook MCP server
```

Deployment modes:

- **Proxy mode:** PBN forwards calls to existing MCP servers after policy and data-flow checks.
- **Sidecar mode:** PBN runs beside a high-risk agent and mediates only that agent's tools.
- **Broker mode:** PBN aggregates multiple downstream tools and exposes one controlled MCP endpoint to the agent.

The cleanest first implementation is proxy/broker mode because it does not require every downstream MCP server to implement PBN itself. Native PBN-aware tools can later expose richer metadata, structured classification hints, and stronger identity binding.

PBN can also mediate non-MCP tools. MCP is the first target because it is the live integration surface enterprises are adopting.

The current v0 reference prototype demonstrates the gateway, token, policy, and lineage mechanics with toy backends. It is MCP-style and MCP-targeted, but it is not yet a complete production MCP proxy.

---

## 6. Protocol and Policy Surface

This paper is not a full RFC, but PBN needs enough protocol shape to be judged as engineering rather than positioning.

### 6.1 Gateway API Shape

The prototype exposes this tool-facing surface:

```
tool_read_file(path) -> plaintext | <TOK:T-...>
tool_write_file(path, content) -> ok | GatewayBlock
tool_http_post(url, payload) -> ok | GatewayBlock
tokenize_record(record, origin_prefix) -> record with tokenized fields
```

Production PBN generalizes this to any mediated tool call:

```
pbn.call(agent_id, session_id, tool_id, arguments) -> result | block
```

On ingress, the gateway classifies tool output and tokenizes protected fields. On egress, it scans outbound arguments for token references and resolves them only if the destination scope permits release.

### 6.2 Token Format

The prototype's agent-visible token reference is:

```
<TOK:T-32-hex-characters>
```

Example:

```
<TOK:T-55b9ceaef0ab0a3f100eb0d6072da3a1>
```

The visible token is an opaque handle. It carries no embedded classification, source, tenant, or plaintext. Those facts live in the gateway registry and audit log.

Production token records should include:

- token id
- classification
- origin tool
- origin field/path
- tenant/session binding
- creation time
- expiration time
- integrity tag or registry-backed authenticity

### 6.3 Policy Shape

Classifier policy describes data. Tool scope describes destinations. Flow policy describes cross-tool movement.

```yaml
classifiers:
  - match: "transactions[*].account_number"
    classification: REGULATED
    validator: account_number_shape

tools:
  slack.post_message:
    max_classification: INTERNAL
    destinations:
      - "workspace://company-secops"

  http.post:
    max_classification: PUBLIC
    destinations:
      - "https://approved.example/*"

flows:
  - from: database.customer_records
    to: slack.post_message
    allow: false
```

A tool may be callable while still not being allowed to receive a specific class of data from a specific origin.

### 6.4 Failure Semantics

PBN must fail closed.

- Gateway down: protected tools are unavailable through PBN.
- Unknown classification: block or require human/policy review.
- Classifier confidence too low: classify as UNKNOWN and block release.
- Ambiguous policy: deny by default.
- Backend error: do not leak partial data; return an opaque refusal.
- Scope violation: block before dispatching to the destination.

The prototype returns one agent-visible message for refusals:

```
gateway refused this operation
```

The detailed reason stays in the gateway audit log. This avoids turning error messages into a classification oracle for the agent.

### 6.5 Minimum Viable PBN

A first useful PBN product does not need every future research feature. It needs:

- MCP-compatible proxy/broker
- tool identity registry
- classification rules
- token vault
- destination scopes
- lineage log
- policy engine
- audit/export endpoint

That minimum product is enough to demonstrate the core value: connect an existing agent to existing tools through PBN and show a protected value blocked from an unauthorized destination with a readable lineage trail.

---

## 7. Prototype and Red-Team Results

The current reference implementation demonstrates:

- file-level tokenization demo
- per-field financial-record demo
- scope-bound token resolution
- HTTP destination allowlist enforcement
- lineage rendering
- red-team scenarios for direct exfil, token-in-payload, token-in-URL, encoded-token smuggling, path traversal, write-smuggling, nested payloads, multi-token payloads, and multi-step chains

The demo shape:

```
agent obeys injection
gateway tokenizes protected data
agent attempts exfil
gateway blocks unauthorized destination
lineage shows the chain
```

Local verification for this draft:

- `py -3.12 -m pytest -q` with plugin autoload disabled: 159 passed
- `py -3.12 -m pytest tests/test_redteam.py -q`: 33 passed
- `python -m demo.redteam.run_redteam`: all eleven implemented scenarios blocked with PBN
- `python -m demo.run_compare`: credentials leak without PBN and are blocked with PBN
- `python -m demo.run_financial`: per-field account number exfil is blocked

Implemented red-team outcome summary:

| Scenario | Without PBN | With PBN |
|---|---:|---:|
| Direct secret exfil | Leaks | Blocked |
| Context smuggling | Leaks | Blocked |
| Token copy to webhook | Leaks | Blocked before dispatch |
| Token in URL query | Leaks | Blocked before dispatch |
| Base64 / hex encoded token | Leaks encoded credential without PBN | Blocked as encoded token smuggling |
| Nested JSON payload | Leaks | Blocked |
| Multiple tokens in one payload | Leaks | Blocked |
| Unknown path / path traversal | Leaks planted value | Blocked at classification |
| Path/write smuggling | Possible | Blocked by destination classification |
| Multi-step chain | Leaks | Blocked |

Not yet covered by the implemented red-team suite:

| Scenario | Current status |
|---|---|
| Approved internal destination / URL allowlist | Covered by gateway positive tests; not modeled as a red-team attack |
| Unstructured secret in email | Depends on classifier / recognizer; needs separate classifier benchmark |
| Derivative summary leak | Possible; not fully blocked by token checks alone |

Prototype latency check:

- 5,000 local in-process iterations
- read-file token minting followed by blocked HTTP-post egress
- median: 0.0203 ms
- p95: 0.0228 ms
- max: 0.1907 ms

These are not production MCP proxy latency numbers. Real deployments add transport, serialization, downstream tool latency, policy I/O, audit I/O, and vault/HSM overhead. The narrow conclusion is that the core token/scope check is not the expensive part of an LLM-agent workflow.

---

## 8. Prior Art and Differentiation

PBN should not claim that its building blocks are new. They are not.

| Existing primitive | Examples | What PBN changes |
|---|---|---|
| Tokenization vaults | PCI tokenization, Skyflow, VGS, Piiano | Moves scope-bound token release to the AI agent-tool boundary |
| Mandatory gateways | Cloudflare AI Gateway, Portkey, Kong AI plugins | Uses the gateway as a data-flow enforcement point, not only routing, cost, caching, or observability |
| Information-flow control | Taint tracking, IFC systems, DLP lineage | Applies lineage to agent tool calls and token resolution across a session |
| Capability-based agent design | CaMeL and related agent-security work | Complements capability restriction with containment after a steered agent attempts a data move |
| Crypto custody policy engines | Fireblocks-style signing policy, HSM/KMS systems | Applies pre-action policy and signer isolation ideas to broader AI tool actions, not only custody |

CaMeL is the closest conceptual neighbor because it attacks prompt injection by redesigning the agent execution model: trusted control flow is separated from untrusted data, and a capability-bounded execution language prevents the untrusted side from exceeding the planner's authority.

The distinction should be direct:

> CaMeL redesigns the agent execution model. PBN constrains tool-boundary data movement around existing agents and MCP servers.

That makes PBN less general as a formal agent architecture, but more directly deployable as middleware in current tool stacks.

The defensible originality claim is therefore narrow:

**PBN is a synthesis of established security primitives into a mandatory, lineage-aware gateway for agent-tool communication.**

The novel part is not tokenization alone, a gateway alone, or audit logging alone. The novel part is the integration contract:

- all protected agent-tool calls pass through the same enforcement point
- protected values are tokenized before entering the agent's context
- token release is scoped to the destination tool
- lineage ties reads, token minting, attempted sends, blocks, and allowed releases into one chain
- policy evaluates data movement across tools, not only tool access in isolation

The standard objection is "this is just DLP plus tokenization plus an AI gateway." The answer:

**Traditional DLP sees content. PBN sees the agent-tool chain. AI gateways see model traffic. PBN mediates tool calls, token release, destination scope, and session lineage at the point where agents actually move data between systems.**

That distinction is the whole game.

---

## 9. Core Enforcement Layers

### 9.1 Identity Layer

The Identity Layer verifies who is participating.

It binds calls to:

- agent identity
- user identity where applicable
- tool identity
- session identity
- environment or tenant

This layer composes with OAuth, mTLS, signed tool manifests, service identity, and enterprise IAM. PBN does not replace those systems. It uses them to answer who is calling and under what authority.

### 9.2 Policy Layer

The Policy Layer decides what is allowed.

Policies can cover:

- which agents may call which tools
- which classifications each tool may receive
- rate limits
- time and environment restrictions
- allowed or blocked tool-call sequences
- destination allowlists
- tenant boundaries
- emergency lockouts

The policy question is not only "can this agent call this tool?" It is also "can this agent send this data to this tool at this point in the session?"

### 9.3 Tokenization and Data Protection Layer

When protected data enters through a tool response, PBN classifies it. Sensitive fields can be replaced before they reach the agent:

```
account_number: <TOK:T-8f12...>
routing_number: <TOK:T-91ac...>
amount: 1247.50
merchant: PUBLIC_VALUE
```

The agent can reason over the fields it is allowed to see. It can carry token references through the workflow. It does not automatically receive the raw protected values.

When the agent later calls another tool, the gateway checks token references in the outgoing payload. If the destination is authorized for that classification, the gateway can resolve the token. If not, the call is blocked or the token remains unresolved depending on policy.

### 9.4 Data Flow Layer

The Data Flow Layer tracks where data came from and where it tries to go.

It records:

- source tool
- classification
- token minting
- attempted resolution
- allowed sends
- blocked sends
- policy reasons
- session chain

This is what lets PBN catch cross-tool movement. The gateway does not only see a single outgoing call. It sees the call in the context of what the agent previously read.

### 9.5 Audit Layer

The Audit Layer writes the record.

Every significant event should be logged in append-only form:

- tool request
- tool response metadata
- classification decision
- token creation
- resolve attempt
- policy allow or block
- destination
- timestamp
- agent/session identity

This gives operators visibility, incident response, compliance evidence, and a baseline for detecting abnormal agent behavior.

---

## 10. Enterprise, Trading, and DeFi Use Cases

### 10.1 Enterprise AI Agents

Enterprise agents are the first obvious market.

They touch customer records, CRM systems, internal documents, source code, cloud consoles, support tickets, finance systems, email, and chat. PBN gives these organizations a controlled way to let agents work without turning every connected tool into a shared unbounded context.

### 10.2 Trading Bots and Autonomous Finance

Trading agents and portfolio agents need stronger boundaries because mistakes have direct financial consequences.

PBN can enforce:

- which exchanges or brokers a bot can call
- dry-run vs live mode boundaries
- per-session transaction ceilings
- destination allowlists
- strategy-specific tool scopes
- key isolation
- audit trails for every attempted action

The default safe design is that agents do not receive raw signing material. They request actions through a gateway that validates scope and policy.

### 10.3 DeFi and Wallet Automation

PBN cannot fix vulnerable smart contracts and it cannot stop an on-chain exploit at the protocol level once a signed transaction is valid. The useful boundary is before signing and broadcasting.

For DeFi automation, PBN can guard:

- wallet or signer access
- allowed contract addresses
- allowed function selectors
- max value per transaction
- max slippage
- chain ID and destination checks
- flash-loan-like action sequences by policy
- oracle-dependent transaction conditions before signing

This overlaps with Fireblocks-style custody policy engines. The PBN difference is that the same policy boundary can cover non-custody AI tool calls too: database reads, browser calls, notifications, analysis tools, and transaction requests in one agent session.

---

## 11. Limits

PBN is a boundary layer, not magic.

It does not make an unsafe model safe. It does not prove all prompt injection impossible. It does not sanitize every output. It does not secure a compromised gateway. It does not replace IAM, secrets management, endpoint security, model hardening, or human review.

| What PBN is not |
|---|
| Not a model alignment system |
| Not a replacement for IAM, OAuth, or MCP authorization |
| Not a replacement for DLP |
| Not a full semantic-leak prevention system |
| Not a smart-contract exploit defense |
| Not useful unless the gateway is mandatory |

### 11.1 Unstructured Data

Structured records are the clean first target. A transaction object with `account_number`, `routing_number`, `amount`, and `merchant_memo` can be classified field by field.

Most agent inputs are messier: emails, tickets, chat history, logs, markdown, web pages, code comments, and prose. PBN can still help, but the classification problem is harder. A credible deployment needs one of these patterns:

- extract structured fields from unstructured text, then tokenize the extracted fields
- use deterministic recognizers for known secrets, keys, IDs, account numbers, and regulated identifiers
- treat LLM/NLP classification as advisory unless backed by confidence thresholds and fail-closed behavior
- route low-confidence or ambiguous content to human review
- scope unstructured tools more tightly than structured, schema-backed tools

PBN should not claim that an NLP classifier is magically more trustworthy than the agent. The gateway classifier is useful only if it is constrained, tested, audited, and allowed to fail closed.

### 11.2 Derivative Leaks

Tokenization protects against direct copying of protected values. It does not automatically prevent semantic leakage.

If an agent reads many protected records and posts "the VIP customer in New York has six suspicious wire transfers totaling about $90,000" to Slack, the payload may contain no token reference and still leak sensitive information.

This is the derivative-leak problem. PBN can reduce it with aggregation policies, output review, destination restrictions, human-in-the-loop workflows, and future derivation tracking. The v3 claim should stay narrower: PBN blocks unauthorized release of protected tokenized values and gives operators lineage to reason about broader leakage.

### 11.3 Multi-Agent Topologies

The simple diagram shows one agent and many tools. Real deployments increasingly use supervisor agents, worker agents, delegated sub-agents, and agent-to-agent handoffs.

PBN should extend lineage across those handoffs:

- parent session id
- child agent id
- delegated tool scope
- inherited data classifications
- handoff audit event
- revocation of child scopes

Until that is specified, the strongest claim is for a mediated agent-tool session, not every multi-agent topology.

### 11.4 Fit

PBN fits best when:

- tools are mediated through a gateway
- sensitive fields can be classified
- destinations can be scoped
- the agent can still do useful work without seeing every raw value
- auditability matters

PBN fits poorly when:

- the whole task requires raw sensitive text
- the main threat is code execution rather than data movement
- latency budgets are extremely tight
- the gateway cannot be made mandatory
- the classification policy is not trustworthy

These limits define where it should be sold and where it should not.

---

## 12. Adoption Path

PBN should support staged adoption.

**Audit mode:** log every tool call and data-flow event, block nothing. This gives immediate visibility into what agents are actually doing.

**Policy mode:** enforce identity, tool allowlists, destination controls, and chain restrictions.

**Full protection mode:** turn on tokenization, scope-bound release, lineage enforcement, and high-confidence blocking.

Deployment models:

- **Centralized gateway:** one gateway for many agents and tools.
- **Per-agent gateway:** each high-risk agent gets its own boundary.
- **Hybrid:** shared gateway for common tools, dedicated gateways for privileged agents.

Most organizations should start centralized for visibility, then move high-risk agents into isolated gateways.

---

## 13. Conclusion

AI agents are becoming a new communication layer between critical systems. That layer needs security controls designed for how agents actually work.

The risk is not only that an attacker calls a tool. The risk is that an agent reads from one trusted system, gets steered by untrusted content, and writes to another trusted system while every individual call appears authorized.

PBN is a fail-closed gateway for AI-agent tool communication, combining identity, policy, data protection, lineage, and audit.

The strongest claim is not that every part of PBN is unprecedented. The strongest claim is that the agent-tool boundary needs a dedicated security layer, and PBN is Tyler Walker's proposed synthesis for that layer.

**Agents can use tools. Data cannot cross boundaries just because the agent was talked into moving it.**

That is the security layer AI communication needs before the breach pattern becomes normal.
