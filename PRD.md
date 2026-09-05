# PRD — AI Automation Chrome Extension

**Product Name:** AutoMate (working title) — AI Web Automation Assistant
**Version:** 0.2 (revised — design & logic overhaul)
**Author:** grace23iq
**Status:** Draft for review

---

## 0. What Changed in v0.2

| Area | v0.1 (draft) | v0.2 (this) | Why |
|---|---|---|---|
| Sidebar shell | Injected iframe into page | **Native `chrome.sidePanel` API** | Zero risk to host page, survives navigation, own lifecycle/CSS |
| Component ownership | Everything through service worker | **Clear split**: Panel = UI + streaming; SW = routing + storage + orchestration; content script = page I/O only, injected on demand | MV3 workers sleep after ~30s; streamed chat must not live in SW |
| Context extraction | "Dump the DOM" | **Relevance scoring + token budget + attach-picker** | Model windows are limited; structure beats volume |
| Locators | Single CSS/XPath per element | **Locator v2: ordered candidates + fingerprint fallback** | Pages change; single selector breaks |
| Script updates | Full rewrite on change | **Guided edit ops + step-level diffs + versioning** | Rewriting breaks fixed steps |
| Execution | Run steps blindly | **Preflight health-check, dry-run, pause→edit→resume, guardrails, nav resilience** | Automation only works if it knows when to stop |
| Security | Vague | Explicit redaction, sandboxed evaluate, remote-code ban | Trust is table stakes |

---

## 1. Overview

A Chrome extension that provides a fully functional **AI chat side panel** in the browser. The AI has structured, on-demand access to the current page (DOM, shadow DOM, HTML, CSS, JS handlers, per-element ids/attributes/CSS selectors/XPath/content). From natural-language requests it **generates, validates, and runs automation scripts** that perform repetitive page work. Scripts are **saved, versioned, reused, and updated on the fly** — by chat, by structured edit, or by re-attaching live elements.

All AI calls go through the **OpenRouter API**; the user supplies the key and picks any model.

## 2. Problem Statement

Repetitive browser workflows (form filling, data entry, scraping, QA, admin tasks) still require hand-written Playwright/Puppeteer/Selenium code or brittle record-and-replay tools that break at the first markup change. Users want to **describe the job in plain English** and get automation that is resilient, explainable, and editable — not a one-off recorder dump.

## 3. Goals

- G1 — A fully functional, page-safe AI chat in the browser side panel.
- G2 — Rich, **curated, budgeted** page context so generated scripts target the right elements reliably.
- G3 — Natural-language → validated automation script (not raw JSON that may not run).
- G4 — Scripts that live in a library: save, run, version, update in place without breaking what works.
- G5 — Safe-by-default: preflight checks, dry-run, guardrails, redaction, sandboxing.
- G6 — All AI on OpenRouter; user owns the key and the model choice.

## 4. Non-Goals (v1)

- No Firefox/Safari; Chrome ≥ 114 only (MV3, `chrome.sidePanel`).
- No paid recorder/visual flow editor — generation is chat-driven (recorder = later milestone).
- No cloud sync; local storage only.
- No marketplace, subscriptions, or account system.

## 5. Personas

| Persona | Description | Primary need |
|---|---|---|
| **Automation novice** | Non-dev with repetitive ops/admin tasks | Say it in English; get a working, safe script |
| **QA engineer** | Maintains flaky end-to-end flows | Reliability: health-checks, diffs, fast fixes |
| **Data worker** | Repeated form-filling/scraping | Reuse + resume + run on many pages |

## 6. User Stories

| ID | User | Want | So that |
|---|---|---|---|
| US-1 | All | Open a chat panel on any tab | Work without leaving the page |
| US-2 | All | "Fill this form and submit" | AI creates script targeting the real elements |
| US-3 | All | See the page context the AI used | Trust + catch wrong selector early |
| US-4 | All | **Preflight:** "check the script before it does anything" | Never blind-run against a changed page |
| US-5 | All | **Dry-run** a script | Test without mutating anything |
| US-6 | Novice | Save & reuse a script | Do the task again tomorrow |
| US-7 | All | "Update it to the new login button" | Change one step, keep the rest |
| US-8 | QA | Pause a failing run, fix, resume at same step | Long flows don't have to restart |
| US-9 | All | Switch models / see cost | Control quality & spend |
| US-10 | All | Stop, and know what was touched | No runaway or surprise mutations |

## 7. Design Principles

1. **The page is untouchable.** The chat lives in the native side panel. Nothing is injected unless the user asks to extract or run.
2. **Context is curated, not dumped.** The model gets a scored, budgeted snapshot — enough to act, never more than needed.
3. **Selectors are a ladder, not a rung.** Always generate ordered locators with fallbacks; the runner walks the ladder.
4. **Machines on the observable, not the assumed.** Every step is validated against the live DOM before it runs.
5. **Never mutate silently.** Mutations, uploads, navigation are interrupted visually; destructive actions require one session acknowledgment.
6. **Update in place, not from scratch.** Edits preserve working steps; each change is diffed and versioned.
7. **The user always knows what the model saw and did** — context badges, step log, screenshots on demand.

## 8. UX / Product Design

### 8.1 Panel layout (three tabs)

- **Tab 1 — Chat**: messages, streaming output, code blocks (script) with Run/Explain/Edit actions, context chips ("page @14:32 · 3 attached · 1.2k tokens"), suggestion chips, model picker in the composer bar, stop button during generation.
- **Tab 2 — Scripts** library: search/filter/tag, cards with name, desc, urlPattern, health dot (green/amber/red from last preflight), run/edit/duplicate/export/import/delete; detail view shows steps as read-able rows, version history with diff.
- **Tab 3 — Run console + Settings**: live trace/step progress, controls (play, pause, step, stop, speed), preflight result panel ("2 of 5 targets found — 3 marked missing, review"), and Settings (API key, default model, temperature, max context budget, redaction toggle, domain allow/block list).

### 8.2 Interaction flows

```
Generate:  user prompt → [context badge updates: "attaching page context"]
           → preflight optional → script JSON → validated → rendered in chat
           → user: Run (preflight again → health check) or Save.

Edit:      "user: update the login step" → old script attached → model returns
           edit ops (modify step s2, add step s9) → diff shown → apply → v3.

Resume:    run fails at s5 → paused with highlight → user picks new element
           → locator regenerated for that step → resume from s5 (checkpoint).

Attach:    user picks element (picker overlay) → context chip added to message.
```

## 9. Architecture (v0.2)

```
┌───────────────────────────────────────────────────────────────┐
│ chrome.sidePanel (native side panel, isolated document)        │
│  • Chat UI + streaming (owns the OpenRouter stream)             │
│  • Script library UI  • Run console  • Settings                │
└───────────────┬───────────────────────────────────────────────┘
                │ chrome.runtime messages (never blocked by page)
┌───────────────▼───────────────────────────────────────────────┐
│ Service Worker (MV3)                                          │
│ • message router      • thread/script store (chrome.storage)  │
│ • OpenRouter proxy (simple/quick calls)                       │
│ • chrome.alarms scheduling   • API-key holder                 │
└───────────────┬───────────────────────────────────────────────┘
                │ chrome.scripting.executeScript (on demand only)
┌───────────────▼───────────────────────────────────────────────┐
│ Tab (page world)                                              │
│  content scripts injected per action:                         │
│   – context.js   extractor, locators, budgets, shadow walk    │
│   – picker.js    element picker + attach                      │
│   – runner.js    execution state machine, trace, guardrails   │
└───────────────────────────────────────────────────────────────┘
```

**Ownership rules (why it works with MV3):**
- **The panel** owns all streaming — the panel is a live document while open, so the fetch stream is never cut by service-worker sleep. It reads the API key from `chrome.storage.local`.
- **The SW** stays stateless: routing, persistence, alarms. On wake it rehydrates chat/session state from storage — a chat survives SW restarts.
- **The content script** is injected only when the user explicitly extracts or runs — keeps permissions minimal (`activeTab` + `scripting`), and the page world is never touched for UI.
- **Shadow DOM**: a small `document_start` shim script is the only page-world code. It keeps a registry of new `attachShadow` roots so `context.js` can pierce open **and closed** roots.

## 10. Functional Requirements (MoSCoW)

### FR-1 — Side panel chat (Must)
- FR-1.1 Native `chrome.sidePanel` opens on toolbar click / toggle. Panel switches content per tab; survives navigation.
- FR-1.2 Full chat: history, **streaming** markdown, code blocks with syntax highlight + actions (Run, Copy, Explain, Save).
- FR-1.3 Threads: multiple conversations stored in `chrome.storage.local`; switch restores scroll + context badge.
- FR-1.4 Suggestion chips ("Generate script to…") derived from the current page snapshot.
- FR-1.5 Keyboard navigation, accessible labels, dark/light theme via `prefers-color-scheme`, resize between docked and popout.

### FR-2 — OpenRouter (Must)
- FR-2.1 Key set once in Settings; stored `chrome.storage.local`; never in logs; toggle "show key".
- FR-2.2 `POST https://openrouter.ai/api/v1/chat/completions` with `stream: true` from the panel (streaming survives panel open). Sends `HTTP-Referer`/`X-Title` per OpenRouter policy.
- FR-2.3 Per-chat model/temperature/max tokens/system prompt overrides.
- FR-2.4 Error map: 401 (bad key → link to settings), 402 (credit), 429 (retry-after backoff), model overloaded (auto-fallback suggestion), network (retry button).
- FR-2.5 Cost tracking per message (from OpenRouter usage) displayed in chat footer.

### FR-3 — Page context system (Must — the core differentiator)

**Extraction and deliver as a structured, budgeted payload (schema in §11).**

- FR-3.1 **Extraction scope**: DOM (top document + fragment around selection), **shadow DOM** (open + closed via registry), CSS (inline/stylesheets only rules matching captured elements), JS (event-handler assignments for captured elements — inline `onclick` etc. and a summary of event listeners), element metadata (id, class, name, type, `data-*`, text, value, placeholder, options).
- FR-3.2 **Per-element payload** per §11.1: attrs, content, **locator ladder** (id → data-attr → unique CSS → short CSS → relative XPath → role/text), fingerprint, visible/editable/disabled/focusable state.
- FR-3.3 **Relevance scoring** for the element index: forms/inputs/buttons/links/interactive > static; visible > hidden; children of the user-attached subtree score highest. Order in payload preserves document order within a score tier for model predictability.
- FR-3.4 **Token budget** per model (config, default e.g. 8k context tokens, minus input-buffer): allocations: 60% element index (top-N by score), 15% page HTML structure (excerpts + hashes), 10% context around the attach point, 10% relevant CSS, 5% handler snippets); dropped buckets are declared to the model (`"truncated": true, "dropped": ["css:full"]`) so it never assumes completeness.
- FR-3.5 **Attach via picker**: element picker overlay → focus element → chip with element's locators → added to message. Attach refreshes context to that subtree (strong signal).
- FR-3.6 **Context freshness**: snapshot records `capturedAt` + `url`; when a prompt is sent on a changed location/DOM since capture, an automatic refreshed snapshot occurs and the badge updates; a **stale-context warning** triggers if the diff exceeds a threshold (measured via a hash-chunk diff).
- FR-3.7 **Privacy**: `input[type=password]` values and elements marked `:sensitive` (per-user list) are never included; "redact form values" toggle defaults ON; per-domain blocklist disables context/execute entirely.

### FR-4 — Script generation & editing (Must)

- FR-4.1 Strict output contract: model returns ONLY a JSON `<script block>` in `schema v2` (see §11.2), plus an optional `plan` human sentence.
- FR-4.2 **Two modes**: `CREATE` (from scratch given context) and `EDIT` (given current script + user instruction → produces **operation set** rather than a full rewrite: `update-step {id, newStep}`, `insert-step {id, afterId}`, `delete-step {id}`, `reorder`). Ops applied, then diff rendered. Editing can't silently reorder/punish unchanged steps.
- FR-4.3 Validation gate: JSON.parse → schema validator → locator sanity (each step must include at least 1 resolvable locator+fallback) → execute a **preflight** locator check (above) → only then "Ready to run".
- FR-4.4 One auto-retry for malformed/invalid output (with error message), then manual mode with the raw output for editing.
- FR-4.5 Generated script shows its rationale (`plan` field) for easy human review; the chat renders it as readable steps first, JSON on-demand.
- FR-4.6 `/redo-step s5` style commands and free-form sentences both work (intent parser in SW).

### FR-5 — Script library & versions (Must)

- FR-5.1 Each saved script: `id`, name, description, tags, urlPattern, variables, steps, options, `schemaVersion`, `createdAt`, `versions[]` (last 10), `lastHealth`.
- FR-5.2 Actions: run, edit, duplicate, rename/tag, delete (confirm), export JSON, import, **version history with step-diff and restore**.
- FR-5.3 **Selectors decay tracking**: every run records which locator candidate matched; over time, script getsa "health score" (found % in last runs) and the UI suggests "refreshing" for low-health steps ("this step's locator failed 3× since 2 weeks — redo?").

### FR-6 — Execution engine (Must)

State machine: `IDLE → VALIDATING → RUNNING ⇄ PAUSED → DONE | FAILED | CANCELLED`, with `WAITING_FOR_DOM` (navigation).

- FR-6.1 **Preflight/health check**: on Run, resolve all targets against the live tab **with no mutations**. Result panel shows per-step found/notFound/missing. If any `MUST` steps fail, pause and ask; `OPTIONAL` failures are flagged and skipped. **"Preflight only" button** = check + report, nothing else.
- FR-6.2 **Dry-run**: only observer-safe steps (waitFor/assert/extract/log) execute; all mutations are faked into the log ("would type → 'x'" w/ target) — safe rehearsal.
- FR-6.3 Run **without preflight**: available, but the UX nudges toward it after 1 failed run.
- FR-6.4 Step execution: resolve via locator ladder (each fallback attempted in order **and timing**), verify state (visible/editable), execute action, record trace event (§11.3).
- FR-6.5 Controls: Play, Pause/Resume, Step-through, Stop, Speed (0.5×/1×/2×); step timeout (default 10 s); per-step retries (default 1).
- FR-6.6 **Checkpoints and resume**: state persisted per-step (scriptId, stepId, vars, locator used) — after a pause/restart, resume from the exact next uncompleted step. After navigation, the runner wait-resolves the target for the next step; steps no longer reachable fail gracefully with reason.
- FR-6.7 Live trace in Tab-3: per step: id, action, target, time, matched element, screenshot thumb (optional), status chips. `Download trace` (JSON) — future replay.
- FR-6.8 Nav-safety: navigation is a step (`goto`) — no path triggers navigation except an explicit `goto` step; if the page changes on its own (SPA), runner marks `WAITING_FOR_DOM` and re-resolves.

### FR-7 — Guardrails & security (Must)

- FR-7.1 **Destructive-click detection**: button/link text fine-grained (submit, confirm, delete, pay, buy, order, send, publish, block, cancel-subscription…) → requires a one-time per-session confirmation for that step. Unknown high-signal words → treated as nomal.
- FR-7.2 **No mutations without a visual marker**: when a step mutates (type/click/select/check), the runner briefly highlights the target element in the page (1 s default; toggleable for headless quiet).
- FR-7.3 **Sandbox**: `evaluate` steps default **off** (user must enable per script); when enabled, generated JS is shown and must be accepted once per script, and it runs isolated from the page script's globals wherever possible.
- FR-7.4 Secrets: any value referencing a `secret` variable is never sent to the model and never logged; picker/type steps with `secret: true` mask the value.
- FR-7.5 No remote code; no `chrome://` or store pages; extraction blocked on blocklisted domains (subset of FR-3.7).
- FR-7.6 All storage is `chrome.storage.local`; nothing syncs automatically; "Erase all data" button in Settings.

### FR-8 — Multi-tab & scheduling (Should)

- FR-8.1 Run a script across N matching tabs (urlPattern) sequentially, progress bar, per-tab result.
- FR-8.2 `chrome.alarms`-based scheduled runs (min interval ~ 1 min; only runs when logged in / tab open? scheduling is experimental) — flagged experimental in UI.

## 9. Non-Functional Requirements

- **NFR-1 Performance**: extraction <250 ms typical page; engine overhead <15 % CPU; panel cold start <150 ms.
- **NFR-2 Reliability**: extractor never throws on malformed DOM (wrapped walker); every network call has timeouts; auto-backoff on 429; the runner catches per-step errors with structured reasons, no silent failures.
- **NFR-3 Scale**: extraction handles 50k-node pages by priority sampling (§ 13.4? §11.3); script library to ~500 scripts; 1000 logs.
- **NFR-4 Compatibility**: Chrome ≥ 114; works under strict CSP pages (panel is an extension origin; content uses `executeScript`); no globals leaks into the page.
- **NFR-5 Accessibility**: keyboard-usable, ARIA labels, contrast AA.

## 10. Data Schemas

### 10.1 Page context payload (sent to model)

```json
{
  "version": 2,
  "capturedAt": 1750000000000,
  "url": "https://example.com/form",
  "title": "Signup",
  "viewport": [1440, 900],
  "totals": { "elements": 1234, "shadowHosts": 3, "iframes": 2, "inputs": 12 },
  "attached": ["e-42", "e-43"],
  "truncated": false,
  "dropped": [],
  "budget": { "targetTokens": 3400, "used": 3170 },
  "elements": [
    {
      "id": "e-42",
      "tag": "button",
      "role": "button",
      "attrs": { "id": "send", "data-testid": "send-btn", "type": "submit" },
      "text": "Create account",
      "state": { "visible": true, "editable": false, "disabled": false, "focusable": true },
      "locators": [
        { "type": "id", "value": "send" },
        { "type": "data-attr", "value": "[data-testid=\"send-btn\"]" },
        { "type": "css", "value": "form#signup > button[type=submit]" },
        { "type": "xpath", "value": ".//button[text()=\"Create account\"]", "axis": "relative" },
        { "type": "fingerprint", "value": "e8f2c1a7" }
      ],
      "content": { "value": "", "placeholder": "you@example.com", "options": [] }
    }
  ],
  "shadowRoots": [
    { "host": "wc-card", "depth": 1, "elements": [ { "id": "5", "tag": "input", "locators": [] } ] }
  ],
  "htmlFragments": ["<form id=\"signup\">…(trimmed)…</form>"],
  "eventHandlers": [ { "elementId": "42", "type": "click", "code": "submitForm()" } ],
  "cssRules": ["#send { … }" ]
}
```

### 10.2 Executable script (v2)

```json
{
  "schemaVersion": 2,
  "id": "s-8f3a",
  "name": "fill_signup_and_submit",
  "description": "Fills the signup form and submits",
  "urlPattern": "example.com/form*",
  "variables": [
    { "key": "email", "value": "user@example.com", "secret": false },
    { "key": "token", "value": "{{SECRET:user_api_token}}", "secret": true }
  ],
  "plan": "Fills email & plan, clicks Create, waits for the success banner.",
  "steps": [
    { "id": "s1", "action": "goto", "url": "https://example.com/form", "timeoutMs": 20000 },
    { "id": "s2", "action": "waitFor", "target": { "locator": {"type":"id","value":"email"}, "fallbacks": [{"type":"xpath","value":".//input[@name='email']"}] }, "state": "visible", "timeoutMs": 15000 },
    { "id": "s3", "action": "type", "target": { "locator": {"type":"data-attr","value":"[data-testid=email]"}, "fallbacks": [] }, "value": "{{email}}", "guardrail": true },
    { "id": "s4", "action": "select", "target": { "locator": {"type":"id","value":"plan"} }, "optionText": "Pro" },
    { "id": "s5", "action": "click", "target": { "locator": {"type":"css","value":"button[type=submit]"}, "fallbacks": [{"type":"xpath","value":".//button[text()='Create account']"}] }, "destructive": true },
    { "id": "s6", "action": "waitFor", "target": { "locator": {"type":"css","value":".success-banner"} }, "state": "visible", "timeoutMs": 20000 },
    { "id": "s7", "action": "extract", "target": { "locator": {"type":"css","value":".welcome-message"} }, "saveAs": "welcomeText" }
  ],
  "options": { "retries": 1, "stepTimeoutMs": 10000, "preflight": true, "pauseOnError": true, "guardrails": "default", "evaluate": "off" }
}
```

**Action set (v2):** `goto`, `waitFor` (visible/checked/enabled/url/text), `type`, `clear`, `click`, `select`, `check`, `uncheck`, `extract`, `assert`, `evaluate` (needs allow), `loop/repeat`, `branch` (if var cond), `log`, `pause` (confirmation), `screenshot`.

### 10.3 Update/Edit ops (v2)

```json
{ "mode": "edit", "ops": [
  { "op": "update", "id": "s5", "action": "click", "newTarget": { "locator": {"type":"id","value":"login-now"}} },
  { "op": "insert", "afterId": "s2", "step": { "id": "s2b", "action": "type", ... } },
  { "op": "delete", "id": "s4" }
]}
```

### 10.4 Run trace event (streamed to Tab-3)

```json
{ "ts": 1750000000000, "kind": "step", "scriptId": "s-8f3a", "stepId": "s3", "status": "done", "locatorUsed": "data-attr", "resolveMs": 2, "actionMs": 340, "matched": "input#email", "mutation": true, "screenshot": null }
```

## 12. Logic details worth calling out

- **First-error budget**: extractor runs with a try/catch per node and a 2 s global budget; if exceeded, returns partial with `truncated:true` and `dropped["rest"]` — never hangs.
- **Freshness rule**: prompts always capture the current tab snapshot **at send time** (unless the user explicitly references an attach chip). Attach chips store `capturedAt`; if a re-run is triggered by a locator still found, reuse.
- **Locator ladder failover** is logged; a step that used fallback `n>0` on a successful run flags the script health: 3+ fallback-usages in a row → suggestion to refresh that step.
- **Deterministic retry**: model retry for invalid JSON sends the same prompt + the parse/schema errors, exactly once.
- **Intent classification**: SW classifies message as `question | generate | edit | run | stop | ops-command` — routing and prompts differ per intent; unknown/ambiguous → default to chat with a clarifying prompt.

## 13. Edge Cases & Acceptance Criteria

| # | Case | Behavior |
|---|---|---|
| 1 | SPA adds selector later | Locator ladder resolves fresh at step time; `waitFor` covers late elements |
| 2 | Closed shadow root | Shims registry pierces; if a root is unmounted/opaque, element marked `shadow:unreadable` and model told |
| 3 | Cross-origin iframe | Marked `crossOrigin`, skipped with warning; same-origin traversed |
| 4 | 50k-node page | Priority sampling in budget; hash-lighting Git for full-history need |
| 5 | Malformed model output | 1 structured retry + fallback message |
| 6 | OpenRouter down | Backoff + "there's a problem, retry" and switch-model hint |
| 7 | Nav mid-run | `WAITING_FOR_DOM` state; resume at next step; un-reachable steps fail logically |
| 8 | Element changed between generate & run | Preflight catches; user re-attaches or "fix this step" |
| 9 | Destructive click | One-time confirmation (FR-7.1); script remembers choice per session |
| 10 | `evaluate` step in imported script | Off by default; explicit accept required |
| 11 | Page contains credentials in inputs | Redaction default ON; secrets never captured (FR-7.4) |

## 13. Milestones

- **M1 — Panel + context** (sidePanel, streaming chat, settings/key, extractor with locators v2, shadow registry, budgets, picker-attach)
- **M2 — Generation + library** (create/edit pipeline, validation, diff, versioning, rest of editing tools)
- **M3 — Execution** (preflight, dry-run, state machine, trace, checkpoints, resume, guardrails, fix-with-AI)
- **M4 — Scale & Polish** (multi-tab, scheduling, templates, onboarding, packaging/Web Store)

## 14. Success Metrics

- Activation: % installs chatting ≤48 h (>40%)
- Generation: % prompts → validated script (target >75%)
- Preflight catch: % of run attempts redirected to fix-selector before runaway (target >60% of failing runs)
- Execution success: % runs all steps done (target >85%)
- Reuse: % scripts run 3+ times (target >30%)
- Edit success: % "edit" follow-ups produce an applied diff with working preflight (target >70%)
- Health: mean locator FAIL rate per script/week → should trend down with refresh nudges

## 15. Open Questions

1. Default model vs cost — recommend a default (e.g. Claude Haiku-class or GPT-4o-mini-class) for generate? Which tier?
2. Chrome `sidePanel` requires user install from store usually; anywhere else needed?
3. Multi-tab default behavior: all matching tabs vs. ask each time?
4. Scheduling in MVP or M4 (chrome.alarms semantics on sleep)?
5. Export format: raw JSON, readable Markdown, or both?

## 16. References

- OpenRouter API: https://openrouter.ai/docs
- Chrome `sidePanel` API: https://developer.chrome.com/docs/extensions/reference/api/sidePanel
- MV3 service workers: https://developer.chrome.com/docs/extensions/develop/concepts/service-workers
- Shadow DOM / closed roots: https://developer.mozilla.org/en-US/docs/Web/API/Web_components/Using_shadow_DOM

*Changelog: v0.2 rewrote architecture (sidePanel + ownership split), context system (scoring/budget/curation), locators (ladder+fingerprint), generation (edit ops + preflight), execution (health-checks, dry-run, checkpoints, resume, guardrails), plus design principle section and trace schema.*