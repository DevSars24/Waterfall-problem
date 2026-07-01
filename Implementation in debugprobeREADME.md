# Waterfall / Timeline View — Implementation Plan (Phase 1)

## 1. The Problem, Explained With a Real-Life Example

Imagine a food delivery backend. A single incoming request — "place an order" — internally triggers three outgoing calls:

1. Call the **Restaurant API** to confirm the order → takes 800ms
2. Call the **Payment API** to charge the customer → takes 400ms
3. Call the **Delivery API** to assign a rider → takes 600ms

Run sequentially, total time = 800 + 400 + 600 = **1800ms**.

Today DebugProbe captures all of this but shows it as a flat **list**:

```
- Restaurant API → 800ms
- Payment API → 400ms
- Delivery API → 600ms
```

A list tells you *how long* each call took, but not:
- **When** each call started relative to the others (sequential? parallel? overlapping?)
- **Which** call is the actual bottleneck at a glance
- **Where** the "dead time" is

### The same data, as a waterfall

```mermaid
gantt
    title Order Request — Outgoing Calls Timeline
    dateFormat X
    axisFormat %Lms

    section Order Request
    Restaurant API (bottleneck) :crit, restaurant, 0, 800
    Payment API                 :active, payment, 800, 400
    Delivery API                :delivery, 1200, 600
```

One glance at the chart tells you Restaurant API is the longest bar → it's the bottleneck. Text can't give you that "aha" instantly; a timeline can. **This is the entire value proposition of Phase 1.**

---

## 2. Scope Decision: Simple Bars vs. Full DevTools Clone

```mermaid
flowchart LR
    A[Two options] --> B["Simple CSS bars\n(left% + width%)"]
    A --> C["Full DevTools clone\n(zoom, tooltips, canvas/SVG)"]

    B --> B1[1–2 days effort]
    B --> B2[Zero new dependencies]
    B --> B3["Matches HtmlRenderer.cs\nexisting server-HTML pattern"]
    B --> B4[Small diff, fast review]

    C --> C1[4–5+ days effort]
    C --> C2[Possibly a charting/canvas lib]
    C --> C3[New client-rendering approach\ninconsistent with codebase]
    C --> C4[Large diff, slow review]

    B4 --> D{{"Decision: ship B\n(Phase 1)"}}
    C4 -.->|"deferred"| E[["Phase 2 candidate"]]

    style D fill:#1f6f43,color:#fff,stroke:#144d2e
    style E fill:#5b5b5b,color:#fff,stroke:#333
    style B fill:#245a8d,color:#fff
    style C fill:#7a4a1f,color:#fff
```

**Decision: Phase 1 ships simple CSS bars only.** This matches the existing pattern in `HtmlRenderer.cs`, where every dashboard element (`BuildTraceCard`, `BuildOutgoingRequestCard`, `BuildHeaderSection`) is a server-rendered HTML string with CSS classes — not client-side JS rendering. Hover-tooltips or zoom become a Phase 2 proposal only after the maintainer accepts the simple version.

---

## 3. What Data Already Exists (No New Data Collection Needed)

The most important reason this feature is low-risk: **the data waterfall needs is already being captured.**

```mermaid
flowchart TD
    subgraph Existing["Already captured — no changes needed"]
        direction TB
        E1["DebugOutgoingRequest.cs\nMethod, Url, StatusCode, Duration, start timestamp"]
        E2["DebugEntry.cs\nParent request start time + total duration\n+ list of OutgoingRequests"]
    end

    subgraph New["New — Phase 1 presentation math only"]
        direction TB
        N1["left% = (call.StartTime − parent.StartTime)\n÷ parent.TotalDuration × 100"]
        N2["width% = call.Duration\n÷ parent.TotalDuration × 100"]
    end

    E1 --> N1
    E1 --> N2
    E2 --> N1
    E2 --> N2
    N1 --> R["HTML bar:\n<div style='left:X%; width:Y%'>"]
    N2 --> R

    style Existing fill:#1c3d5a,color:#fff
    style New fill:#3d1c5a,color:#fff
    style R fill:#1f6f43,color:#fff
```

So Phase 1 requires **zero changes to data capture** — `DebugProbeMiddleware.cs`, `DebugProbeHttpClientHandler.cs`, and `DebugEntryStore.cs` all stay untouched. No new algorithms, no new state, no persistence changes — just two percentages per call.

---

## 4. Files That Will Change, and Why

```mermaid
flowchart LR
    subgraph Verify["Verify only"]
        F1["DebugOutgoingRequest.cs\nConfirm ms-precision start time exists"]
    end

    subgraph Core["Core change"]
        F2["HtmlRenderer.cs\n+ BuildWaterfallSection()\ncalled from RenderDetailsPage"]
    end

    subgraph Style["Styling"]
        F3["CSS asset\n.waterfall-row / .wf-track / .wf-bar"]
    end

    subgraph Maybe["Touch only if new file added"]
        F4["EmbeddedResources.cs\nregister new asset, if any"]
    end

    subgraph NoChange["No change expected"]
        F5["ResourceLoader.cs"]
        F6["DebugProbeMiddleware.cs"]
        F7["DebugEntryStore.cs"]
        F8["DebugProbeHttpClientHandler.cs"]
    end

    F1 --> F2 --> F3 --> F4

    style Verify fill:#7a4a1f,color:#fff
    style Core fill:#1f6f43,color:#fff
    style Style fill:#245a8d,color:#fff
    style Maybe fill:#5b5b5b,color:#fff
    style NoChange fill:#3a3a3a,color:#aaa
```

| File | Change | Why this file |
|---|---|---|
| `DebugOutgoingRequest.cs` | **Verify only** — confirm the stored timestamp is millisecond-precise and relative/convertible to the parent request's start time. Add one property only if missing. | This is the data source for the bar's position. If start-time isn't precise, the whole waterfall math breaks — check this before writing any UI code. |
| `HtmlRenderer.cs` | Add `BuildWaterfallSection(DebugEntry entry)` — loops `entry.OutgoingRequests`, computes `left%`/`width%`, returns HTML bar rows. Called from inside `RenderDetailsPage`, placed above the existing outgoing-request cards (which stay, unchanged). | This file already generates **all** dashboard HTML (`BuildTraceCard`, `BuildOutgoingRequestCard`, etc.). Keeping the waterfall here preserves the single-responsibility architecture instead of introducing a second rendering path. |
| CSS asset (via `EmbeddedResources.cs`) | Add `.waterfall-row`, `.wf-label`, `.wf-track`, `.wf-bar`, plus a `.wf-bar--error` modifier for failed calls. | Bars are pure CSS — a `<div>` with inline `left`/`width`, styled by these classes. No JS required. |
| `EmbeddedResources.cs` | **Likely no change** — only touched if CSS goes into a brand-new file instead of the existing stylesheet. | It's just a static cache of embedded text assets; only needs a new loader entry if a new file is introduced. |
| `ResourceLoader.cs` | **No change expected.** | Generic manifest-resource loader; unaffected unless the resource folder structure itself changes. |

**Blast radius:** 1 new method in `HtmlRenderer.cs`, a handful of new CSS rules, and a possible small property in `DebugOutgoingRequest.cs`. No middleware, no storage, no HTTP handler changes.

---

## 5. Step-by-Step Implementation Plan (Phase 1)

```mermaid
gantt
    title Phase 1 — 2-Day Implementation Timeline
    dateFormat  YYYY-MM-DD
    axisFormat  %d %b

    section Day 1
    Verify/add start-time field in DebugOutgoingRequest.cs   :d1a, 2026-07-03, 1d
    Write BuildWaterfallSection() in HtmlRenderer.cs          :d1b, after d1a, 1d

    section Day 2
    Wire into RenderDetailsPage                               :d2a, 2026-07-04, 1d
    Add CSS classes for bars                                   :d2b, after d2a, 1d
    Manual test on ShopApi sample project                      :d2c, after d2b, 1d
    Open focused PR                                             :d2d, after d2c, 1d
```

**Day 1**
1. Open `DebugOutgoingRequest.cs`, confirm start-time data exists at ms precision. Add a `StartedAt` field only if missing.
2. Write `BuildWaterfallSection` in `HtmlRenderer.cs`:
   - Guard clause: empty `OutgoingRequests` → return empty string (no empty section rendered).
   - Compute total timeline span from the parent request's duration.
   - Per call: compute `left%` / `width%`, clamp 0–100 to avoid edge-case rendering bugs.
   - Return one `<div class="waterfall-row">` per call — label, track, bar.

**Day 2**
3. Call `BuildWaterfallSection` from `RenderDetailsPage`, above the existing per-call cards.
4. Add CSS for `.waterfall-row` / `.wf-track` / `.wf-bar`, reusing the existing red/green status-code convention for failed calls.
5. Manually test on `ShopApi`: hit an endpoint with multiple outgoing calls, open `/debug`, confirm bars render proportionally and failed calls are visually distinct.
6. Open a small, focused PR — one render method, CSS additions, optional one model field. No JS, no new dependencies.

---

## 6. What's Explicitly Out of Scope for Phase 1

```mermaid
flowchart TD
    P1["Phase 1 — Ship Now"] --> P2["Phase 2 — Only after Phase 1 is accepted"]

    subgraph InScope["Phase 1"]
        S1["Static horizontal bars"]
        S2["left% / width% positioning"]
        S3["Color distinction for failed calls"]
    end

    subgraph OutOfScope["Deferred to Phase 2"]
        O1["Hover tooltips with exact timestamps"]
        O2["Zoom / pan on timeline"]
        O3["Time-axis ruler with ms gridlines"]
        O4["Any JS-driven interactivity"]
    end

    P1 -.-> InScope
    P2 -.-> OutOfScope

    style InScope fill:#1f6f43,color:#fff
    style OutOfScope fill:#5b5b5b,color:#fff
```

Proposing tooltips/zoom/gridlines now would inflate the PR and slow down review. They become natural, low-risk follow-ups once the maintainer has reviewed and accepted the simple version.
