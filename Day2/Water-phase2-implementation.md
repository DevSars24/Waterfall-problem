# Waterfall/Timeline View — Phase 2 Implementation Journey

**Project:** DebugProbe.AspNetCore
**Feature:** Interactive Waterfall Timeline — Hover Tooltips + Time-Axis Ruler
**Date:** July 4, 2026
**Branch:** `feature/waterfall-view-phase2`
**Status:** ✅ Implemented, Manually Verified, Ready for PR — 

---

## 1. Context — Where We Started

Phase 1 (already merged to `main`) replaced DebugProbe's flat text list of outgoing
HTTP calls with static, server-rendered horizontal bars — a "waterfall" showing
proportional duration of each dependency call within a parent request.

Phase 1 was intentionally **non-interactive**: no tooltips, no ruler, no JS. The
goal of today's session was to plan, review, implement, and verify **Phase 2**,
which adds the interactive layer on top of that foundation.

**Decision:** Phase 2 is the final phase of the waterfall/timeline feature.
Zoom/pan was considered and deliberately dropped — the current feature set
(proportional bars + ruler + tooltips) fully solves the original problem
(spotting the bottleneck at a glance), so no further phase is planned.

```mermaid
flowchart LR
    A[Phase 1 — Merged] -->|static bars, zero JS| B[Phase 2 — Today]
    B --> C[Hover Tooltips]
    B --> D[Time-Axis Ruler]
    B --> E[✅ Feature Complete]

    style A fill:#3a3a3a,color:#aaa
    style B fill:#1f6f43,color:#fff
    style C fill:#245a8d,color:#fff
    style D fill:#245a8d,color:#fff
    style E fill:#144d2e,color:#fff
```

---

## 2. The Full Day, As a Timeline

```mermaid
gantt
    title Today's Session — Phase 2 Waterfall Feature
    dateFormat  HH:mm
    axisFormat  %H:%M

    section Planning
    Wrote Phase 2 prompt for coding LLM        :done, p1, 09:00, 30m
    LLM proposed initial implementation plan   :done, p2, after p1, 20m
    Security review — URL redaction + XSS gap  :done, p3, after p2, 15m
    LLM updated plan with fixes                :done, p4, after p3, 15m
    Plan approved                              :milestone, done, m1, after p4, 0m

    section Implementation
    Branch created (feature/waterfall-view-phase2) :done, i1, after m1, 5m
    HtmlRenderer.cs — ruler + data attributes  :done, i2, after i1, 40m
    debugprobe.css — ruler + tooltip styles    :done, i3, after i2, 30m
    debugprobe-ui.js — tooltip event logic     :done, i4, after i3, 30m
    HtmlRendererTests.cs — new unit test       :done, i5, after i4, 20m

    section Verification
    Fixed local port-in-use error              :done, v1, after i5, 10m
    Ran app, hit /demo/ExecuteExternalRequests :done, v2, after v1, 10m
    Verified ruler + tooltip on /debug page    :done, v3, after v2, 15m
    Confirmed git status — only 4 files changed :done, v4, after v3, 5m

    section Shipping
    git add / commit / push                    :active, s1, after v4, 10m
    Detailed PR description written             :active, s2, after s1, 10m
```

---

## 3. The Problem We Were Solving (Recap)

A flat list tells you *how long* each call took, but not *when* it happened
relative to others, or exactly what the numbers mean without cross-referencing
the card below. Phase 2 closes that last gap — glance at a bar, hover it,
get every detail instantly.

```mermaid
flowchart TD
    A["Phase 1: Static bar\n(visual duration only)"] --> B{"User wants exact\nstart time / URL / status?"}
    B -->|Before Phase 2| C["Scroll down to matching\nHTTP CLIENT card manually"]
    B -->|After Phase 2| D["Hover the bar →\ntooltip shows it instantly"]

    style C fill:#7a4a1f,color:#fff
    style D fill:#1f6f43,color:#fff
```

---

## 4. Planning Phase — Catching Issues Before They Became Bugs

Before any code was written, the implementation plan was reviewed twice.
Two real risks were caught and fixed at the **planning** stage — which is far
cheaper than fixing them after code was written.

```mermaid
flowchart TD
    subgraph Review1["First Plan Review"]
        R1["Risk 1: data-wf-url using RAW request URL\n→ could leak sensitive query params/tokens"]
        R2["Risk 2: Tooltip built via innerHTML\n→ potential XSS if URL contains unsafe chars"]
    end

    subgraph Fix["Plan Updated"]
        F1["data-wf-url now uses GetDisplayTarget()\n— same redaction Phase 1 already trusted"]
        F2["Double-layer encoding:\nserver-side Encode() + client-side escapeHtml()"]
    end

    R1 --> F1
    R2 --> F2
    F1 --> Approved["✅ Plan Approved"]
    F2 --> Approved

    style R1 fill:#7a1f1f,color:#fff
    style R2 fill:#7a1f1f,color:#fff
    style F1 fill:#1f6f43,color:#fff
    style F2 fill:#1f6f43,color:#fff
    style Approved fill:#245a8d,color:#fff
```

---

## 5. What Actually Got Built

```mermaid
flowchart LR
    subgraph Server["Server-Side"]
        H["HtmlRenderer.cs\nBuildWaterfallSection()"]
        H --> H1["Ruler row: 0% / 25% / 50% / 75% / 100%\nticks in ms, InvariantCulture"]
        H --> H2["data-wf-start\ndata-wf-duration\ndata-wf-url (redacted)\ndata-wf-status"]
        H --> H3["All attribute values\nHTML-encoded"]
    end

    subgraph Style["Styling"]
        C["debugprobe.css"]
        C --> C1[".waterfall-ruler-row\n.wf-ruler-tick"]
        C --> C2[".wf-track gridlines\nat 25% intervals"]
        C --> C3[".wf-tooltip\ndark theme, shadow"]
    end

    subgraph Client["Client-Side"]
        J["debugprobe-ui.js"]
        J --> J1["#wfTooltip div\ncreated once, reused"]
        J --> J2["mouseenter / mousemove\n/ mouseleave on .wf-bar"]
        J --> J3["escapeHtml() before\ninserting into DOM"]
        J --> J4["Viewport-bounds check\n(no off-screen clipping)"]
    end

    subgraph Tests["Test Coverage"]
        T["HtmlRendererTests.cs"]
        T --> T1["Verifies ruler markup exists"]
        T --> T2["Verifies data-wf-* attributes present"]
        T --> T3["Verifies URL is redacted, not raw"]
        T --> T4["Verifies HTML-encoding applied"]
    end

    style Server fill:#1c3d5a,color:#fff
    style Style fill:#3d1c5a,color:#fff
    style Client fill:#1f6f43,color:#fff
    style Tests fill:#7a4a1f,color:#fff
```

**Files touched — exactly as scoped, nothing more:**

| File | Role |
|---|---|
| `HtmlRenderer.cs` | Renders ruler + embeds redacted, encoded data attributes |
| `debugprobe.css` | Ruler layout, gridlines, tooltip visual style |
| `debugprobe-ui.js` | Tooltip creation, hover events, safe DOM insertion |
| `HtmlRendererTests.cs` | Confirms markup, redaction, and encoding all present |

No middleware, storage, HTTP handler, or model files were touched — the
blast radius stayed exactly where Phase 1 left it.

---

## 6. The One Real Hiccup — And How It Was Solved

```mermaid
flowchart TD
    A["dotnet run"] --> B["❌ IOException:\nFailed to bind to 127.0.0.1:5281\naddress already in use"]
    B --> C["Diagnosis: an earlier dotnet process\nwas still running in the background"]
    C --> D["Get-Process -Name DebugProbe.SampleApi\n| Stop-Process -Force"]
    D --> E["dotnet run — retried"]
    E --> F["✅ Now listening on\nhttp://localhost:5281"]

    style B fill:#7a1f1f,color:#fff
    style F fill:#1f6f43,color:#fff
```

This wasn't a Phase 2 code bug — it was a leftover process from a previous run
still holding the port. Killing it and re-running fixed it immediately.

---

## 7. Manual Verification — Proof It Works

Steps taken to verify the feature end-to-end:

```mermaid
sequenceDiagram
    participant Dev as Developer (You)
    participant App as SampleApi (localhost:5281)
    participant Probe as DebugProbe Dashboard

    Dev->>App: POST /demo/ExecuteExternalRequests
    App->>App: Calls jsonplaceholder.typicode.com/posts (3178ms)
    App->>App: Calls jsonplaceholder.typicode.com/users (446ms)
    App-->>Dev: 200 OK (total 4076ms)

    Dev->>Probe: Open /debug/{traceId}
    Probe-->>Dev: Renders waterfall with ruler (0/1019/2038/3057/4076 ms)
    Dev->>Probe: Hover the /posts bar
    Probe-->>Dev: Tooltip → Start +217ms, Duration 3178ms, Status 201
```

### Result Screenshot

The final rendered waterfall — ruler ticks scaled correctly to the 4076ms
total duration, and the hover tooltip showing exact start offset, duration,
and status for the `jsonplaceholder.typicode.com/posts` call:

![Waterfall Phase 2 verification screenshot](./images/waterfall_phase2_verification.png)

**Confirmed working:**
- ✅ Time-axis ruler: `0 ms → 1019 ms → 2038 ms → 3057 ms → 4076 ms`
- ✅ Tooltip on hover: `Start: +217 ms`, `Duration: 3178 ms`, `Status: 201`
- ✅ Bars remain proportional and color-correct (green/success shown here)
- ✅ `git status` confirmed only the 4 intended files were modified

---

## 8. Security Posture — Why This Is Safe to Ship

```mermaid
flowchart LR
    A["Outgoing call URL"] --> B["GetDisplayTarget()\n(existing Phase 1 redaction)"]
    B --> C["Server-side Encode()\n(HTML-encode before render)"]
    C --> D["data-wf-url attribute\nin final HTML"]
    D --> E["Client JS: escapeHtml()\nbefore inserting into tooltip DOM"]
    E --> F["✅ Safe tooltip content\nNo raw URL, no unescaped HTML"]

    style F fill:#1f6f43,color:#fff
```

Two independent safety nets — server-side encoding *and* client-side escaping —
mean a single missed edge case doesn't turn into an XSS or data-leak issue.

---

```mermaid
flowchart LR
    P1["Phase 1\nStatic bars"] --> P2["Phase 2\nRuler + Tooltips"]
    P2 --> Done["✅ Feature Complete\nNo Phase 3 planned"]

    style P1 fill:#3a3a3a,color:#aaa
    style P2 fill:#245a8d,color:#fff
    style Done fill:#144d2e,color:#fff
```

---


---

*Documented on July 4, 2026 — DebugProbe.AspNetCore, Waterfall Timeline feature (Phase 1 + Phase 2, final).*
