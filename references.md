# References

Prior art and external sources cited in PBN whitepaper v2 and threat model.

## Indirect Prompt Injection

- **Greshake, K. et al. (2023).** *Not what you've signed up for: Compromising Real-World LLM-Integrated Applications with Indirect Prompt Injection.* arXiv:2302.12173.
  - Foundational taxonomy of indirect prompt injection. Cite for: tool-response injection threat (whitepaper §3.1, §4.3).

- **Willison, S. (2025).** *The lethal trifecta for AI agents: private data, untrusted content, and external communication.* simonwillison.net.
  - The three-ingredient framing PBN's defense maps onto. Cite for: motivation, why scope-bound resolution matters. PBN's contribution is mechanism, not the framing itself — credit upfront.

## Information Flow Control (the IFC tradition PBN inherits from)

- **Efstathopoulos, P. et al. (2005).** *Labels and event processes in the Asbestos operating system.* SOSP '05.
  - Process-level information flow labels. PBN's tokens are conceptually an IFC label transferred across an agent boundary.

- **Zeldovich, N. et al. (2006).** *Making information flow explicit in HiStar.* OSDI '06.
  - Decentralized IFC. Relevant to multi-tenant gateway designs.

- **Krohn, M. et al. (2007).** *Information flow control for standard OS abstractions.* SOSP '07. (Flume.)
  - IFC at the syscall boundary. PBN's analogue: IFC at the MCP tool-call boundary.

- **Myers, A. C. (1999).** *JFlow: Practical mostly-static information flow control.* POPL '99. (Jif.)
  - Static IFC in a Java extension. Cite for: precedent that label-tracking can be made practical without crippling expressivity.

## Agent Architectures

- **Debenedetti, E., Tramèr, F. et al. (DeepMind, 2025).** *CaMeL: Defeating Prompt Injections by Design.*
  - Dual-LLM / capability-based design where a "privileged" planner cannot see untrusted content. Closest live work to PBN. Compare/contrast: CaMeL splits the LLM; PBN splits the data with tokens. Both are capability-style answers to the lethal trifecta.

## Protocol Specs

- **Model Context Protocol specification.** *Authorization.* modelcontextprotocol.io/specification.
  - MCP's authorization story is OAuth 2.1 + Resource Indicators. PBN sits *above* this — even fully-OAuth'd MCP is vulnerable to lethal-trifecta exfiltration because OAuth authenticates the connection, not the data flow. Be precise: don't claim MCP "has no security" in v2 — claim it has no *data-flow* security.

## Tokenization (the operational analogue)

- **PCI Security Standards Council.** *Tokenization Product Security Guidelines.* pcisecuritystandards.org.
  - Industry-standard token vault model. PBN's tokenizer borrows directly: opaque token, scope-bound resolution, vault-as-trust-boundary. Citing this is how we land "PBN is not novel cryptography — it's IFC + PCI tokenization applied at the agent boundary."

## Notes for v2

- Treat every citation as load-bearing — if we can't articulate *what specifically* we inherit from a paper, drop it.
- Volunteered residual channels (timing, token-count, classification-label leakage, gateway compromise) need their own §; cite Asbestos's covert-channel discussion as precedent that enumeration > denial.
