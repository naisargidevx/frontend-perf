---
name: frontend-perf
description: Audit and fix frontend performance issues — Core Web Vitals (LCP, CLS, INP, FCP, TTFB, TTI), image optimization, bundle size, React rendering, CSS/JS bottlenecks. Also handles Fynd theme performance (transformImage, SSR safety, section code splitting). Trigger when the user mentions slow page loads, Lighthouse scores, LCP/CLS/INP/FID/TTI, image optimization, lazy loading, bundle splitting, render performance, Fynd theme, "audit my project", or "my site is slow".
---

# Frontend Performance Helper

Scan the project, surface all performance issues with severity, then fix them one at a time — only after user approval for major/critical changes.

> **Fynd theme?** Read `references/fynd-theme.md` FIRST. It defines what is protected and what is safe to touch.

---

## Step 0 — Existing Audit Check (ALWAYS ask this first, before anything else)

**Before scanning any file or asking any other question**, ask the user these two questions together:

> 1. "Do you already have an audit report? (PageSpeed Insights, GTmetrix, Chrome Lighthouse, WebPageTest, or any other tool)"
> 2. "Which page/route is most important to your business? (e.g. homepage, product page, checkout) — I'll prioritize fixes for that route first."

Then branch based on the answer:

### If YES — they have an audit report

Ask them to share it (paste scores, screenshot description, or raw data). Extract:

| Field | What to look for | Why it matters |
|-------|-----------------|----------------|
| Overall Perf score | 0–100 | Baseline to beat |
| LCP | Value in seconds + which element (image/text/video) | Biggest score lever |
| CLS | Value (threshold 0.1) | Layout stability |
| INP / FID | Value in ms (threshold 200ms) | Interaction responsiveness |
| FCP | Value in seconds | First visible content |
| TTFB | Value in ms (flag if > 800ms) | Server/CDN health |
| TBT | Value in ms (lab proxy for INP) | Long task debt |
| Speed Index | Value | Visual load progression |
| Flagged Opportunities | Listed items + estimated savings | Prioritise biggest savings first |
| Flagged Diagnostics | Listed items | Confirms root causes |

> **Field vs Lab — critical distinction:**
> - **Field data** (CrUX in PSI, labelled "real users") = what actual visitors experience on their devices/networks. This is what Google uses for ranking signals. Trust these numbers for business decisions.
> - **Lab data** (Lighthouse, labelled "simulated") = controlled synthetic test. Good for catching regressions and debugging. Can differ significantly from field. If field LCP = 4.5s but lab LCP = 2.8s, the field number is the real problem to solve.

Once you have the data:
1. **Summarize what the audit found** in a short table with field vs lab clearly separated
2. **Identify the single biggest score blocker** — usually the metric furthest from "Good" threshold
3. **Map each flagged Opportunity** to the relevant Check ID from the scan table below (IMG-1, LCP-1, BUNDLE-2, etc.)
4. **Prioritize those specific checks first** when scanning the codebase in Step 1
5. Tell the user: "Your biggest score blocker is [metric] at [value] — I'll focus the scan on that first."

### If NO — no existing audit

Acknowledge and move straight to Step 1 code scan. Optionally suggest:
> "No problem — I'll scan the code directly. After we're done, you can run a free PageSpeed Insights check at https://pagespeed.web.dev to validate the improvements."

---

## Mode 1 — Full Project Audit (default when no specific issue is given)

Run this when the user says "audit", "check my project", "what's slow", or similar.

### Step 1: Scan the project

Use Glob + Grep to scan these folders: `theme/sections/`, `theme/components/`, `theme/pages/`, `theme/custom-templates/`, `theme/helper/`

Run ALL of these checks in parallel:

| Check ID | What to grep for | Issue if found |
|----------|-----------------|----------------|
| IMG-1 | `<img` without `width=` AND `height=` | CLS — missing dimensions |
| IMG-2 | `<img` with `loading="lazy"` on hero/first image | LCP — lazy on LCP image |
| IMG-3 | `<img` using raw `.src` or string URL, no `transformImage` | LCP/Images — no CDN resize |
| IMG-4 | `<img` without `loading=` attribute (below-fold) | TTI — no lazy load |
| SSR-1 | `window.` or `document.` not inside `typeof window !== 'undefined'` or `useEffect` | SSR crash |
| SSR-2 | `localStorage` or `sessionStorage` at module/component top level | SSR crash |
| REACT-1 | `.map(` in JSX without `key=` prop | React perf warning |
| REACT-2 | Large component files (> 200 lines) with no `useMemo`/`useCallback`/`memo` | INP — re-render risk |
| REACT-3 | `useEffect` with no dependency array `[]` | INP — runs on every render |
| BUNDLE-1 | `import * as` or default import from lodash/moment/date-fns | Bundle — no tree shaking |
| BUNDLE-2 | Sections NOT using `React.lazy` / dynamic import | TTI — no code splitting |
| CLS-1 | CSS animations using `top`/`left`/`width`/`height`/`margin` | CLS — non-composited animation |
| LCP-1 | Hero/banner image without `fetchpriority="high"` | LCP — no priority hint |
| FONT-1 | `@font-face` without `font-display:` property | FCP — FOIT/FOUT risk |
| THIRD-1 | `<script src=` in `<head>` without `async` or `defer`; or third-party tag manager loading sync | INP/LCP — render-blocking third party |
| REACT-4 | Heavy filter/search/sort state update (`.filter(`, `.sort(`, `.reduce(`) without `startTransition` | INP — blocks main thread on input |
| NEXT-1 | `next/image` `<Image>` on first/hero slot without `priority` prop | LCP — missing preload for LCP image |
| NEXT-2 | `next/script` `<Script>` without `strategy=` prop (or strategy="beforeInteractive" on non-critical script) | INP/LCP — third-party blocks critical path |
| PRECONNECT-1 | Third-party origins (fonts.googleapis.com, fonts.gstatic.com, CDN host, analytics, API domain) referenced in code/CSS but no `<link rel="preconnect">` in `<head>` | FCP/LCP — DNS + TLS handshake cost per origin adds 50–200ms |
| IMG-5 | `<img>` or `<Image>` on variable-width slots (product cards, banners) with no `srcset`/`sizes` | LCP/Images — browser downloads full-size image even on mobile |
| IMG-6 | `<img>` without `decoding="async"` on non-LCP images | LCP — synchronous image decode can delay LCP paint |
| COMPRESS-1 | `vite.config` without `vite-plugin-compression`; `next.config` without `compress: true`; no `.gz`/`.br` assets in dist | LCP — uncompressed JS/CSS inflates transfer size (Brotli saves ~20–25% vs gzip) |
| CACHE-1 | `vite.config`/`webpack.config` with no `manualChunks` separating vendor libs from app code | TTI/repeat-load — React + vendor bundle re-downloaded on every app deploy |
| FONT-2 | `@font-face` with `font-display: block` or `font-display: auto` | FCP — invisible text during font load (FOIT); swap/optional is better |
| RUM-1 | No `web-vitals` package in `package.json`; no `onLCP`/`onINP`/`onCLS` calls anywhere in codebase | No field data — can't measure real-user CWV or prove improvements ship |
| REACT-5 | `useEffect(() => { fetch(...)  }, [])` chains where one fetch triggers another (waterfall) instead of parallel fetches or RSC | LCP — sequential fetches delay content; each waterfall step adds one RTT |
| SERVER-1 | SSR route handlers or API routes with no `Server-Timing` header and no `Cache-Control` on HTML responses | TTFB — slow server is invisible; can't distinguish db vs app vs CDN miss |

### Step 2: Present the issue report

After scanning, output a structured report — DO NOT apply any fix yet.

```
## Performance Audit Report — [project name]

### Summary
| Severity  | Count |
|-----------|-------|
| 🔴 Critical | N    |
| 🟠 Major    | N    |
| 🟡 Minor    | N    |
| Total       | N    |

---

### 🔴 Critical Issues  (break functionality or crash SSR)
| # | Check | File | Line | Issue | Metric |
|---|-------|------|------|-------|--------|
| 1 | SSR-1 | sections/HeroBanner.jsx | 12 | `window.innerWidth` used outside SSR guard | SSR crash |

### 🟠 Major Issues  (significant metric impact — ask permission before fixing)
| # | Check | File | Line | Issue | Metric | KPI Risk | ICE Priority |
|---|-------|------|------|-------|--------|---------|-------------|
| 1 | LCP-1 | sections/HeroBanner.jsx | 5 | Hero image missing fetchpriority="high" | LCP −600ms | Conversion ↓ | **High** (small change, big gain) |
| 2 | IMG-3 | sections/ProductCard.jsx | 34 | Raw image URL — no transformImage | LCP −300ms | Bounce ↑ | **High** (medium effort, clear win) |
| 3 | IMG-1 | components/Banner.jsx | 8 | img missing width/height | CLS | Misclicks, rage clicks | **Medium** |

> **ICE scoring guide:** Impact (metric distance from threshold × user volume on route) × Confidence (evidence strength: trace/waterfall = high, code inspection only = medium) ÷ Effort (S=high, L=low). Fix highest ICE first.

### 🟡 Minor Issues  (low risk, easy wins)
| # | Check | File | Line | Issue | Metric | KPI Risk |
|---|-------|------|------|-------|--------|---------|
| 1 | IMG-4 | sections/BlogCard.jsx | 22 | Below-fold img missing loading="lazy" | TTI | Slower perceived load |
| 2 | REACT-1 | components/List.jsx | 45 | .map() missing key= prop | React warning | None direct |
```

After the report, ask:
> "Found N issues. Should I start fixing them? I'll go through Critical first, then Major — asking your permission before each one. Minor issues I'll batch at the end."

---

## Step 2b — Metric Subpart Triage (run when LCP or INP is flagged as Major/Critical)

Before jumping to fixes, diagnose **which subpart** is the actual bottleneck. This prevents applying the wrong fix (e.g., optimising the image when TTFB is the real problem).

### LCP Subpart Triage

LCP = TTFB + Resource Load Delay + Render Delay. Ask the user or infer from audit data:

| Subpart | Signal | Primary Fix |
|---------|--------|-------------|
| **TTFB > 800ms** | PSI "Server response time" opportunity flagged; PSI field TTFB in red | CDN caching, reduce server render time, SSG/ISR where possible |
| **Resource Load Delay** (image loads late) | Waterfall shows hero image starts late; no preload in Lighthouse | Add `fetchpriority="high"` + `<link rel="preload">` for LCP image; Next.js: add `priority` to `<Image>` |
| **Render Delay** (image downloaded but paint is late) | Lighthouse flags render-blocking CSS/JS; large JS bundles before paint | Defer non-critical JS (`defer`/`async`); move critical CSS inline; reduce TBT |

**Decision rule:** Fix in order — TTFB first (no point optimising images if server is slow), then resource load, then render delay.

### INP Phase Triage

INP = Input Delay + Processing Time + Presentation Delay. Diagnose by asking: "What interaction triggers the lag? (click, typing, tap)"

| Phase | Signal | Primary Fix |
|-------|--------|-------------|
| **Input Delay** (long tasks block the event) | Lighthouse TBT high; third-party scripts in `<head>`; large JS bundles | Move third-party scripts to `async`/`defer`; use `next/script strategy="lazyOnload"`; code-split bundles |
| **Processing Time** (event handler is slow) | DevTools shows long "Event: click" task; React re-render on every keystroke | Wrap expensive state updates with `startTransition`; memoize with `useMemo`/`useCallback`; isolate state |
| **Presentation Delay** (render after handler is slow) | Lighthouse flags layout/paint after interaction | Avoid layout thrash (batch DOM reads before writes); use CSS `transform`/`opacity` for animations |

---

## Step 3: Fix Flow — One Issue at a Time

Work through issues in order: **Critical → Major → Minor**

For **every Critical and Major issue**, before touching any file:

1. Show the current code (read the file)
2. Show exactly what the fix will change (before/after diff)
3. State which metric it improves and by how much
4. Ask: **"Apply this fix? (yes / skip / stop)"**
5. Only edit the file after explicit "yes"
6. Confirm after applying: "✓ Fixed — [file:line]"
7. Show the **verification step** — what to check to confirm the fix worked:

```
Verify:
  Lab:   Re-run Lighthouse (incognito, 3 runs, average). Expect LCP to drop from ~4.2s → ~2.8s.
         Check "Largest Contentful Paint" element — confirm it now loads with priority.
         Check Network waterfall — hero image request should start within first 500ms.
  Field: After deploying, check PageSpeed Insights in 24–48h for CrUX update.
         Target: LCP field p75 moves from Poor (>4s) toward Needs Improvement (<4s).
         If RUM is instrumented: watch p75 LCP for the route cohort shift within 48h.
```

> **TTFB blocker?** If lab TTFB > 800ms, the above won't fully land. Before fixing images/scripts, first add `Server-Timing` to identify whether the delay is db, app, or CDN miss. See `references/server-timing.md`.

For **Minor issues**, after all Critical/Major are done:
- Ask once: "Apply all N minor fixes? They're low-risk (lazy load attributes, key props, etc.)"
- Apply all if approved, skip all if not

### Fix request format (shown to user before each fix)

```
---
Fix 2 of 7 — 🟠 Major

File:    theme/sections/ProductCard.jsx  line 34
Metric:  LCP  (expected ~300ms improvement)
Issue:   Raw image URL used — browser gets no CDN resize or WebP conversion

Before:
  <img src={product.media[0].url} alt={product.name} />

After:
  <img
    src={transformImage(product.media[0].url, { width: 600, format: 'webp' })}
    srcSet={`
      ${transformImage(product.media[0].url, { width: 400, format: 'webp' })} 400w,
      ${transformImage(product.media[0].url, { width: 600, format: 'webp' })} 600w
    `}
    sizes="(max-width: 480px) 100vw, 300px"
    loading="lazy"
    width="600"
    height="600"
    alt={product.name}
  />

Apply this fix? yes / skip / stop
---
```

---

## Mode 2 — Targeted Fix (when user describes a specific issue)

Still run **Step 0** first — the existing audit report may contain the exact metric score and affected resource, saving a full code scan.

Skip the full audit. Go straight to:
1. Read `references/fynd-theme.md` if Fynd context
2. Read the relevant metric reference file
3. Read the affected file(s)
4. Present the fix with before/after
5. Ask permission, then apply

---

## Metric → Reference Map

| Symptom / Metric | Reference File |
|------------------|----------------|
| Fynd theme, transformImage, SSR crash, folder rules | `references/fynd-theme.md` |
| LCP > 2.5s, hero image, TTFB, preconnect, srcset | `references/lcp.md` |
| Layout shifts, CLS > 0.1 | `references/cls.md` |
| INP > 200ms, click lag, janky UI | `references/inp.md` |
| TTI high, long tasks, main thread | `references/tti.md` |
| Images, format, srcset, lazy load | `references/images.md` |
| Bundle size, tree shaking, code split, vendor chunk, compression | `references/bundle.md` |
| React re-renders, memoization, lists | `references/react-perf.md` |
| FCP, render-blocking, fonts, preconnect, compression | `references/fcp.md` |
| RUM, web-vitals, field data, route segmentation, attribution | `references/rum.md` |
| Server-Timing, TTFB diagnosis, caching headers | `references/server-timing.md` |

---

## Severity Definitions

| Severity | Meaning | Fix behavior |
|----------|---------|-------------|
| 🔴 Critical | Crashes SSR, breaks routing, breaks editor | Ask permission, fix first |
| 🟠 Major | Measurable metric regression (LCP/CLS/INP > threshold) | Ask permission per issue |
| 🟡 Minor | Small improvements, no risk of breakage | Batch ask at end |

---

## End-of-Session Score Projection

After all fixes are applied, show a summary table with projected impact:

```
## Fix Summary

| # | Issue | Check | Status | Metric | Expected Gain |
|---|-------|-------|--------|--------|---------------|
| 1 | Hero image missing priority | LCP-1 | ✅ Fixed | LCP | ~600ms faster |
| 2 | No code splitting on sections | BUNDLE-2 | ✅ Fixed | TTI/TBT | ~200ms TBT reduction |
| 3 | Raw image URLs | IMG-3 | ✅ Fixed | LCP | ~300ms faster |
| 4 | Sync third-party script in <head> | THIRD-1 | ⏭ Skipped | INP | — |

Estimated Lighthouse score delta: +12–18 points (from ~52 → ~65–70)
Next step: Re-run PageSpeed Insights to confirm. Field CrUX data updates on a ~28-day rolling window.
```

> **Important:** Always remind the user that lab (Lighthouse) improvements show immediately; field (CrUX/PSI real-user data) takes days to weeks to reflect after deploy. Both matter.

---

## CI Gating (offer this at end of session if user has CI/CD)

After fixes, offer to generate an LHCI config to prevent regressions:

```json
{
  "ci": {
    "collect": {
      "url": ["https://your-staging-url.com/", "https://your-staging-url.com/product/123"],
      "numberOfRuns": 3,
      "settings": { "preset": "mobile", "onlyCategories": ["performance"] }
    },
    "assert": {
      "assertions": {
        "categories:performance":        ["error", { "minScore": 0.75 }],
        "largest-contentful-paint":      ["error", { "maxNumericValue": 3000 }],
        "cumulative-layout-shift":       ["error", { "maxNumericValue": 0.1 }],
        "total-blocking-time":           ["error", { "maxNumericValue": 300 }],
        "unused-javascript":             ["warn",  { "maxNumericValue": 120000 }]
      }
    }
  }
}
```

Ask: "Would you like me to add this as `.lighthouserc.json` to your project to catch future regressions in CI?"

---

## RUM Setup (offer this at end of session if RUM-1 was flagged or no web-vitals found)

After fixes, offer to add real-user monitoring so improvements are measurable in field data:

> "Would you like me to add a `web-vitals` snippet to track real-user LCP/INP/CLS? This sends data to your analytics so you can see the improvement in field data after deploy — not just Lighthouse."

If yes, add to the app entry point (or analytics module):

```js
import { onCLS, onINP, onLCP, onFCP, onTTFB } from 'web-vitals/attribution';

function sendToAnalytics({ name, value, rating, attribution }) {
  // Replace with your analytics endpoint / gtag / dataLayer
  console.log({ metric: name, value: Math.round(value), rating, attribution });
}

onLCP(sendToAnalytics);
onINP(sendToAnalytics);
onCLS(sendToAnalytics);
onFCP(sendToAnalytics);
onTTFB(sendToAnalytics);
```

> **Why `/attribution`?** The attribution build tells you *which element* caused LCP and *which interaction phase* caused INP — makes debugging field regressions much faster. See `references/rum.md` for full setup including GA4 integration and SPA route segmentation.

---

## Fynd Safety Gate

Before applying ANY fix in a Fynd theme, verify:
- [ ] Not renaming/moving `index.jsx` or any file in `pages/`
- [ ] Not changing `config/settings_data.json` or `settings_schema.json` structure
- [ ] Section still exports both `Component` and `settings`
- [ ] No `window`/`document`/`localStorage` added outside SSR guard
- [ ] Using `transformImage` for all image src values

If a fix would violate any of these → **mark as blocked, explain why, do not apply**.

---

## Key Principles

- **Always audit before fixing** — never edit files without showing the issue list first
- **Always ask permission** for Critical and Major fixes — never auto-apply
- **Show exact before/after** for every fix — no vague descriptions
- **Read the file first** before showing a fix — never guess the current code
- **Fynd context: read `fynd-theme.md` first** — folder contract is non-negotiable
- **One fix at a time** — don't batch Critical/Major changes
- Minor fixes can be batched only after user approves all Critical/Major work
- After all fixes: show a summary table of what was fixed, skipped, and remaining
