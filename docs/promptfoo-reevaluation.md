# Promptfoo for AI Red Teaming: Re-evaluation

| | |
|---|---|
| Date | 12 November 2026 |
| Status | Draft for discussion |
| Scope | Suitability of Promptfoo as an AI red-teaming framework, against the current garak + PyRIT stack |
| Constraint baseline | Air-gapped execution, zero runtime egress, attack corpora and failure traces stay in-house |

---

## 1. Summary

**Recommendation:** Do not adopt Promptfoo as the primary red-teaming framework. Keep garak (baseline breadth) and PyRIT (depth, multi-turn, agentic) as the strategic stack, and proceed with the PyRIT budget. Promptfoo can at most be a CI regression harness, run in fully local mode, and only after a two-week proof of concept with clear exit criteria (Section 10).

**Key points**

1. **Ownership changed.** OpenAI announced its acquisition of Promptfoo on 9 March 2026 and plans to integrate it into OpenAI Frontier [S1]. The core stays MIT open source [S2]. We found no public notice that the deal has closed, so confirm this with the vendor.
2. **Data leaves our environment by default.** Without our own model key, test generation and grading run on api.promptfoo.app. That includes the prompts sent to our targets and the targets' responses [S3]. With an OpenAI key set, the same traffic goes to OpenAI directly [S3].
3. **Local mode removes most of the reason to use it.** Everything that would set Promptfoo apart from garak + PyRIT is remote-only and produces no tests when remote generation is off [S3][S7][S12]: the 12 financial-services plugins, BOLA/BFLA, MCP, RAG poisoning and exfiltration, indirect prompt injection, the harmful-content and bias categories, and two of the three default strategies.
4. **The free tier is capped.** The Community edition allows 10,000 probes per month, and scans are blocked once the cap is reached [S5]. The cap still applies when we use our own models [S4]. At CI or fleet scale we would exceed it several times over (Section 7.2), which forces a paid tier.
5. **Full coverage inside our constraint needs Enterprise On-Prem.** This is a commercial contract, with pricing not published [S6], with a company that OpenAI is acquiring.
6. **Genuine strengths remain:** CI-native YAML configs, a web report, OWASP LLM and Agentic Top 10 mapping, and remediation guidance in reports [S18][S19]. These are useful, but in local mode they are mostly reporting and workflow conveniences that can be built on PyRIT.
7. **Cost:** inference cost is small and roughly the same for every option. The deciding costs are the On-Prem licence, integration effort (estimated 6–10 person-weeks), and third-party risk review (Section 7).

---

## 2. What changed since the last evaluation

| Item | Change | Source |
|---|---|---|
| Promptfoo ownership | OpenAI acquisition announced 9 Mar 2026, subject to closing conditions; technology to be integrated into OpenAI Frontier | [S1][S2] |
| Promptfoo privacy notice | Updated 13 Mar 2026; names Promptfoo, Inc.; allows sharing with "affiliates or others in our corporate group" | [S10] |
| Promptfoo Community limits | 10,000 probes/month, reset on the 1st (UTC); new scans blocked at the limit | [S5] |
| Promptfoo runtime | Requires Node.js 22.22.0 or later (Node 24 LTS suggested); no official Docker image | [S11] |
| PyRIT | Repository moved to microsoft/PyRIT (Mar 2026); v1.0.0 on 24 Jul 2026 with breaking API changes; v1.1.0 current | [S16][S12] |
| garak | v0.17.0 (Sep 2026) adds EU AI Act mapping tags on probe results; we run 0.17.1 | [S17] |

---

## 3. Strengths

| Strength | Detail | Relevance to us |
|---|---|---|
| Tests whole applications, not just models | Targets apps, agents and RAG pipelines rather than only the model endpoint [S14] | High in principle; see Section 6 for what survives in local mode |
| CI-native | YAML configs, shareable web reports, integrates with GitHub Actions, GitLab, Jenkins [S18] | Medium: integrates with our CI |
| Compliance mapping | OWASP LLM Top 10 and OWASP Top 10 for Agentic Applications mappings [S22] | Medium: governance asks for coverage-to-taxonomy evidence |
| Remediation guidance | Reports map findings to prioritised fixes [S19] (confirm whether this is in Community or Enterprise only) | High: governance wants reports that say what to fix |
| Custom policy plugins | `policy` and `intent` plugins run on our own provider [S7] | Medium: organisation-specific rules can be expressed declaratively |
| Adoption and assurance | Vendor claims 156 of the Fortune 500 as users and shows SOC 2 and ISO 27001 badges [S21] | Low for the OSS path; relevant only to procurement |

---

## 4. Weaknesses and risks

| Risk | Detail | Severity for us |
|---|---|---|
| Vendor neutrality | Owned by a model provider once the deal closes. Some teams now want a red-teaming stack not owned by any model provider [S13]. Observers note that multi-provider support may face pressure as Frontier's commercial roadmap develops [S15] | High: we red-team applications that may use OpenAI models; an independence question for audit |
| Default data egress | Hosted generation and grading send target prompts and responses off-premises [S3] | High (see Section 5) |
| Local-mode coverage loss | Remote-only plugins and strategies produce no tests when remote generation is off [S3][S12] | High: removes the main differentiators |
| Local generation quality | The vendor itself warns that local generation quality "is generally low for most models" [S8] | Medium: we have our own attacker models, but this needs validation |
| Probe cap | 10,000 probes/month on Community, including runs on our own models [S4][S5] | High at CI or fleet scale |
| Disabling remote calls is not isolation | The disable flag "is not a network isolation guarantee"; it does not stop telemetry, account/licence checks, sharing or Cloud sync [S3] | Medium: needs an enforced egress-deny rule |
| Text-based grading | Documented grading inputs are the prompt, the response and grading criteria [S3]. Our agentic POC bases authorisation verdicts on trace evidence | Medium: Promptfoo verdicts on agent authorisation would be weaker than our trace-graded approach |
| Toolchain fit | Node.js runtime and npm dependency tree in a Python-standardised, hardened-image environment [S11] | Medium: extra image, supply-chain review |
| Breadth over depth | Its strength is breadth in one binary, at the cost of depth in any one area [S13] | Medium |

---

## 5. How data travels from the runner to Promptfoo and OpenAI

### 5.1 Network paths

All calls are HTTPS from the machine running the CLI (a developer laptop, a CI agent, or our cloud project) directly to internet endpoints.

```
 Our controlled environment (laptop / CI agent / air-gapped project)
 +---------------------------------------------------------------------+
 |  promptfoo CLI  ------------------------>  Target app or agent      |
 |       |                                    (behind guardrails)       |
 |       |  optional, in-network:                                       |
 |       |  attacker model, LLM grader                                  |
 +-------|-------------------------------------------------------------+
         |  HTTPS egress: blocked in the air-gapped project,
         |  open on laptops and other CI unless proxy-blocked
         v
  api.promptfoo.app ...... attack generation and grading (default mode)
  api.promptfoo.dev ...... consent tracking for harmful plugins; version check
  a.promptfoo.app / r.promptfoo.app ... telemetry (PostHog)
  Promptfoo Cloud ........ sharing, Cloud sync, hosted reports
  api.openai.com ......... generation and grading when OPENAI_API_KEY is set
  Codex / ChatGPT ........ generation and grading when Codex is signed in
         |
         v
  Promptfoo, Inc. (acquisition by OpenAI announced 9 Mar 2026).
  Privacy notice permits sharing within its "corporate group".
```

### 5.2 What is sent, by endpoint

| Endpoint | Operator | Data sent | When | Control |
|---|---|---|---|---|
| api.promptfoo.app (generation) | Promptfoo, Inc. | Application purpose (the `purpose` field), plugin and strategy configuration, attack context, user email [S3][S12] | Default, when no OpenAI key or Codex login is present; always for remote-only plugins | `PROMPTFOO_DISABLE_REMOTE_GENERATION=true` |
| api.promptfoo.app (grading) | Promptfoo, Inc. | The prompt sent to the target, the target's response, grading criteria; conversation history for multi-turn tests [S3][S12] | Default hosted grading | Own `redteam.provider` or grading provider |
| api.promptfoo.app (`redteam poison`) | Promptfoo, Inc. | Full document contents and the requested goal; always hosted, even with our own key [S3] | When `redteam poison` is used | Do not use |
| api.promptfoo.app (setup helpers) | Promptfoo, Inc. | Target URLs, request examples, auth headers entered in setup or test forms [S3] | UI setup and target-test features | Do not use; never enter real secrets |
| api.promptfoo.dev | Promptfoo, Inc. | Consent tracking for harmful plugins [S3]; version check unless disabled [S12] | Harmful plugins; CLI start-up | `PROMPTFOO_DISABLE_UPDATE=1` |
| a.promptfoo.app, r.promptfoo.app | Promptfoo, Inc. (PostHog) | Commands run, plugin, strategy and assertion types; may include package version, CI status, user ID, email, login status. Not prompt content, responses or test cases [S3] | Always, unless disabled. One acknowledgement event is still sent after opt-out, per a third-party source review [S12] | `PROMPTFOO_DISABLE_TELEMETRY=1` |
| Promptfoo Cloud | Promptfoo, Inc. | Eval inputs and outputs; shared results kept for two weeks [S10] | `share`, Cloud sync, hosted reports | `PROMPTFOO_DISABLE_SHARING=1`, `--no-share` |
| api.openai.com | OpenAI | Generation and grading traffic (same content as the grading row above). Default `redteam.provider` is `openai:gpt-5` [S8] | When `OPENAI_API_KEY` is set | Configure a non-OpenAI provider |
| Codex SDK (ChatGPT account) | OpenAI | Default text generation and grading through `openai:codex-sdk` [S3] | When Codex is installed and signed in, and no higher-priority credentials are set | Do not install Codex on runners |

### 5.3 Why this matters for a regulated organisation

- **The most sensitive artefacts are the ones sent for grading.** Target responses to attacks are the findings. When an attack succeeds, they can contain system prompts, tool outputs or customer data the bot was tricked into disclosing.
- **Terms of the hosted service** [S10]:
  - Data may be processed in "the United States or other locations".
  - No general retention period is stated.
  - De-identified information may be used "for any purpose".
  - Unnamed vendors covering "artificial intelligence platforms" are used.
  - Sharing within the corporate group is permitted.
- **Shadow-usage risk.** Any engineer can run `npx promptfoo redteam` on a laptop with default settings and send target data out. This is independent of any adoption decision.
- **Our air-gapped environment blocks these calls, which protects the data, but the failure is silent.** Remote-only plugins return zero test cases with only an error logged [S12]. The resulting report can look complete when it is not.

### 5.4 Operating modes compared

| Mode | Generation | Grading | Coverage | Meets our constraint |
|---|---|---|---|---|
| A. Default (no key) | Promptfoo API | Promptfoo API | Full | No |
| B. `OPENAI_API_KEY` set | OpenAI; remote-only plugins still go to Promptfoo API [S12] | OpenAI | Full | No |
| C. Own `redteam.provider`, no disable flag | Promptfoo API for remote-only plugins and harm categories [S12] | In-network | Full | No |
| D. Own providers + all offline flags + egress-deny | In-network | In-network | Reduced (Section 6) | Yes, if egress-deny is enforced |
| E. Enterprise On-Prem | In-network runner and self-hosted inference for all plugins [S3] | In-network | Full | Yes, subject to contract and TPRM |

---

## 6. Coverage in fully local mode (Mode D)

This is the deciding technical point.

| Area | Available locally | Remote-only (unavailable in Mode D) |
|---|---|---|
| Access control and agent security | `rbac`, `pii:*`, `prompt-extraction`, `sql-injection`, `shell-injection`, `debug-access`, `tool-discovery`, `cross-session-leak`, `excessive-agency` | `bola`, `bfla`, `ssrf`, `indirect-prompt-injection`, `mcp`, `agentic:memory-poisoning`, `hijacking`, `data-exfil`, `system-prompt-override`, `ascii-smuggling` |
| RAG | None of the RAG-specific plugins | `rag-poisoning`, `rag-document-exfiltration`, `rag-source-attribution` |
| Financial services | None | All 12 `financial:*` plugins (for example `sox-compliance`, `data-leakage`, `confidential-disclosure`, `misconduct`) |
| Harmful content and bias | Public datasets (HarmBench, DoNotAnswer, CyberSecEval, XSTest and others; mirroring needed) | `harmful:*` categories (22 per [S12]), `bias:*` |
| Custom | `policy`, `intent` (run on our own provider) | None |
| Strategies | `basic`, plus multi-turn strategies such as crescendo and iterative jailbreak running on our own attacker model [S12] | `jailbreak:meta` and `jailbreak:composite` (two of the three defaults [S8]), `goat`, `hydra`, `likert`, `goblin`, `gcg`, `citation`, `audio` [S3] |

Sources: [S3][S7][S12]. Plugin status as listed on 12 Nov 2026; re-verify against the version we pin.

**Implication.** In Mode D, what Promptfoo adds over garak + PyRIT is:

- a CI harness and declarative configuration
- a web report
- a handful of application-level plugins (`rbac`, `pii`, SQL and shell injection, `excessive-agency`, `tool-discovery`, `cross-session-leak`)

All of these can be built as PyRIT scenarios. The capabilities a financial institution would most want (financial-services packs, BOLA/BFLA, MCP, RAG, indirect injection) need either hosted inference or Enterprise On-Prem.

---

## 7. Financial impact

### 7.1 Licence options

| Tier | Price | What it includes | Fit |
|---|---|---|---|
| Community (MIT) | USD 0 | All evaluation features, local/self-hosted use, red teaming up to 10,000 probes/month [S6] | Usable only in Mode D, with reduced coverage |
| Enterprise (cloud) | Custom, not published [S6] | Custom red-team limits, team sharing, continuous monitoring, compliance dashboard [S6] | Hosted; does not meet our constraint |
| On-Premise | Custom, not published [S6] | Deployment on our infrastructure, complete data isolation, dedicated deployment engineer [S6] | Only full-coverage option inside our constraint |

- **Category reference only, not Promptfoo pricing:** third-party price trackers put commercial AI security platforms in the tens of thousands of USD per year. For example, Prompt Security is typically USD 20K–80K per year, and HiddenLayer's tiers top out around USD 150K [S20].

### 7.2 What the Community probe cap means in practice

A probe is one request to the target. Each turn of a multi-turn attack counts separately [S5]. The default is 5 tests per plugin, and strategies apply across plugins [S8].

**Illustrative assumptions (to be replaced with POC measurements):**

- 20 locally available plugins × 5 tests = 100 base test cases per app
- `basic` = 100 probes
- one single-turn transform strategy = 100 probes
- one multi-turn strategy at about 5 turns = 500 probes
- one iterative jailbreak at about 4 attempts = 400 probes
- **Total: about 1,100 probes per full app scan**

| Scenario | Scans/month | Probes/month | Multiple of the 10,000 cap |
|---|---|---|---|
| Ad-hoc assessments | 9 | ~9,900 | ~1.0x |
| CI gate: 10 apps, weekly | 40 | ~44,000 | ~4.4x |
| Fleet: 30 apps or agents, weekly | 120 | ~132,000 | ~13.2x |

At CI or fleet scale, the Community tier is not viable. A paid tier becomes necessary, which is how the pricing is designed to work [S19]. The documentation does not say whether the counter is enforced locally or server-side, or what happens with zero egress [S5]. This must be checked in the POC and treated as a licensing question for Legal. Bypassing the cap must not be part of any plan.

### 7.3 Inference cost (the same for every option)

Every framework needs an attacker model and a judge. Our attacker models run on GPUs we have already provisioned. The judge runs on a hosted LLM through our internal gateway.

**Illustrative judge cost at the CI scenario (44,000 probes/month):**

- About 1,500 input and 300 output tokens per graded probe, giving 66M input and 13.2M output tokens per month.
- At a placeholder rate of USD 3 per million input tokens and USD 15 per million output tokens, that is about **USD 400/month**.
- Replace the placeholder rate with the internal chargeback rate.
- This is similar for PyRIT's LLM scorers. garak's detectors are mostly non-LLM and cheaper.

**Conclusion:** inference does not separate the options. Hosted inference in Modes A–C is free within the cap only because our data goes to the vendor.

### 7.4 One-time and ongoing effort (planning estimates, to be validated)

| Work item | promptfoo Mode D | promptfoo On-Prem |
|---|---|---|
| Hardened Node.js 24 image, internal npm mirror, pinned version | 1–2 pw | 0.5–1 pw (vendor runner) |
| npm supply-chain review, egress-deny verification | 1–2 pw | 1 pw |
| Providers: judge, attacker model, target adapters behind guardrails | 1–2 pw | 1–2 pw |
| CI integration and report publishing | 1 pw | 0.5–1 pw |
| Taxonomy mapping and judge–human (κ) calibration for Promptfoo graders | 2–3 pw | 2–3 pw |
| **Total one-time** | **6–10 pw** | **5–8 pw** |
| Ongoing maintenance (fast release cadence, re-validation) | +0.1–0.2 FTE | +0.1 FTE plus vendor management |
| Third-party risk management (TPRM) and procurement | Light (OSS review) | Full TPRM, outsourcing and cross-border review, contract with an OpenAI-owned entity, exit plan |

pw = person-weeks. Apply the internal rate to convert to cost.

---

## 8. Cost–benefit against the existing garak + PyRIT stack

### 8.1 Capability comparison

| Capability | garak 0.17.1 | PyRIT 1.x | promptfoo Mode D | promptfoo On-Prem |
|---|---|---|---|---|
| Known-attack breadth at the model layer | Strong (189 probes in 0.15.1 [S17]) | Moderate (datasets) | Moderate (datasets, local plugins) | Strong |
| Multi-turn adaptive attacks | Limited | Strong (Crescendo and others; custom attacks) | Partial (crescendo, iterative on our model) | Strong (adds GOAT, Hydra) |
| Agent authorisation (BOLA/BFLA), MCP, memory poisoning | No | Custom-built; trace-graded in our POC | No (remote-only) | Yes (text-graded) |
| RAG poisoning and exfiltration | Limited | Custom-built | No (remote-only) | Yes |
| Financial-services test packs | No | Custom-built | No (remote-only) | Yes (12 plugins) |
| CI regression gating | Custom workflow | Custom code | Strong | Strong |
| Taxonomy mapping in reports | EU AI Act tags (0.17) [S17] | Custom | OWASP LLM and Agentic | OWASP LLM and Agentic, dashboards |
| "What to fix" remediation output | No | Custom | Yes, to be confirmed for Community [S19] | Yes |
| Multimodal attacks | Limited | Strong (text, image, audio, video [S13]) | Limited | Limited (audio strategy) |
| Data sovereignty | No telemetry; Hugging Face downloads need mirroring [S12] | Sends only where pointed; remote seed datasets need mirroring [S12] | Needs five offline flags plus egress-deny [S9] | Designed for no external transmission [S3]; verify |
| Licence | USD 0, Apache 2.0 | USD 0, MIT | USD 0, MIT, 10K-probe cap | Custom |
| Stack fit | Python | Python | Node.js 22.22+ | Node.js plus vendor runner |
| Sponsor | NVIDIA | Microsoft | OpenAI (once the deal closes) | OpenAI (once the deal closes) |

**On neutrality.** All three tools are backed by vendors. What sets Promptfoo apart is where data goes in its default and hosted modes, not who sponsors the code.

### 8.2 Three-year view

| Cost element | Current path: garak + PyRIT | Add promptfoo Mode D | Add promptfoo On-Prem |
|---|---|---|---|
| Licence | 0 | 0 (capped) | 3 × annual quote |
| PyRIT 0.x to 1.x migration | ~2–3 pw (needed anyway) | Same | Same |
| Build missing coverage in PyRIT: financial packs, RAG, CI wrapper, reporting | ~9–12 pw | ~7–10 pw (still needed, because the packs are remote-only) | ~0–2 pw |
| Promptfoo integration | 0 | 6–10 pw | 5–8 pw |
| Ongoing maintenance | Baseline | +0.1–0.2 FTE | +0.1 FTE plus vendor management |
| Residual data-sovereignty risk | Low | Low if egress-deny is enforced | Low, to be verified contractually |

**Reading the table**

- **Mode D** adds 6–10 pw of integration plus a second toolchain, while saving only about 2 pw of PyRIT build. Its incremental benefit is small and the cost is net positive. **Verdict: not justified** unless a CI gate with Promptfoo's reporting is a near-term requirement.
- **On-Prem** saves about 7–10 pw of build effort and gives coverage sooner, at the cost of three years of licence plus 5–8 pw of integration and full TPRM. **Verdict: justified only if** three years of licence costs less than the build effort saved plus the value of having financial-services coverage early, and TPRM accepts an OpenAI-owned vendor in the red-teaming chain.

---

## 9. Recommendation

1. **Strategic stack:** keep garak as the baseline scanner and PyRIT for application-specific, multi-turn and agentic campaigns. This matches independent guidance for regulated teams that cannot send prompts or outputs out [S12]. Proceed with the PyRIT budget.
2. **Promptfoo hosted and default modes (A–C):** not permitted. Block `*.promptfoo.app` and `api.promptfoo.dev` at the corporate proxy to close the shadow-usage path, and publish guidance to engineering teams.
3. **Promptfoo Mode D:** optional, time-boxed POC (Section 10), and only if a CI regression gate is a near-term requirement.
4. **Promptfoo Enterprise On-Prem:** do not procure unless the quote passes the Section 8.2 break-even and TPRM clears the ownership question.
5. **Revisit when any of these happen:**
   - the acquisition closes and Frontier packaging is announced
   - Community licence terms or the probe cap change
   - the On-Prem quote changes materially

---

## 10. Proof-of-concept plan (two weeks, Mode D)

| Step | Activity | Pass criterion |
|---|---|---|
| 1 | Build a hardened Node.js 24 image with a pinned Promptfoo version from the internal mirror. Set `PROMPTFOO_DISABLE_TELEMETRY=1`, `PROMPTFOO_DISABLE_UPDATE=1`, `PROMPTFOO_DISABLE_REMOTE_GENERATION=true`, `PROMPTFOO_DISABLE_SHARING=1`, `PROMPTFOO_SELF_HOSTED=1` [S9] | Image passes security scan |
| 2 | Run behind a default-deny egress rule; log every blocked destination for one full run [S12] | Zero unexpected egress attempts, or each one explained |
| 3 | Run `promptfoo redteam usage` under zero egress | Cap behaviour understood and documented for Legal |
| 4 | Scan a pilot chatbot with the locally available plugins; run garak and PyRIT against the same target | At least N validated findings not found by garak or PyRIT (N agreed in advance) |
| 5 | Check that the report lists every planned plugin and generated test counts | No silent zero-test plugins |
| 6 | Review report quality with governance | Remediation output maps findings to guardrail or application fixes |
| 7 | Record effort, probes per scan and judge tokens | Actuals replace the estimates in Section 7 |

**Exit:** if Step 4 or Step 6 fails, stop and close the evaluation.

---

## 11. Open questions for the vendor and procurement

1. Has the acquisition closed? Which legal entity would contract with us, and under which data processing terms?
2. Will On-Prem keep supporting non-OpenAI providers and graders on a contractual basis?
3. Are remediation reports included in Community, or only in Enterprise?
4. How is the Community probe cap enforced in a zero-egress environment?
5. What is the On-Prem price, the probe allowance, the support SLA and the exit and data-return terms?
6. What is the roadmap for Promptfoo inside OpenAI Frontier, and what happens to standalone On-Prem customers?

---

## Appendix A. Sources

| Ref | Source |
|---|---|
| S1 | OpenAI, "OpenAI to acquire Promptfoo" (9 Mar 2026): https://openai.com/index/openai-to-acquire-promptfoo/ |
| S2 | Promptfoo, "Promptfoo is joining OpenAI": https://promptfoo.dev/blog/promptfoo-joining-openai |
| S3 | Promptfoo docs, Red team data handling: https://www.promptfoo.dev/docs/red-team/troubleshooting/data-handling/ |
| S4 | Promptfoo docs, Configuring inference: https://www.promptfoo.dev/docs/red-team/troubleshooting/inference-limit/ |
| S5 | Promptfoo docs, Open source limits: https://www.promptfoo.dev/docs/red-team/open-source-limits |
| S6 | Promptfoo pricing: https://www.promptfoo.dev/pricing |
| S7 | Promptfoo docs, Red team plugins: https://www.promptfoo.dev/docs/red-team/plugins/ |
| S8 | Promptfoo docs, Red team configuration: https://www.promptfoo.dev/docs/red-team/configuration/ |
| S9 | Promptfoo FAQ (offline environment variables): https://www.promptfoo.dev/docs/faq/ |
| S10 | Promptfoo privacy notice (updated 13 Mar 2026): https://www.promptfoo.dev/privacy |
| S11 | Promptfoo docs, Installation: https://www.promptfoo.dev/docs/installation/ |
| S12 | Particula Tech, Promptfoo self-hosted red teaming and alternatives (22 Oct 2026; source and docs review at promptfoo 0.124.1, garak 0.17.1, PyRIT 1.1.0): https://particula.tech/blog/promptfoo-self-hosted-red-teaming-alternatives |
| S13 | AppSec Santa, Promptfoo alternatives: https://appsecsanta.com/ai-security-tools/promptfoo-alternatives |
| S14 | AppSec Santa, Garak vs Promptfoo: https://appsecsanta.com/ai-security-tools/garak-vs-promptfoo |
| S15 | Futurum Group, OpenAI acquires Promptfoo: https://futurumgroup.com/insights/openai-acquires-promptfoo-gaining-25-foothold-in-fortune-500-enterprises/ |
| S16 | StationX, PyRIT tutorial (v1.0.0 release, canonical repository): https://app.stationx.net/articles/pyrit-microsoft |
| S17 | garak 0.17.0 install notes (EU AI Act tags): https://www.qwe.edu.pl/ai-tools/garak-0-17-0-llm-security-scanner-install/ ; StationX garak review (probe counts, licence): https://app.stationx.net/articles/garak-llm-scanner |
| S18 | Hakky, garak / PyRIT / promptfoo comparison: https://book.st-hakky.com/security/llm-red-teaming-tools-comparison |
| S19 | Sacra, Promptfoo company profile: https://sacra.com/c/promptfoo/ |
| S20 | CostBench, Prompt Security pricing (category reference): https://www.costbench.com/software/ai-security/prompt-security/ |
| S21 | Promptfoo homepage: https://www.promptfoo.dev/ |
| S22 | Promptfoo release notes (OWASP Agentic Top 10 mapping, Dec 2025): https://www.promptfoo.dev/docs/releases |

Third-party sources (S12–S20) are used for corroboration and estimates. Vendor documentation (S3–S11) takes precedence where they differ. All plugin and pricing details were read on 12 November 2026 and should be re-verified against the version and quote in hand.
