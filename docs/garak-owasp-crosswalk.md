# Garak Probe → OWASP Top 10 Crosswalk (LLM / Agentic / MCP)

**Scope:** garak **0.16.0** (released 2026-08-04) — 41 active probe modules. Verified against the released sdist; probe list and `owasp:*` tags are identical to `main` at time of writing. Pin `garak==0.16.0` to reproduce this table.
**Taxonomies:** OWASP Top 10 for LLM Applications 2025 · OWASP Top 10 for Agentic Applications 2026 (ASI01–ASI10, published 9 Dec 2025) · OWASP MCP Top 10 v0.1

---

## 0. Critical caveat before you use garak's own tags

Garak probes ship with native `owasp:llmNN` tags in `probe.tags`. **These use the 2023 v1.1 numbering, not the 2025 list.** Ingesting them directly into a 2025-aligned report will silently mis-map roughly half your findings.

Proof from the source: `leakreplay` is tagged `owasp:llm06, owasp:llm10` (2023: Sensitive Info Disclosure + Model Theft). `agent_breaker` is tagged `owasp:llm07, owasp:llm08` (2023: Insecure Plugin Design + Excessive Agency). Neither makes sense under 2025 numbering.

### 2023 → 2025 renumbering key

| 2023 (garak tags) | 2025 (report target) |
|---|---|
| LLM01 Prompt Injection | LLM01 Prompt Injection |
| LLM02 Insecure Output Handling | LLM05 Improper Output Handling |
| LLM03 Training Data Poisoning | LLM04 Data & Model Poisoning |
| LLM04 Model DoS | LLM10 Unbounded Consumption |
| LLM05 Supply Chain | LLM03 Supply Chain |
| LLM06 Sensitive Info Disclosure | LLM02 Sensitive Information Disclosure |
| LLM07 Insecure Plugin Design | LLM06 Excessive Agency (closest fit) |
| LLM08 Excessive Agency | LLM06 Excessive Agency |
| LLM09 Overreliance | LLM09 Misinformation |
| LLM10 Model Theft | *(dropped; partially LLM02)* |
| — | LLM07 System Prompt Leakage *(new)* |
| — | LLM08 Vector & Embedding Weaknesses *(new)* |

Extract the raw native tags with:
```bash
garak --list_probes --verbose
# or programmatically:
python -c "import garak._plugins as p; [print(k, p.plugin_info(k).get('tags')) for k,a in p.enumerate_plugins('probes')]"
```

---

## 1. Garak → OWASP LLM Top 10 (2025)

| 2025 ID | Category | Garak probe modules | Coverage |
|---|---|---|---|
| LLM01 | Prompt Injection | `promptinject`, `latentinjection`, `dan`, `encoding`, `smuggling`, `badchars`, `ansiescape`, `dra`, `fitd`, `sata`, `suffix`, `tap`, `atkgen`, `goat`, `adaptive_attacks`, `grandma`, `doctor`, `phrasing`, `continuation`, `visual_jailbreak`, `audio`, `goodside` | **Strong** |
| LLM02 | Sensitive Information Disclosure | `leakreplay`, `propile`, `divergence`, `apikey`, `web_injection` (markdown exfil), `grandma` | **Strong** |
| LLM03 | Supply Chain | `packagehallucination`, `fileformats`, `av_spam_scanning` | Moderate |
| LLM04 | Data & Model Poisoning | `divergence`, `leakreplay` (memorisation evidence only), `glitch` | **Weak** — garak is a black-box output scanner; it evidences poisoning, it can't test the pipeline |
| LLM05 | Improper Output Handling | `web_injection` (`MarkdownXSS`, `TaskXSS`, markdown exfil), `exploitation` (`SQLInjectionSystem/Echo`, `JinjaTemplatePythonInjection`), `ansiescape`, `malwaregen`, `packagehallucination` | **Strong** |
| LLM06 | Excessive Agency | `agent_breaker`, `exploitation`, `latentinjection` (indirect) | Moderate |
| LLM07 | System Prompt Leakage | `sysprompt_extraction`, `promptinject`, `suffix` (GCG goal = disregard system prompt), `goodside` | **Strong** |
| LLM08 | Vector & Embedding Weaknesses | `latentinjection` (RAG-borne injection only) | **Gap** |
| LLM09 | Misinformation | `snowball`, `misleading`, `packagehallucination`, `donotanswer`, `goat`, `goodside`, `lmrc` | **Strong** |
| LLM10 | Unbounded Consumption | `divergence` (repeated-token divergence, partial) | **Gap** |

---

## 2. Garak → OWASP Top 10 for Agentic Applications 2026 (ASI)

No official garak→ASI mapping exists. This is an inferred crosswalk.

| ID | Risk | Garak probe modules | Coverage |
|---|---|---|---|
| ASI01 | Agent Goal Hijack | `latentinjection`, `agent_breaker`, `promptinject`, `web_injection`, `goat`, `tap`, `atkgen` | **Strong** |
| ASI02 | Tool Misuse & Exploitation | `agent_breaker` (tool manipulation — its stated goal), `exploitation`, `apikey` | Moderate |
| ASI03 | Identity & Privilege Abuse | `apikey`, `leakreplay`/`propile` (credential surfacing only) | **Gap** — no scope/token-escalation probes |
| ASI04 | Agentic Supply Chain | `packagehallucination`, `fileformats`, `av_spam_scanning` | Moderate |
| ASI05 | Unexpected Code Execution (RCE) | `exploitation` (Jinja RCE, SQLi), `malwaregen`, `agent_breaker` | Moderate |
| ASI06 | Memory & Context Poisoning | `latentinjection`, `divergence` | **Gap** — garak is largely single-turn/stateless; no cross-session memory writes |
| ASI07 | Insecure Inter-Agent Communication | — | **Gap** — nothing in garak tests A2A channels |
| ASI08 | Cascading Failures | — | **Gap** — no multi-agent blast-radius harness |
| ASI09 | Human-Agent Trust Exploitation | `misleading`, `snowball`, `packagehallucination`, `doctor`, `lmrc` | Moderate |
| ASI10 | Rogue Agents | `agent_breaker`, `atkgen` (persistence untested) | **Gap** — requires behavioural baselining, not prompt probing |

**Headline:** garak covers roughly ASI01/02/05 well and leaves ASI03, ASI06–ASI08, ASI10 essentially untested. `agent_breaker` (`IterativeProbe`, goal: *"Identify weaknesses in agentic applications through tool manipulation"*) is the only genuinely agent-aware module.

---

## 3. Garak → OWASP MCP Top 10 (v0.1)

| ID | Risk | Garak probe modules | Coverage |
|---|---|---|---|
| MCP01 | Token Mismanagement & Secret Exposure | `apikey` (`GetKey`, `CompleteKey`), `leakreplay`, `propile`, `web_injection` exfil | Moderate |
| MCP02 | Privilege Escalation via Scope Creep | `agent_breaker` | **Gap** |
| MCP03 | Tool Poisoning (rug pull / schema poisoning / shadowing) | `agent_breaker`, `latentinjection` | **Gap** — no probe reads or mutates tool descriptions |
| MCP04 | Supply Chain & Dependency Tampering | `packagehallucination`, `fileformats`, `av_spam_scanning` | Moderate |
| MCP05 | Command Injection & Execution | `exploitation`, `malwaregen`, `ansiescape` | **Strong** |
| MCP06 | Intent Flow Subversion (prompt injection via context) | `latentinjection`, `promptinject`, `web_injection`, `goat`, `tap`, `smuggling` | **Strong** |
| MCP07 | Insufficient AuthN/AuthZ | — | **Gap** — protocol layer, out of garak's scope by design |
| MCP08 | Lack of Audit & Telemetry | — | **Gap** — design control, not prompt-testable |
| MCP09 | Shadow MCP Servers | — | **Gap** — inventory/discovery problem |
| MCP10 | Context Injection & Over-Sharing | `latentinjection`, `divergence`, `leakreplay`, `propile` | Moderate |

---

## 4. Coverage summary for the report

| Taxonomy | Well covered | Partial | Not covered |
|---|---|---|---|
| LLM Top 10 2025 | 5 / 10 | 3 / 10 | 2 / 10 |
| Agentic ASI 2026 | 1 / 10 | 4 / 10 | 5 / 10 |
| MCP Top 10 v0.1 | 2 / 10 | 4 / 10 | 4 / 10 |

**The structural reason for the gaps:** garak is a single-turn, black-box, prompt-in/text-out scanner with a probe→detector architecture. Every uncovered category is one that requires *state* (memory across sessions), *protocol inspection* (MCP auth, tool schemas), *multi-agent topology* (A2A, cascades), or *runtime telemetry* (audit, rogue-agent baselining) — none of which garak models.

**Implication for the hybrid POC:** garak is a good regression-grade layer for LLM01/02/05/07/09 and MCP05/06. It will not close ASI03, ASI06–08, ASI10, or MCP02/03/07–09. Those need PyRIT multi-turn orchestrators for memory poisoning and goal drift, Promptfoo's agentic plugins plus custom tool-schema mutation for tool poisoning and scope creep, and OTel trace assertions rather than output detectors for the audit/rogue-agent categories.

## 5. Notes on tag fidelity

Some native tags are questionable and shouldn't be trusted unreviewed:

- `topic` → `owasp:llm10` (2023 Model Theft) — the probe tests restricted-topic traversal; the tag looks wrong.
- `glitch` → `owasp:llm05` (2023 Supply Chain) — defensible only as a model-artefact argument.
- `apikey`, `atkgen`, `suffix`, `tap`, `malwaregen`, `lmrc`, `realtoxicityprompts`, `audio`, `badchars` carry **no** OWASP tag at all despite obvious mappings.

Recommend maintaining your own mapping table as the source of truth and treating garak's tags as a hint, not an authority.
