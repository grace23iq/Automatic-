# PRD — AI Automation Chrome Extension

**Product Name:** AutoMate (working title) — AI Web Automation Assistant
**Version:** 0.1 (Draft)
**Author:** grace23iq
**Status:** Draft for review

---

## 1. Overview

A Chrome extension that embeds a fully functional AI chat interface in a **sidebar**. The AI has direct access to the current page's structure and content (DOM, shadow DOM, HTML, CSS, JS, element attributes, CSS selectors, XPath, and element text/content). When the user asks in natural language, the AI generates an **automation script** that performs repetitive actions on the page. Scripts can be **saved, reused, and updated on the fly** to adapt to the user's changing needs.

The extension uses the **OpenRouter API** as its model gateway, so any model available on OpenRouter can be used.

---

## 2. Problem Statement

Repetitive browser workflows (form filling, data entry, scraping round-trips, QA flows, admin tasks) currently require either hand-written automation code (Playwright/Puppeteer/Selenium) or brittle record-and-replay tools that break the moment a page's markup changes.

Users want to **describe what to automate in plain English** and have the automation built for them, resilient to page structure, and editable — not buried in a one-off recorder session.

---

## 3. Goals

- G1 — Provide a fully functional AI chat sidebar in the browser.
- G2 — Give the AI rich, structured page context so generated scripts target the right elements reliably.
- G3 — Let users generate automation scripts purely from natural-language prompts.
- G4 — Let users save scripts to a library, run them anytime, and update them via chat or direct edit.
- G5 — Keep all AI calls on OpenRouter so the user controls the model and API key.

## 4. Non-Goals (v1)

- No cross-browser support (Firefox/Safari) in v1 — Chrome only.
- No visual "no-code recorder" — generation is via AI chat (a recorder can be a later milestone).
- No cloud sync of scripts — local storage only for v1.
- No paid marketplace / subscription.

---

## 5. User Personas

| Persona | Description | Primary need |
|---|---|---|
| **Automation novice** | Non-dev with repetitive admin/ops tasks | Describe task in English, get a working script |
| **QA / test engineer** | Writes/maintains flaky end-to-end flows | Reliable selectors, re-runnable scripts, quick fixes |
| **Data worker** | Regular form-filling/scraping of repeated pages | Repeat the same flow many times across pages |

---

## 6. User Stories

| ID | As a... | I want to... | So that... |
|---|---|---|---|
| US-1 | User | Open a chat sidebar on any tab | I can talk to the AI without leaving the page |
| US-2 | User | Ask "fill this form and submit it" | AI creates a script targeting the actual elements on the page |
| US-3 | User | Inspect the page context the AI sees | I understand why a selector was chosen, and can correct it |
| US-4 | User | Save a generated script | I can reuse it later |
| US-5 | User | Run a saved script | Page actions execute automatically |
| US-6 | User | Tell AI "update the script to use the new login button" | The existing script is modified, not rewritten from scratch |
| US-7 | User | Re-generate an element reference | The script doesn't break when the page changes (re-resolve via role/text/relative XPath fallback) |
| US-8 | User | Switch models (GPT-4o, Claude, or cheap model) | I control cost/quality |
| US-9 | User | Stop an in-progress script | Long loops don't run away |

---

## 7. Functional Requirements

### FR-1 — Sidebar Chat Interface (fully functional)

- FR-1.1 Inject a resizable, collapsible sidebar panel into any tab (all origins except chrome:// and the Web Store).
- FR-1.2 Full chat: message history, streaming responses (token-by-token), markdown rendering, syntax-highlighted code blocks for scripts.
- FR-1.3 **Conversation threads**: multiple chats, each with own history/context; restoring a thread restores scroll state.
- FR-1.4 Must stay functional on SPAs and pages that mutate their DOM (single injected host + content script that survives navigation).
- FR-1.5 Provide one-tap suggestions like "Generate script to …".
- FR-1.6 Never break the host page (sandboxed iframe or shadow DOM container with isolated CSS).

### FR-2 — OpenRouter Integration

- FR-2.1 API key user-configurable in options; stored securely via `chrome.storage.local` (never hardcoded).
- FR-2.2 Call `https://openrouter.ai/api/v1/chat/completions` with `stream: true` via Fetch from the extension.
- FR-2.3 Configurable per chat: model, temperature, max_tokens, system prompt; send `HTTP-Referer`/`X-Title` per OpenRouter policy.
- FR-2.4 Handle errors: 401 bad key, 402 insufficient credit, 429 rate limit (retry-after), model-overloaded retry.
- FR-2.5 (Future) "bring your own OpenAI-compatible endpoint" as drop-in.

### FR-3 — Page Context Extraction & Delivery (core differentiator)

The extension must hand the model a complete, structured snapshot of the page.

- **FR-3.1 Extract:**
  - Top-of-document **DOM (HTML)** + the relevant HTML fragment around the user's selection.
  - **Shadow DOM**: pierce open and closed shadow roots (debug-shim injected before page scripts; capture `shadowRoot` descendants recursively).
  - **CSS**: stylesheet text selectively (inline style, `<style>` blocks, rules matching extracted elements).
  - **JS**: event handler attachments relevant to targeted elements (inline `onclick`/etc.), so the AI knows what triggers when an element is activated.
  - **Element metadata**: **id**, `data-*` attributes, **CSS selector (short + unique)**, **XPath (absolute + relative)**, **element content** (text, `value` for inputs, option lists for selects).
- **FR-3.2 Context payload schema** (§10): each element in a stable JSON block the model can index: `{id, tag, attributes, content, cssSelector, xpath, visible, editable}`.
- **FR-3.3 Dynamic attach**: element-picker mode — click an element → snapshot that subtree into the conversation ("Attach element").
- **FR-3.4 Curation/truncation**: prioritize main content, drop `script`/`style` bodies, cap shadow-DOM depth (default max 10).
- **FR-3.5 Privacy**: toggle to strip form values / sensitive fields; per-domain allow/blocklist to disable extraction.

### FR-4 — AI-Driven Automation Script Generation

- **FR-4.1** From the last user message (+ attached context), the model returns a script in a **strict schema** (see §10): `goto`, `waitFor`, `type`, `click`, `select`, `check`, `extract`, `evaluate`, `assert`, `loop`, `repeat`.
- **FR-4.2** Generated scripts use **multi-layer element resolution**: id → data-attr → unique CSS → relative XPath → role/text fallback.
- **FR-4.3** Scripts rendered as code block with Copy / Run, and a step list linking to target elements.
- **FR-4.4** Refinement loop: follow-up prompts return with the script + conversation history → co-edited by the model.
- **FR-4.5** Deterministic output: model returns only script JSON (+ short comment); parsed and validated before execution; invalid → one regeneration.

### FR-5 — Script Library (save / reuse / update on the fly)

- **FR-5.1** Save generated scripts with name/description/tags to `chrome.storage.local` (structured, versioned).
- **FR-5.2** Library UI in sidebar: search/filter, run, edit, duplicate, delete, export (JSON), import.
- **FR-5.3** Run saved scripts in a sandboxed content-script context with DOM helper APIs.
- **FR-5.4** Update on the fly:
  - *By chat*: "update the script to X" → old script + new request to model → save as v2.
  - *By edit*: structured JSON editor with reorderable/inline-edit steps and live validation.
  - *By re-attach*: pick new element on page → regenerate that action's selector → save.
- **FR-5.5** Version history (last ~10) with diff + restore.

### FR-6 — Execution Engine

- FR-6.1 Sequential step runner with logging/tracing: status, timing, matched element, screenshot thumbnail (if permitted).
- FR-6.2 Controls: Play, Pause/Resume, Stop, Speed (0.5×/1×/2×), step timeout (default 10s), per-step retry (default 1).
- FR-6.3 Wait-based waits (`waitFor` selector / network idle) instead of fixed sleeps where possible.
- FR-6.4 On failure: highlight failing element, show step in console, offer "Fix with AI" (see FR-4.4).
- FR-6.5 Before/after screenshots (optional).

---

## 8. Non-Functional Requirements

- **NFR-1 Security**: script code runs only in the extension sandbox, never the page; sidebar content isolated; `evaluate` output serialized/banned by default; no remote code loading; CSP-clean.
- **NFR-2 Privacy**: page snapshot sent to OpenRouter only when user triggers a chat generation; sensitive mode toggle; local-only storage.
- **NFR-3 Performance**: snapshot generation < 300 ms on typical pages; engine < 20% CPU overhead; sidebar cold-start < 150 ms.
- **NFR-4 Reliability**: extractor never crashes on malformed DOM; timeout everywhere; auto-retry with backoff on 429.
- **NFR-5 UX**: keyboard navigation, dark/light theme, ARIA labels.
- **NFR-6 Compatibility**: Chrome ≥ 114 (MV3); works under strict CSP pages; no conflict with page libs.

---

## 9. Architecture (High Level)

```
Page (window)                        Extension (isolated)
┌───────────────────┐   postMessage   ┌──────────────────────────┐
│ content script     │ ◄─────────────► │ service worker (MV3)     │
│ (MAIN world)       │                 │ • OpenRouter API calls    │
│ - page snapshot()   │                 │ • script runner           │
│ - element picker    │                 │ • library store manager   │
│ - run scripts       │                 └───────────┬──────────────┘
└───────────────────┘                            chrome.storage
        ▲         (sidebar iframe)             (scripts, keys,
        │          shadow DOM host             settings, history)
        └──────────────────────────────────────────────►
```

Components:
1. **Content Script (isolated)** — injects side panel iframe, message bridge to service worker.
2. **Page-World Helper script (MAIN world)** — DOM walking, shadow roots, event listeners, snapshot; executes automation steps "as the user".
3. **Service Worker** — state, chat orchestration, OpenRouter calls (streamed), script storage, execution queue.
4. **Sidebar UI (iframe)** — chat + script library + runner panel + settings.
5. **Options page** — API key, model defaults, allow/block list.

**Generate flow:** user prompts → content script captures snapshot → service worker builds system prompt + context payload → OpenRouter (streamed) → model returns script JSON → validated → rendered → save/run.

---

## 10. Data Schemas

### 10.1 Page Context Payload

```json
{
  "url": "https://example.com/form",
  "title": "Signup",
  "timestamp": 1750000000000,
  "viewportSize": [1440, 900],
  "domSummary": { "totalElements": 1234, "forms": 1, "inputs": 12, "iframes": 2, "shadowHosts": 3 },
  "elements": [
    {
      "id": "email",
      "tag": "INPUT",
      "type": "email",
      "name": "email",
      "attributes": { "autocomplete": "off", "data-testid": "signup-email" },
      "content": { "value": "", "placeholder": "you@example.com" },
      "selectors": {
        "css": "#email",
        "cssUnique": "form#signup > input[name=email]",
        "xpathAbs": "/html/body/main/form/input[1]",
        "xpathRel": "//input[@name='email']"
      },
      "visible": true,
      "editable": true,
      "isShadow": false
    }
  ],
  "shadowRoots": [
    {
      "hostId": "wc-card",
      "hostTag": "my-card",
      "depth": 1,
      "elements": [ { "id": "confirm-btn", "tag": "button", "selectors": { } } ]
    }
  ],
  "scripts": ["<inline handler snippets attached to extracted elements>"],
  "styles": ["<minified CSS rules matching extracted elements>"]
}
```

### 10.2 Executable Script Schema

```json
{
  "name": "fill_signup_and_submit",
  "description": "Fills the signup form and submits",
  "steps": [
    { "action": "goto", "url": "https://example.com/form", "timeout": 15000 },
    { "action": "waitFor", "selector": { "css": "#email" }, "state": "visible", "timeout": 10000 },
    { "action": "type", "selector": { "css": "#email" }, "value": "user@example.com" },
    { "action": "select", "selector": { "xpath": "//select[@id='plan']" }, "optionText": "Pro" },
    { "action": "click", "selector": { "css": "button[type=submit]", "fallback": { "xpath": "//button[text()='Sign up']" } } },
    { "action": "assert", "selector": { "css": ".success-banner" }, "state": "visible" },
    { "action": "extract", "selector": { "css": ".welcome-message" }, "saveAs": "welcomeText" }
  ],
  "variables": { "welcomeText": "" },
  "options": { "retries": 1, "stepTimeoutMs": 10000, "pauseOnError": false }
}
```

**Action dictionary:** `goto`, `waitFor`, `type`, `clear`, `click`, `select`, `check`, `uncheck`, `extract`, `assert`, `evaluate`, `loop`, `repeat`, `if`, `log`.

**Selector resolution order:** id → data-testid → unique CSS → short → relative XPath → text match.

---

## 11. Edge Cases

| # | Edge case | Behavior |
|---|---|---|
| 1 | SPA — element added by JS route | Steps re-resolve on run; `waitFor` covers late-appearing elements |
| 2 | Closed shadow root | Pre-injected shim captures it; if capture fails, AI is told and generates fallback |
| 3 | Iframes | Same-origin traversed; cross-origin flagged and skipped with warning |
| 4 | Huge pages (50k nodes) | Sample top ~2k relevant nodes + source hash; attach specific element to focus |
| 5 | Malformed script JSON | Validate → regenerate once → else show error in chat |
| 6 | OpenRouter down/rate-limited | Backoff retry; user can switch model |
| 7 | Page navigates mid-run | Runner respawns in tab; missing-previous-page steps fail gracefully |
| 8 | Page changes mid-script | Re-verify target per step; if not found, pause + "Fix with AI" |
| 9 | Login walls / CSRF frames | Nothing auto-submitted without explicit `click` step |
| 10 | Malicious model output | Discrete schema steps only; no arbitrary code outside sandboxed `evaluate` (off by default) |

---

## 12. Milestones / Roadmap

**M1 — Chat & Context**
- Sidebar shell (resize/collapse/theme), streaming chat, OpenRouter wiring + key settings
- Page snapshot extractor: HTML, attrs, selectors, XPath, shadow-DOM, curation/limits

**M2 — Generation & Library**
- Script generation with strict schema + validation
- Script renderer/editor (JSON + step list)
- Save/load/list/export/delete in `chrome.storage.local`

**M3 — Execution & Resilience**
- Step runner with log panel, run/pause/stop, speed, timeout/retries
- Failure → "Fix with AI" loop, selector re-resolution, version history + diff
- Element picker (highlight + attach)

**M4 — Polish & Scale**
- Template gallery, multi-page flows, scheduling (cron), screenshots, onboarding, packaging (zip, Web Store listing)

---

## 13. Success Metrics

- Activation: % installs that chat within 48h (target > 40%)
- Generation: % prompts with valid script (target > 80%)
- Execution: % runs completing all steps (target > 85%)
- Reuse: % saved scripts run 3+ times (target > 30%)
- Fix loop: median failed-run → working-rerun time via "Fix with AI" (< 2 min)

---

## 14. Open Questions

1. Default model for new users (cost vs capability)?
2. Permissions: `activeTab` + `scripting` + `storage` vs domain allowlist?
3. Multi-tab execution in v1? 
4. Scheduling (cron) in MVP or M4?
5. Export format: raw and/or human-readable Markdown?

---

## 15. References

- OpenRouter API: https://openrouter.ai/docs
- Chrome MV3: https://developer.chrome.com/docs/extensions
- Shadow DOM: https://developer.mozilla.org/en-US/docs/Web/API/Web_components/Using_shadow_DOM
- XPath: https://developer.mozilla.org/en-US/docs/Web/XPath