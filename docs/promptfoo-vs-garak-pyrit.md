# Would Promptfoo fill real gaps in our garak + PyRIT red-teaming stack?

*A quick comparison for red teaming our LLM apps and agents*

---

## The short version

garak and PyRIT do a good job of attacking the **model**. As we move to agents that call tools, use MCP and keep memory, more of our risk sits in the **application around the model**. That's where Promptfoo is strongest. It wouldn't replace what we have. It covers the layer our current tools mostly leave to us to build ourselves.

---

## Where Promptfoo fills a gap

| Area | Where we are today (garak + PyRIT) | What Promptfoo brings |
|---|---|---|
| **Attack generation** | garak uses a fixed probe library. With PyRIT, we write the objectives and orchestrators ourselves. | Writes attacks to fit each app's stated purpose, user roles and tools, so the tests match how the app is actually used. |
| **Agent and tool attacks** | Mostly model or chat level. PyRIT covers indirect prompt injection (XPIA). Anything else we build ourselves. | Ready-made plugins for tool misuse, excessive agency, unauthorised access to objects or functions (BOLA/BFLA), SSRF, MCP abuse, memory poisoning and RAG poisoning. |
| **Agent framework targets** | Generic REST/HTTP targets, plus glue code we maintain | Built-in connectors for the OpenAI Agents SDK, the Claude Agent SDK and Azure AI Foundry Agents, with OTLP tracing |
| **Compliance mapping** | garak tags its probes with some OWASP categories. PyRIT doesn't map to frameworks. | Reports mapped to OWASP LLM Top 10, OWASP Top 10 for Agentic Applications, NIST AI RMF and MITRE ATLAS. Useful for audit and risk reviews. |
| **CI/CD gating** | garak's CLI works in pipelines. PyRIT is mostly run from code or notebooks. | One YAML config with pass/fail thresholds, so red teaming can run on every pull request. |
| **Reporting** | garak gives JSONL/HTML. PyRIT keeps results in a local database with no dashboard. | Web report showing severity, results by attack type and history across runs. Easy to share with people outside the team. |
| **Quality evals** | Security only | Quality tests and red teaming use the same setup, so app teams only learn one tool. |
| **Model-file scanning** | Not covered | ModelAudit (now a separate open-source tool) checks model files such as pickles for malicious code before we load them. |
| **Multi-turn jailbreaks** | PyRIT already handles these well (Crescendo, PAIR, TAP) | Overlaps (Crescendo, GOAT, Hydra), so **not a real gap** |

---

## Where it doesn't help, and what to watch

- **It adds to the stack rather than replacing it.** garak still covers more model-level probes (encodings, glitch tokens, package hallucination). PyRIT is still the more flexible tool for custom converters, orchestration and scoring. We'd keep both.
- **Data residency comes first.** By default, Promptfoo sends some attack-generation requests to its hosted service. For us, that has to be turned off, with generation pointed at our in-house attacker model, before any internal data or system prompts go near it.
- **It adds a Node.js toolchain.** Promptfoo runs on Node.js, while the rest of our stack is Python. It's manageable, but it's another runtime to harden, scan and patch in our images.
- **Some features need the paid tier.** Continuous or scheduled red teaming, team dashboards and guardrails sit in the enterprise product. The open-source CLI covers what's in the table above.
- **Ownership has changed.** OpenAI bought Promptfoo in March 2026 and has promised to keep it open source under its current MIT license and to keep supporting other model providers. That seems fine for now, but we should watch the roadmap.

---

## Bottom line

If we bring Promptfoo in, the main new value is:

1. **Attacks on agents and their tools** that we'd otherwise build ourselves
2. **Attacks tailored to each app** instead of one generic probe set
3. **CI gates mapped to compliance frameworks**, so findings are easy to take to risk and audit

A small pilot on one agentic use case would be a reasonable next step: run it side by side with garak and PyRIT, with remote generation turned off.

---

### Sources

- [OpenAI to acquire Promptfoo](https://openai.com/index/openai-to-acquire-promptfoo/)
- [Promptfoo release notes](https://www.promptfoo.dev/docs/releases)
- [Promptfoo: testing prompts and agents (jacar.es)](https://jacar.es/en/promptfoo-testing-prompts-and-agents/)
- [Promptfoo vendor profile (guptadeepak.com)](https://guptadeepak.com/agent-security/vendor/promptfoo/)
- [Promptfoo overview (latent.space)](https://latent.space/wiki/promptfoo-32060fb2/)
