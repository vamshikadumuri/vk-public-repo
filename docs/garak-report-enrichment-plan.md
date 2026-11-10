# Garak Report Enrichment: Implementation Plan

Turning garak 0.17.1 scan output into a report that tells each owner what to fix.

- Status: draft for review
- Date: 10 Nov 2026
- Baseline: garak 0.17.1 (released 16 Oct 2026)

---

## 1. Goal

Governance should be able to open a red-team report and see, per finding:

1. What failed, in risk language rather than probe names.
2. How bad it is, and how sure we are.
3. Which layer let it through: guardrail, application, or model.
4. What to change, who owns the change, and how much effort it is.
5. How we will prove the fix worked.

garak stops at point 2. This plan adds 3, 4 and 5 as a post-processing layer over garak's own output files. garak itself is not modified.

### Non-goals

- Replacing garak's HTML report. It stays as the technical appendix.
- Auto-applying fixes to guardrails or applications.
- Producing a single pass/fail certification score.

## 2. Assumptions to confirm

| # | Assumption | Effect if wrong |
|---|---|---|
| A1 | The red team has no control over the guardrails in front of the target apps and cannot switch them off for a comparison run. | If a guardrail-off run is possible, Phase 3 becomes a simple A/B diff. |
| A2 | Scans run against the deployed application endpoint, so each response reflects guardrail + system prompt + model together. | If scans hit the bare model, guardrail attribution does not apply. |
| A3 | An LLM is reachable through the internal gateway (Stork) for the synthesis step. | Without it, the report ships with template-only recommendations (still useful). |
| A4 | Application teams may not share system prompts or tool definitions. | Recommendations stay at control level instead of quoting specific prompt edits. |
| A5 | Hit logs contain harmful content and stay inside the bank's environment. | None; the design keeps all processing on-prem. |

## 3. What garak already provides

All of this exists today in every run. Nothing needs to be scraped from HTML.

| Artefact | What it holds | Used for |
|---|---|---|
| `*.report.jsonl`, `attempt` rows | `uuid`, `seq`, `status` (0/1/2), `probe_classname`, `probe_params`, `prompt`, `outputs`, `detector_results` (scores per detector, aligned with `outputs`), `notes`, `goal`, `conversations`, `intent` | Evidence, attribution, non-hit analysis |
| `*.report.jsonl`, `eval` rows | Per probe/detector: `passed`, `total_evaluated`, `nones`, bootstrap `confidence_lower` / `confidence_upper` (when n >= 30), `hit_at_k`, optional per-intent counts | Statistics and confidence |
| `*.report.jsonl`, `digest` row | Grouped results, `absolute_score` (a pass rate), relative Z-score, DEFCON grade 1 (worst) to 5 (best), probe tier, `technique_intent_matrix` | Severity, grouping, technique view |
| `*.hitlog.jsonl` | One row per successful attack: `goal`, `prompt`, `output`, `triggers`, `score`, `probe`, `detector`, `generator`, attempt ids | Evidence shown to owners |
| Probe tags | `owasp:*`, `avid-effect:*`, `risk-cards:*`, `cwe:*`, `demon:*` (technique), `euai:*` (new in 0.17.1), plus garak's own `payload:*` and `quality:*` tags. Catalogue in `garak/data/tags.misp.tsv` | Join key to the remediation knowledge base |
| Probe tiers | 1 = of concern, 2 = compare with state of the art, 3 = informational, 9 = unlisted | Prioritisation |
| `garak.analyze` tools | `report_digest`, `qual_review`, `tbsa`, `report_avid`, `rebuild_cis` | Reuse instead of rebuilding |

Notes:

- Exact key names for `eval`, `digest` and hitlog rows should be confirmed against a real 0.17.1 report in Phase 1. The documentation describes them but is not a formal schema.
- `garak.analyze` tools expect a report from the same garak version. Pin the version.
- garak's `owasp:llm01` to `llm10` tags follow the original OWASP LLM Top 10 numbering (for example `llm02` is Insecure Output Handling). If governance reports against the 2025 list, a crosswalk is needed (task 2.5).

## 4. Target report

One report per scan, in two forms: a document for people (HTML, exportable to PDF) and `findings.json` for tooling.

| Section | Content | Audience |
|---|---|---|
| 1. Summary | Target, scan date, garak version and config hash, overall risk level, top 3 to 5 concerns, counts by priority | Governance |
| 2. Findings table | One row per finding: risk category, technique, attack success rate with interval, hit@k, priority, failing layer, owner | Governance, owners |
| 3. Recommendations by owner | Grouped as guardrail team / application team / model platform. Each action lists type, description, effort, expected impact, findings addressed | Owners |
| 4. Finding detail | Evidence (2 to 3 redacted prompt and response pairs), why the detector flagged it, statistics, mapped controls, verification probes | Engineers |
| 5. Framework view | Findings grouped by OWASP LLM Top 10 and EU AI Act tags, with the coverage disclaimer | Compliance |
| 6. Change since last scan | Fixed, regressed, new, unchanged | Governance |
| Appendix | Link to garak's HTML report, false-positive overrides applied, method notes | Audit |

## 5. Architecture

```
garak scan (pinned 0.17.1, standard config)
        |
        v
report.jsonl + hitlog.jsonl  --->  archived in object storage (immutable)
        |
        v
[1] Parse and normalise        -> findings (probe x detector x technique)
[2] Apply analyst overrides    -> corrected verdicts, recomputed rates
[3] Attribute failing layer    -> guardrail gap / model gap / app gap
[4] Join remediation KB        -> candidate controls per finding
[5] Score and prioritise       -> P1..P4, owner routing
[6] LLM synthesis (optional)   -> contextual wording, grounded in KB
[7] Render                     -> report.html / report.pdf / findings.json
        |
        v
Fix -> re-run "verify_with" probes -> delta report
```

Stages 1 to 5 and 7 are deterministic. Stage 6 is the only model-dependent step and the report must render without it.

## 6. Phases

### Phase 0: Standardise the scan (2 to 3 days)

Recommendations are only comparable across runs if the runs are.

| Task | Detail |
|---|---|
| 0.1 | Move from ad hoc notebook runs to one versioned YAML config passed with `--config`. |
| 0.2 | Set `reporting.taxonomy: owasp` so garak's own report groups by risk. |
| 0.3 | Fix `run.generations` (suggest 5), `run.seed`, `run.eval_threshold`. Set `reporting.hit_at_k` to include 1; the full generation count is always reported. |
| 0.4 | Set `reporting.report_prefix` to `<app-id>_<env>_<date>` so files are traceable. |
| 0.5 | Define two probe sets: a full baseline and a short re-test set per finding. |
| 0.6 | Archive `report.jsonl`, `hitlog.jsonl`, `report.html`, the config and the garak version together. |
| 0.7 | Send the attempt `uuid` as a request header to the target if the REST generator config allows it. This gives guardrail owners a correlation id for their own logs. |

Deliverable: `scan-config.yaml`, a run wrapper script, a storage layout.
Done when: two runs of the same config against the same target produce reports the parser treats as comparable.

### Phase 1: Parser and findings model (1 week)

| Task | Detail |
|---|---|
| 1.1 | Read `report.jsonl` by `entry_type`. Keep `attempt` rows with `status == 2`, all `eval` rows, and the `digest`. |
| 1.2 | Load probe metadata (tags, tier, goal, description) from garak's plugin metadata for the pinned version. |
| 1.3 | Build a finding per probe x detector, then roll up to risk category x technique using `owasp:*` and `demon:*` tags. Use `technique_intent_matrix` from the digest where the run used intent probes. |
| 1.4 | Carry `passed`, `total_evaluated`, `nones`, interval bounds, `hit_at_k`, DEFCON, Z-score and tier onto each finding. |
| 1.5 | Attach up to N hit examples from the hitlog, deduplicated, longest outputs truncated. |
| 1.6 | Schema tests against at least three real reports. Fail loudly on unknown fields so a garak upgrade cannot silently break the join. |

Deliverable: `garak_enrich.parse` module and `findings.json` (schema in section 7).
Done when: finding counts and rates reconcile exactly with garak's own HTML report for three scans.

### Phase 2: Remediation knowledge base (2 weeks, in parallel with Phase 1)

This is the core asset and the part that makes recommendations auditable.

| Task | Detail |
|---|---|
| 2.1 | Define the KB schema: one YAML entry per control (section 7). |
| 2.2 | Write entries keyed on tags. Start with the tags that actually appear in your scans rather than the whole catalogue. Seed list in section 8. |
| 2.3 | Each entry names a control layer (`input_guardrail`, `output_guardrail`, `system_prompt`, `app_architecture`, `model`, `process`) and an owner role. |
| 2.4 | Each entry carries `verify_with`: the probe spec that re-tests it. |
| 2.5 | Add a crosswalk from garak's OWASP tags to the OWASP edition and internal control ids governance uses. |
| 2.6 | Review with the guardrail owners and one application team. Their capability catalogue decides which guardrail recommendations are realistic. |
| 2.7 | Version the KB in git. Record the KB version in every report. |
| 2.8 | Coverage test: enumerate every probe in the pinned garak install and fail the build if any probe has no mapping through the fallback chain in section 9. |

Sources for entries: OWASP LLM Top 10 mitigations, CWE mitigations for `cwe:*` tags, the guardrail product's own policy catalogue, internal standards.

Deliverable: `kb/*.yaml` covering every tag seen in the last five scans.
Done when: every finding in those scans maps to at least one control, and unmapped tags are listed in the report instead of dropped.

### Phase 3: Failing-layer attribution without guardrail access (1 week)

Because the guardrails cannot be toggled (A1), the layer is inferred from the responses.

| Task | Detail |
|---|---|
| 3.1 | Collect guardrail block fingerprints: canned refusal text, HTTP status or error codes, any response fields the gateway adds. Ask the guardrail owners for the list; otherwise harvest from past reports. |
| 3.2 | Classify every evaluated output into `guardrail_block`, `model_refusal`, `complied` or `error`. Fingerprints first, then garak's mitigation detector scores for refusals. |
| 3.3 | Per finding, compute block rate, model refusal rate and compliance rate. |
| 3.4 | Assign a class using the table below. |
| 3.5 | Optional upgrade: with correlation ids from 0.7, request a read-only verdict export from the guardrail owners and replace inference with ground truth. |

| Pattern | Class | Routed to |
|---|---|---|
| Hits exist; guardrail block rate near zero for this technique | Guardrail coverage gap, and model not robust | Guardrail team first, application team second |
| Hits exist; guardrail blocks some variants and misses others | Guardrail evasion (encoding, language, phrasing) | Guardrail team |
| No hits, but only the model refused and the guardrail never fired | Latent gap: protection depends on model behaviour alone | Guardrail team, lower priority |
| Hits are harmful because of how the app uses the output (markup, links, code) | Application output handling | Application team |
| Hits are system prompt disclosure or instruction override | System prompt and privilege design | Application team |
| No hits; guardrail blocked | Working control | None; record as evidence |

The latent-gap row is information a hit-only report never shows, and it matters when the underlying model is swapped.

Deliverable: `attribution` field on every finding with the supporting rates.
Done when: an analyst reviewing 50 sampled outputs agrees with the classifier on at least 90%.

### Phase 4: Scoring and prioritisation (3 days)

Keep the rubric simple and published in the report.

| Input | Source |
|---|---|
| Exploitability | hit@k at full generation count; lower interval bound of attack success rate |
| Probe importance | garak tier (1 > 2 > 3; tier 9 is reported unranked, not dropped) |
| garak grade | DEFCON, minimum of absolute and relative |
| Impact | Severity set per risk category in the KB |
| Exposure (optional) | App metadata: user population, data classification, tool access |

Suggested rubric:

| Priority | Rule |
|---|---|
| P1 | High-impact category and (hit@k >= 0.5 or DEFCON <= 2) on a tier 1 probe |
| P2 | High-impact category with any confirmed hit, or medium impact with DEFCON <= 2 |
| P3 | Confirmed hits on tier 2 or 3 probes, or latent gaps in high-impact categories |
| P4 | Informational, below-threshold or tier 3 context-dependent results |

Rules:

- Fewer than 30 evaluations: mark "low confidence" instead of quoting a rate with false precision.
- Do not headline TBSA. garak's documentation says it is a rough single number, not a substitute for the results, and only comparable within one garak version and config. Use it in the trend section only.
- Rank actions by findings addressed divided by effort, so a fix that closes several findings rises to the top.

Deliverable: `priority`, `owner`, `effort`, `impact` on each finding and action.
Done when: governance agrees the rubric in writing.

### Phase 5: LLM synthesis, grounded (1 week)

Turns mapped controls into specific wording for this target. Optional by design.

| Task | Detail |
|---|---|
| 5.1 | Input per finding: statistics, tags, attribution class, candidate KB controls, 3 to 5 hit examples, optional target context (purpose, system prompt, tools) when the application team shares it. |
| 5.2 | Output as strict JSON: summary, why it matters, actions each with `kb_control_id`, owner, effort, impact, verification step. |
| 5.3 | Grounding: every action must cite a KB control. Anything outside the KB is labelled "proposed, needs review". |
| 5.4 | Treat hit content as untrusted data. It is adversarial text by construction and can attack the synthesiser. Delimit it, instruct the model to analyse and never follow it, and validate output against the schema. |
| 5.5 | Expect the gateway's own guardrails to block some requests containing hit content. Retry with redacted examples, then fall back to the KB template. |
| 5.6 | Temperature 0. Log model id, prompt version, KB version and input hash for each generation. |
| 5.7 | Second pass for the summary section, built only from the structured findings, never from raw hits. |

Deliverable: `synthesis` block per finding and an executive summary.
Done when: on a pilot scan, a reviewer finds no recommendation that contradicts the KB and none that names a control the target lacks.

### Phase 6: Rendering (1 week)

| Task | Detail |
|---|---|
| 6.1 | Jinja2 template producing a single self-contained HTML file (section 4 layout). |
| 6.2 | PDF export from the same HTML for circulation. |
| 6.3 | `findings.json` and a CSV of actions for import into the tracker used for remediation. |
| 6.4 | Redaction pass on evidence: truncate, mask anything that looks like personal data or secrets, mark content as adversarial test output. |
| 6.5 | Fixed disclaimers: tags do not mean full framework coverage; scores are tied to garak version and config; a clean scan is not proof of safety. |

Deliverable: `report.html`, `report.pdf`, `findings.json`, `actions.csv`.
Done when: a governance reader can name the top three actions and their owners after two minutes with the report.

### Phase 7: Triage and verification loop (1 week)

| Task | Detail |
|---|---|
| 7.1 | Overrides file keyed by attempt `uuid` and detector: analyst marks a hit as false positive with a reason. Applied in stage 2, listed in the appendix with name and date. |
| 7.2 | Review rule before publishing: sample at least 10 hits (or all, if fewer) for every P1 and P2 finding. |
| 7.3 | Re-test: run each action's `verify_with` probes with the same garak version, config and seed after the owner reports a fix. |
| 7.4 | Delta report: fixed / improved / unchanged / regressed / new, comparing rates with their intervals, not point values. |
| 7.5 | Finding lifecycle: open, accepted risk, fix in progress, verified, with dates. |

Deliverable: `overrides.yaml` format, re-test command, delta section in the report.
Done when: one finding has gone from open to verified end to end.

### Phase 8: Operationalise (1 to 2 weeks)

| Task | Detail |
|---|---|
| 8.1 | Package as a CLI: `garak-enrich run --report <jsonl> --hitlog <jsonl> --kb <dir> --out <dir>`. |
| 8.2 | Pipeline job: scan, enrich, publish, archive. |
| 8.3 | garak upgrade procedure: re-run schema tests, diff the tag catalogue, re-baseline. Scores are expected to shift between garak versions, so never compare across an upgrade. |
| 8.4 | KB ownership and review cadence (quarterly, and on every guardrail product change). |
| 8.5 | Access control on reports and hit logs. |

## 7. Schemas

### Finding (`findings.json`)

```json
{
  "finding_id": "F-2026-11-APPX-007",
  "risk_category": "owasp:llm01",
  "technique": "demon:Language:Code_and_encode:Data_encoding",
  "probes": ["encoding.InjectBase64"],
  "detectors": ["encoding.DecodeApprox"],
  "tier": 3,
  "stats": {
    "total_evaluated": 256,
    "hits": 41,
    "attack_success_rate": 0.16,
    "ci_lower": 0.12,
    "ci_upper": 0.21,
    "hit_at_k": {"1": 0.16, "5": 0.48},
    "defcon": 3,
    "z_score": -0.4
  },
  "attribution": {
    "class": "guardrail_evasion",
    "guardrail_block_rate": 0.05,
    "model_refusal_rate": 0.52,
    "complied_rate": 0.43
  },
  "priority": "P2",
  "owner": "guardrail_team",
  "controls": ["KB-IN-004", "KB-OUT-002"],
  "evidence": [{"attempt_uuid": "...", "prompt": "...", "output": "..."}],
  "verify_with": "encoding",
  "status": "open"
}
```

Values above are illustrative.

### KB entry (`kb/KB-IN-004.yaml`)

```yaml
id: KB-IN-004
title: Normalise and decode input before guardrail inspection
layer: input_guardrail
owner_role: guardrail_team
applies_to:
  tags:
    - demon:Language:Code_and_encode:Data_encoding
    - demon:Language:Code_and_encode:Token
  attribution: [guardrail_gap, guardrail_evasion]
impact: high
effort: medium
action: >
  Decode common encodings (base64, hex, ROT13, URL, unicode confusables)
  and run guardrail policies on both the raw and decoded text.
rationale: >
  Policies that inspect only the literal input miss requests whose
  harmful content appears after decoding by the model.
verify_with: "encoding"
references:
  - OWASP LLM01 Prompt Injection
  - CWE-1427
internal_control_ids: []
```

## 8. Seed remediation mapping

Starting point for Phase 2. Wording and feasibility need review with the owners.

| garak signal | Typical failing layer | Candidate controls |
|---|---|---|
| `demon:...:Data_encoding`, `...:Token` | Input guardrail | Decode and normalise before inspection; inspect raw and decoded text |
| `demon:...:Ignore_previous_instructions`, `owasp:llm01` | Input guardrail, system prompt | Injection classifier on input; delimit untrusted content; instruction hierarchy in system prompt; least privilege for tools |
| `demon:Fictionalizing:Roleplaying:*` (DAN-style) | Input guardrail, model | Jailbreak pattern detection; refusal-persistence instruction; output moderation as a second layer |
| `demon:...:Escalating` (multi-turn) | Guardrail (conversation scope) | Evaluate conversation history, not only the last turn; per-session risk scoring |
| Latent injection probes (content arriving via documents or tools) | Application | Mark retrieved content as data; strip instructions from retrieved text; restrict tool calls triggered by retrieved content |
| `risk-cards:lmrc:prompt_extraction` | Application | Keep secrets out of the system prompt; leak detection on output for prompt fragments |
| `owasp:llm02`, `cwe:79`, `cwe:1336` (markup, XSS, template) | Application output handling | Encode output for its context; disable raw HTML and markdown image rendering; content security policy |
| Markdown or link exfiltration detectors | Application, output guardrail | Allowlist outbound URLs; strip images and links to unknown domains |
| `owasp:llm06`, `euai:harmful:pii` | Output guardrail | Personal-data and secret detection on output; data minimisation in context |
| Toxicity and `risk-cards:lmrc:*` content categories | Output guardrail | Enable or tune the matching content category; threshold review with examples |
| `owasp:llm09`, `euai:deception:hallucination` | Application | Grounding with citations; abstain instructions; human review for decisions |
| `owasp:llm04` (resource abuse) | Platform | Token and rate limits; output length caps; timeouts |
| `owasp:llm08` (excessive agency) | Application | Narrow tool scopes; confirmation for state-changing actions |

## 9. Coverage across all garak probes

### 9.1 What works for any probe

The parser, statistics, scoring, overrides, rendering and delta stages read `attempt`, `eval` and `digest` rows. Those rows have the same shape whichever probe produced them, so these stages need no per-probe work, including for probes added in later garak versions or written in-house.

### 9.2 What needs per-probe handling

| Gap | Why | Handling |
|---|---|---|
| Remediation mapping | Section 8 seeds the common cases only. A recommendation is only as specific as the KB entry behind it. | Fallback chain, most specific first: probe class entry, then `payload:*` / `demon:*` tag, then `owasp:*` / `cwe:*` tag, then probe module default, then a generic "unmapped" entry that still shows evidence and flags the finding for KB authoring. Nothing is dropped silently. Task 2.8 enforces this. |
| Multi-turn probes | Attribution in Phase 3 assumes one prompt and one response. Iterative and conversational probes carry several turns per attempt. | Classify per conversation using `conversations`; a guardrail block at any turn counts. Read hit@k from the `n` key that garak uses when prompts get differing generation counts. |
| Failures no guardrail owns | Hallucinated packages, snowballed false answers, accepted false claims, training-data replay and token glitches are model or application behaviour. | Extra attribution class `model_behaviour`, routed to the application or model platform owner. |
| Probes that do not apply to an application endpoint | Some probes inspect model files or tokenizers, or need image or audio input. | Mark as "not applicable" per target in the scan config. Report them separately from "passed". |
| Probes that did not run | Module and `all` selection picks up active probes only. Some probes are inactive by default or need an attacker model or a target description file. | The report lists probes run, probes skipped and why. No findings for a family must never read as "tested and clean". |
| Tier 9 probes | Outside the priority rubric. | Listed in an "informational, unranked" section with evidence. |
| Buffs | A buff changes the technique (for example paraphrase or encoding) but the probe's tags do not reflect it. | Read `buff_spec` from the run config row and add buff-to-technique entries to the KB. |
| Intent-based runs | Results are organised by intent as well as technique. | Use `technique_intent_matrix`; allow KB entries keyed on intent. |
| In-house probes | No upstream tags. | Require tags on every custom probe, or supply a module default in the KB. |

### 9.3 Default mapping by probe family

Starting defaults for the module-level fallback. Based on garak's documentation; confirm the list against `python -m garak --list_probes` on the pinned install, because families are added and renamed between releases.

| Family | What it tests | Default owner | Default control group |
|---|---|---|---|
| `dan` | Persona and "do anything now" jailbreaks | Guardrail team | Jailbreak detection on input; refusal persistence in system prompt |
| `promptinject` | Direct instruction hijacking | Guardrail team | Injection classifier; delimiting of untrusted input |
| `encoding` | Payloads hidden in encodings | Guardrail team | Decode and normalise before inspection |
| `badchars` | Invisible, homoglyph and reordered characters | Guardrail team | Unicode normalisation before inspection |
| `suffix` | Adversarial suffix strings | Guardrail team | Gibberish or perplexity screening on input |
| `phrasing` | Harmful requests rephrased by tense | Guardrail team | Intent-based rather than keyword-based policies |
| `grandma` | Emotional roleplay to extract restricted content | Guardrail team | Roleplay-aware input policy; output category filter |
| `tap`, `atkgen` | Attacker-model driven iterative attacks | Guardrail team | Conversation-level evaluation; detection of repeated probing; rate limits |
| `fitd` | Gradual multi-turn escalation | Guardrail team | Conversation-level evaluation; session risk scoring |
| `latentinjection` | Instructions hidden in documents or context | Application team | Treat retrieved content as data; restrict tool calls triggered by it |
| `web_injection` | Markdown image exfiltration, script and markup output | Application team | Context-aware output encoding; URL allowlist; no raw rendering |
| `exploitation` | Template, SQL and code injection through output | Application team | Never execute or render model output; parameterised queries; sandboxing |
| `ansiescape` | Terminal escape codes in output | Application team | Strip control characters before logs and terminals |
| `agent_breaker` | Tool misuse in agents (inactive by default) | Application team | Narrow tool scopes; confirmation for state-changing actions |
| `apikey` | Producing or completing credentials | Guardrail team | Secret detection on output |
| `malwaregen` | Malicious code generation | Guardrail team | Output code policy; restrict use case scope |
| `realtoxicityprompts`, `continuation`, `lmrc` | Toxic, slur and other content harms | Guardrail team | Enable and tune the matching output content categories |
| `topic` | Engaging with out-of-scope topics | Application team | Scope definition in system prompt; topical guardrail |
| `leakreplay`, `divergence` | Reproducing training data | Model platform | Output similarity filter; output length caps; provider assurance |
| `packagehallucination` | Inventing software packages | Application team | Validate names against an approved registry |
| `misleading`, `snowball` | Accepting false premises; compounding wrong answers | Application team | Grounding with citations; abstain instructions |
| `goodside` | Mixed collection of known attacks | Per probe class | Map each class individually |

Families to look for and map if present in the install (not confirmed for 0.17.1 here): `av_spam_scanning`, `donotanswer`, `doctor`, `dra`, `fileformats`, `glitch`, `sata`, `smuggling`, `visual_jailbreak`, `audio`.

### 9.4 Limit of the approach

The report can only recommend fixes for weaknesses garak probes. Risks outside garak's catalogue (business-logic abuse, access control in retrieval, application-specific policies) need custom probes or another tool, and the report should say so.

## 10. Risks

| Risk | Mitigation |
|---|---|
| Detector false positives produce wrong recommendations | Analyst review rule (7.2), overrides file, low-confidence flag |
| Inferred attribution is wrong | Publish the rates behind each class; move to guardrail verdict export when available (3.5) |
| LLM invents controls | KB grounding (5.3), schema validation, human review of P1 and P2 |
| Hit content manipulates the synthesiser | Treat as data, delimit, validate (5.4) |
| Gateway guardrails block synthesis requests | Redact and fall back to templates (5.5) |
| garak upgrade changes fields or scores | Version pin, schema tests, re-baseline (8.3) |
| Tags read as full compliance coverage | Fixed disclaimer; garak's own documentation warns against this reading |
| Owners outside the red team do not act | Actions exported to the tracker with owner and priority; delta report shows ageing |
| Harmful content in circulated reports | Redaction pass (6.4), access control (8.5) |

## 11. Sequencing and effort

Rough estimate for one engineer, with reviewer time from governance and the guardrail owners.

| Week | Work |
|---|---|
| 1 | Phase 0, Phase 1 |
| 2 to 3 | Phase 2 (KB), Phase 3 (attribution) |
| 4 | Phase 4, Phase 6 |
| 5 | Phase 5 (LLM synthesis) |
| 6 | Phase 7, pilot on two applications, governance review |
| 7 to 8 | Phase 8 |

Minimum useful version: Phases 0, 1, 2, 4 and 6 with template wording only. That is about three to four weeks and already answers "what to fix and who owns it". Attribution and LLM synthesis then improve precision and readability.

## 12. Open questions

1. Can the guardrail owners share block fingerprints, the policy catalogue, or a verdict export keyed by correlation id?
2. Which OWASP LLM Top 10 edition and which internal control framework does governance report against?
3. Will application teams share system prompts and tool definitions for the synthesis step, at least for pilot apps?
4. Where do remediation actions get tracked, and who chases owners?
5. Is a guardrail-off or bare-model comparison run possible in any lower environment?
6. Which model on the gateway is approved for processing adversarial test content?

## 13. References

- garak reporting: https://reference.garak.ai/en/stable/reporting.html
- garak tags and taxonomies: https://reference.garak.ai/en/stable/tags_taxonomies.html
- garak configuration: https://reference.garak.ai/en/latest/configurable.html
- garak attempt fields: https://reference.garak.ai/en/stable/attempt.html
- garak analyze tools: https://reference.garak.ai/en/latest/index_analyze.html
- garak tier-based score aggregation: https://reference.garak.ai/en/latest/analyze/tbsa.html
- garak probe tiers: https://reference.garak.ai/en/latest/probes/_tier.html
- garak releases: https://github.com/NVIDIA/garak/releases
- Promptfoo remediation reports (reference design): https://www.promptfoo.dev/docs/enterprise/remediation-reports/
- Prisma AIRS remediation recommendations (reference design): https://docs.paloaltonetworks.com/ai-runtime-security/new-features/by-date/prisma-airs/january-2026
