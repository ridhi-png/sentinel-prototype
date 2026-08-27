# Sentinel — a live oversight instrument for AI deployments

**Team CtrlAI** · Ridhi, IIT Jodhpur · Accenture Innovation Challenge 2026 · Problem Statement 1 — ControlPlane.ai · **Round 2 submission**

Sentinel is a working prototype of a real-time layer that sits between any AI model and its users, scoring every response across three lanes — **Performance**, **Cost**, and **Responsibility** — and routing it through a profile-aware, four-tier decision (Allow / Auto-edit / Flag / Block) instead of waiting for an after-the-fact audit to catch a bad response.

This document explains what changed for Round 2, what's real vs. simulated, and how the prototype maps to the specific real-world complexities named in the Round 2 brief.

---

## What's new in Round 2

Round 1 scored every response the same way, with one fixed threshold. The Round 2 brief specifically calls out that "a single, one-size-fits-all checking approach rarely works well everywhere" — so this version adds:

| Addition | Why (mapped to the brief) |
|---|---|
| **Three deployment profiles** (Customer Support Bot / Internal Copilot / Decision-Support Tool), each with its own latency budget and per-lane thresholds | *"Different AI use cases... have very different risk tolerance and latency budgets."* Switching profiles in the UI and re-running the same query visibly changes the verdict — the same response that's auto-blocked for a decision-support tool can pass for an internal copilot. |
| **A fourth tier: Auto-edit**, sitting between Allow and Flag | *"Bias, hallucination, and privacy risks often overlap... a fabricated detail about a person can simultaneously be a hallucination and a privacy concern."* Rather than force every responsibility hit into a binary block/pass, PII-shaped content gets redacted and delivered instead of fully blocked — a deliberate, visible design choice for the alert-fatigue-vs-liability tradeoff the brief names directly. |
| **A confidence score on every decision** | The brief notes there's often no reliable real-time ground truth to check a claim against. Sentinel doesn't pretend otherwise — every verdict ships with a confidence percentage instead of a bare true/false, so a reviewer knows how much to trust the flag itself. |
| **A geography/regulatory selector** (US / EU / other) that changes the audit retention policy shown | *"Regulatory expectations differ by geography... and continue to evolve, so rigid, hard-coded rules age quickly."* This is a governance stub, not real compliance logic — see limitations below — but it demonstrates where jurisdiction-specific policy would plug in. |
| **A feedback loop**: every Accept/Override in the reviewer queue now feeds a live false-positive/false-negative estimate and a composite trust score | *"How you would define, measure, and report false positive/negative rates and overall system trustworthiness to a skeptical stakeholder."* This is the panel a compliance lead would actually look at. |
| **Audit trail export** (JSON) | Every finalized decision — profile, geography, scores, confidence, reasons, timestamp — is logged and exportable, addressing the brief's call for "a clear audit trail behind every decision." |

The core three-lane mechanism from Round 1 is unchanged; Round 2 wraps it in the governance, tiering, and feedback layer that a real multi-use-case deployment would actually need.

---

## Stated assumptions (per the brief's reference parameters)

The brief is explicit that its reference parameters are "directional, not a fixed dataset" and asks teams to state their own assumptions clearly. Here are ours:

- **Scale assumed:** tens of thousands of interactions per week, spread across the three profiles modeled (customer support, internal copilot, decision-support) — this shaped the choice to keep detection lightweight enough to run inline rather than as a slow batch process.
- **Data governance assumed:** a mix of well-governed and loosely governed internal sources feed these AI systems, which is why the Responsibility lane does not assume a clean, structured source of truth to check against — it works at the input/output layer only, consistent with the brief's note that most enterprises consume a foundation model via API rather than owning it.
- **Regulatory jurisdiction:** the demo defaults to a US frame with an EU option shown for comparison. This is illustrative — see limitations.
- **No ground truth for verification:** rather than pretend to solve hallucination detection outright, Sentinel treats *resample disagreement* as a proxy signal for "the model itself is unsure," which is a real, defensible technique that doesn't require an external source of truth — directly addressing the brief's point that hallucination and verification often share the same underlying knowledge gap.

---

## What's actually real vs. simulated

Being upfront about this matters more at this stage, since Round 2 is judged as a working mechanism, not just a concept:

| Component | In this prototype | In production |
|---|---|---|
| **Performance lane** | *Real, simplified.* Live mode resamples the same prompt 3× via the Claude API at temperature 1.0, then measures pairwise word-overlap (Jaccard similarity). Low agreement → high entropy → flagged. | Sentence-embedding clustering instead of word overlap, plus retrieval verification against source documents where available (the brief's "AI-as-judge" and "retrieval verification" patterns). |
| **Cost lane** | *Real usage data, simplified formula.* Reads actual `input_tokens`/`output_tokens` from the API response. | Baseline learned per query-type from historical logs rather than a fixed multiplier. |
| **Responsibility lane** | *Real but basic.* Regex detection for emails, phone-like numbers, card-like numbers, a small sensitive-term list. | Fine-tuned bias/safety/PII classifiers, tuned per deployment and per geography. |
| **Deployment profiles & thresholds** | *Fully functional*, hand-set per profile. | Thresholds would be tuned against labeled historical data per use case, not hand-picked. |
| **Confidence score** | *Illustrative heuristic* — distance from threshold boundary, adjusted by entropy. Clearly not a calibrated probability. | A calibrated confidence model, likely trained against reviewer agreement over time. |
| **FP/FN rate & trust score** | *Simulated from session feedback only* (Accept/Override ratio in this browser tab). Resets on refresh. | Computed from a persistent, audited log of reviewer decisions across the whole deployment. |
| **Geography/retention selector** | *UI stub* — changes displayed text only, no real policy engine behind it. | A real configurable policy layer, per the brief's "Governance" solutioning area — see roadmap in the Business Proposal. |
| **Audit trail export** | *Fully functional* for this session's data, exports real JSON. | Persisted server-side with tamper-evident logging, not a client-side download. |
| **Demo mode** | A pool of realistic canned examples, auto-playable, so the dashboard is fully demonstrable without any API key. | N/A — demo-only feature for presentation purposes. |

---

## Solution architecture

```
                    ┌─────────────────────────────┐
                    │  POLICY LAYER — per profile   │
                    │  & geography                  │
                    └──────────────┬────────────────┘
                                   │
 USER QUERY → AI MODEL (any LLM/agent)
                    │
                    ▼
        ┌─────────────────────────────┐
        │        SENTINEL LAYER        │
        │                              │
        │  PERFORMANCE   COST   RESPONSIBILITY
        │        │         │          │
        │        └────┬────┴────┬─────┘
        │             ▼
        │      TIER DECISION + confidence score
        └─────────────┬────────────────┘
                       │
     ┌─────────────┬───┴───────┬──────────────┐
     ▼             ▼           ▼              ▼
 ALLOW→USER   EDIT→REDACT  FLAG→LOG      BLOCK→HOLD→REVIEWER
                                           (accept / override)
                                                 │
                                                 ▼
                              feeds back into confidence & trust metrics

  New deployments run in SHADOW MODE for a calibration period —
  every check still runs and scores, but BLOCK never actually
  holds a live response until calibration ends.
```

See this diagram rendered live in the app under **"06 — Signal chain."**

---

## Tech stack

**This prototype (what's actually running):**
- Plain HTML5, CSS3, and vanilla JavaScript — no framework, no build step, no `npm install`. Runs from a single file.
- [Anthropic Claude API](https://docs.claude.com) — called directly from the browser (bring-your-own API key) for live-mode response generation and resample-based entropy scoring.

**Planned production stack (described, not implemented here):**
- **Anthropic Claude** (or any LLM) as the model(s) being observed.
- **NVIDIA NIM microservices on NVIDIA Triton Inference Server** for the Responsibility classifiers and embedding models once they need low latency and high throughput across many deployments, rather than client-side regex.
- **Sentence-transformer embeddings + a vector store** (e.g. Milvus) to replace word-overlap similarity with real semantic clustering, and to support the brief's "retrieval verification against source documents" pattern.
- **Redis** for live per-query-type cost baselines; **Postgres** for the persistent, tamper-evident audit trail and reviewer queue.
- **A rules/policy engine** (e.g. OPA — Open Policy Agent) to back the geography/use-case governance layer with real, versioned policy instead of the current UI stub.

---

## Dependencies

None, to run the demo. It's a static HTML file — open it in any modern browser (Chrome, Firefox, Safari, Edge).

Optional, for live mode: an [Anthropic API key](https://console.anthropic.com/). The prototype uses the `anthropic-dangerous-direct-browser-access` header for bring-your-own-key browser calls — Anthropic's documented, supported approach. Your key stays in memory in the browser tab only and is cleared on refresh.

---

## Execution instructions

1. **Clone the repo:**
   ```bash
   git clone https://github.com/ridhi-png/sentinel-prototype.git
   cd sentinel-prototype
   ```

2. **Run it:**
   - Double-click `index.html`, or
   - Serve it locally:
     ```bash
     python3 -m http.server 8080
     # then open http://localhost:8080
     ```

3. **Try the Round 2 features specifically:**
   - Pick a **deployment profile** at the top and note the latency budget and threshold readout change.
   - Run the same query under **Customer Support Bot** vs. **Decision-Support Tool** — watch the verdict change even though the underlying scores didn't.
   - Try a query containing something that looks like an email or phone number and watch it land in **Auto-edit** rather than a hard block.
   - Resolve a few queue items with **Accept** vs. **Override** and watch the **Trust metrics** panel move.
   - Switch the **geography selector** and watch the retention policy text change.
   - Click **Export audit trail** to download the session's decisions as JSON.

4. **Live mode (optional):** click the gear icon, paste an Anthropic API key, and run a real check — 3 live calls to Claude, real token usage, real regex scan on real output.

No build tools, no package manager, no server-side code required.

---

## Known limitations (stated deliberately, not discovered by a reviewer)

- Thresholds per profile are hand-set for this demo, not learned from data — the brief's "reference parameters" are directional, and so are these.
- The geography selector changes displayed policy text only; it is not backed by a real rules engine yet (see roadmap in the Business Proposal for the OPA-based plan).
- FP/FN estimates reset per browser session and are derived only from in-session Accept/Override actions — a real deployment would compute these from a persistent, audited log at much larger sample sizes.
- Multi-turn conversation risk and agentic (action-taking) risk, both named explicitly in the brief as compounding risk, are not modeled in this prototype — each check is treated as a single, independent turn. This is the most significant open gap and is called out as a Phase 2 item in the roadmap.

---

## Repo structure

```
sentinel-prototype/
├── index.html      # the entire application — markup, styles, and logic
└── README.md        # this file
```

## Team

**CtrlAI** — Ridhi, IIT Jodhpur (solo entry)
Accenture Innovation Challenge 2026, Round 2 — ControlPlane.ai
