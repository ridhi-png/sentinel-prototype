# Sentinel — a live oversight instrument for AI deployments

LIVE LLINK - https://amazing-mermaid-a23f46.netlify.app/

**Team CtrlAI** · Ridhi, IIT Jodhpur · Accenture Innovation Challenge 2026 · Problem Statement 1 — ControlPlane.ai

Sentinel is a working prototype of a real-time layer that sits between any AI model and its users, scoring every response across three lanes — **Performance**, **Cost**, and **Responsibility** — and routing it through a tiered decision (Green / Yellow / Red) instead of waiting for an after-the-fact audit to catch a bad response.

This repo is a live, functioning demo of the scoring pipeline — not just a mockup. It runs in demo mode out of the box (synthetic traffic, so anyone can see it work with zero setup), and switches to a live mode that calls Anthropic's Claude API directly from the browser if you provide your own API key.

---

## What's actually real vs. simulated in this prototype

Being upfront about this, since it matters for a technical review:

| Lane | In this prototype | In production |
|---|---|---|
| **Performance** | *Real, simplified.* In live mode, resamples the same prompt 3× via the Claude API at temperature 1.0, then measures pairwise word-overlap (Jaccard similarity) across the responses. Low agreement → high "semantic entropy" → flagged. | Same idea, but clustering would use sentence embeddings (meaning-level similarity) instead of word overlap, run through a dedicated NLI/consistency model. |
| **Cost** | *Real usage data, simplified formula.* Live mode reads actual `input_tokens` / `output_tokens` from the Claude API response and derives a cost-per-verified-token score. | Same metric, but the "baseline" would be learned per query-type from historical logs rather than a fixed multiplier. |
| **Responsibility** | *Real but basic.* Regex-based detection for emails, phone-like numbers, card-like numbers, and a small sensitive-term list. | Fine-tuned bias/safety/PII classifiers, tuned per deployment, served as low-latency microservices. |
| **Tiering, reviewer queue, shadow mode** | *Fully functional.* Thresholds, queue state, accept/override actions, and the shadow-mode toggle all work live in the UI. | Same logic, backed by a persistent database and real auth instead of in-memory browser state. |
| **Demo mode** | A pool of ten realistic canned examples with jittered scores, auto-playable, so the dashboard is fully demonstrable without any API key. | N/A — demo-only feature for presentation purposes. |

---

## Solution architecture

```
 USER QUERY → AI MODEL (any LLM/agent)
                    │
                    ▼
        ┌─────────────────────────────┐
        │        SENTINEL LAYER        │
        │                              │
        │  PERFORMANCE   COST   RESPONSIBILITY
        │   (resample     (cost/   (regex / classifier
        │    entropy)   verified-   pattern scan)
        │                token)
        │        │         │          │
        │        └────┬────┴────┬─────┘
        │             ▼         
        │        TIER DECISION  
        └─────────────┬────────────────┘
                       │
        ┌──────────────┼───────────────┐
        ▼              ▼               ▼
   GREEN → USER   YELLOW → LOG    RED → HOLD → REVIEWER
                                   (queue: accept / override)

  New deployments start in SHADOW MODE for a calibration
  period — every check still runs and scores, but RED never
  actually holds a live response until calibration ends.
```

See this same diagram rendered live in the app under **"05 — Signal chain."**

---

## Tech stack

**This prototype (what's actually running):**
- Plain HTML5, CSS3, and vanilla JavaScript — no framework, no build step, no `npm install`. Runs from a single file.
- [Anthropic Claude API](https://docs.claude.com) — called directly from the browser (bring-your-own API key) for live-mode response generation and resample-based entropy scoring.

**Planned production stack (described, not implemented here):**
- **Anthropic Claude** (or any LLM) as the model(s) being observed, and optionally as the reasoning engine behind more sophisticated grounding checks.
- **NVIDIA NIM microservices on NVIDIA Triton Inference Server** — the natural home for the Responsibility classifiers and embedding models once they need to run at low latency and high throughput across many deployments, rather than as client-side regex.
- **Sentence-transformer embeddings + a vector store** (e.g. Milvus) to replace the prototype's word-overlap similarity with real semantic clustering.
- **Redis** for live per-query-type cost baselines; **Postgres** for the audit trail and reviewer queue persistence.

This split is intentional: the prototype proves the *decision logic* works end-to-end with real API calls and real token usage, while the README is honest about which components are simplified stand-ins for what a production build would use.

---

## Dependencies

None, to run the demo. It's a static HTML file — open it in any modern browser (Chrome, Firefox, Safari, Edge).

Optional, for live mode: an [Anthropic API key](https://console.anthropic.com/). The prototype uses the `anthropic-dangerous-direct-browser-access` header to call the API directly from client-side JavaScript — this is Anthropic's documented, supported way to do bring-your-own-key browser calls. Your key is kept in memory in the browser tab only; it is never sent anywhere except directly to `api.anthropic.com`, and it is cleared on refresh.

---

## Execution instructions

1. **Clone the repo:**
   ```bash
   git clone <this-repo-url>
   cd sentinel-prototype
   ```

2. **Run it — either works:**
   - Double-click `index.html` to open it directly in a browser, **or**
   - Serve it locally (recommended, avoids any browser file:// restrictions):
     ```bash
     python3 -m http.server 8080
     # then open http://localhost:8080
     ```

3. **Try it in demo mode (default, no setup):**
   - Click **"Auto-play demo traffic"** to watch Sentinel score a live stream of synthetic queries, or type your own query and click **"Run check."**
   - Toggle **Shadow Mode** in the top bar to see how Red-tier responses are logged-only instead of held.
   - Click into the **Reviewer queue** and try Accept / Override on a flagged item.

4. **Try it in live mode (optional):**
   - Click the gear icon (top right), paste an Anthropic API key, and save.
   - Run a check — Sentinel will make 3 real calls to Claude, measure actual resample agreement, read real token usage for the cost score, and scan the real output for responsibility flags.

No build tools, no package manager, no server-side code required.

---

## Repo structure

```
sentinel-prototype/
├── index.html      # the entire application — markup, styles, and logic
└── README.md        # this file
```

## Team

**CtrlAI** — Ridhi, IIT Jodhpur (solo entry)
Accenture Innovation Challenge 2026, Round 1 — ControlPlane.ai
