---
name: frontend-perf
description: Audit and fix frontend performance issues — Core Web Vitals (LCP, CLS, INP, FCP, TTFB, TTI), image optimization, bundle size, React rendering, CSS/JS bottlenecks. Also handles Fynd theme performance (transformImage, SSR safety, section code splitting). Trigger when the user mentions slow page loads, Lighthouse scores, LCP/CLS/INP/FID/TTI, image optimization, lazy loading, bundle splitting, render performance, Fynd theme, "audit my project", or "my site is slow".
---

# Frontend Performance Helper

Scan the project, surface all performance issues with severity, then fix them one at a time — only after user approval for major/critical changes.

> **Fynd theme?** Read `references/fynd-theme.md` FIRST. It defines what is protected and what is safe to touch.

---

## Step 0 — Existing Audit Check (ALWAYS ask this first, before anything else)

**Before scanning any file or asking any other question**, ask the user these three questions together:

> 1. "Do you already have an audit report? (PageSpeed Insights, GTmetrix, Chrome Lighthouse, WebPageTest, or any other tool)"
> 2. "Which page/route is most important to your business? (homepage / PLP / PDP / checkout) — I'll prioritize fixes for that route first."
> 3. "Do you want me to check a specific file or section? (e.g. `HeroBanner.jsx`, `product-listing.jsx`, `useProductFilter.js`) — if yes, I'll scan that file directly against the Fynd folder structure."

If the user names a specific file in Q3, skip the full audit and jump to **Mode 3 — Specific File Check**.

Then branch based on the answer:

### If YES — they have an audit report (any tool)

**Step A: Detect the tool and data format** from what the user shares:

| What the user shares | Tool | Fetch method |
|---------------------|------|-------------|
| `pagespeed.web.dev/report?url=…` | PageSpeed Insights | Auto-fetch via PSI API |
| Any plain site URL (`https://mystore.com/`) | PageSpeed Insights | Auto-fetch via PSI API |
| `gtmetrix.com/reports/…` | GTmetrix | Can't auto-fetch — ask to paste key data |
| `webpagetest.org/result/…` | WebPageTest | Auto-fetch via WPT JSON endpoint |
| Pasted JSON with `lighthouseResult` key | Lighthouse JSON export | Parse directly |
| Pasted JSON with `data.median.firstView` | WebPageTest JSON export | Parse directly |
| Screenshot / text / pasted scores | Any tool | Manual extraction |

---

### Tool: PageSpeed Insights (PSI)

Triggers when: user shares `pagespeed.web.dev` URL, or any plain site URL.

**Fetch:** Call the PSI API using WebFetch — no API key needed:
```
https://www.googleapis.com/pagespeedonline/v5/runPagespeed?url={ENCODED_URL}&strategy=mobile&category=performance
```
Ask mobile (default) or desktop before fetching.

**Parse `lighthouseResult`:**
| JSON path | Metric |
|-----------|--------|
| `categories.performance.score` × 100 | Overall score |
| `audits['largest-contentful-paint'].displayValue` | LCP |
| `audits['largest-contentful-paint'].details.items` | LCP element |
| `audits['cumulative-layout-shift'].displayValue` | CLS |
| `audits['total-blocking-time'].displayValue` | TBT (INP proxy) |
| `audits['first-contentful-paint'].displayValue` | FCP |
| `audits['server-response-time'].displayValue` | TTFB |
| `audits['speed-index'].displayValue` | Speed Index |
| `audits['render-blocking-resources'].details.items` | Render-blocking resources |
| `audits['uses-optimized-images'].details.items` | Unoptimized images |
| `audits['uses-responsive-images'].details.items` | Missing srcset images |
| `audits['unused-javascript'].details.items` | Unused JS bytes |
| `audits['unused-css-rules'].details.items` | Unused CSS bytes |
| `audits['uses-long-cache-ttl'].details.items` | Cache TTL issues |
| `audits['lcp-lazy-loaded'].displayValue` | LCP image lazy-loaded |
| `audits['efficient-animated-content'].details.items` | GIFs to convert |

**Parse `loadingExperience` (field/CrUX — real users):**
| JSON path | Metric |
|-----------|--------|
| `metrics.LARGEST_CONTENTFUL_PAINT_MS.percentile` | LCP p75 |
| `metrics.CUMULATIVE_LAYOUT_SHIFT_SCORE.percentile` | CLS p75 |
| `metrics.INTERACTION_TO_NEXT_PAINT.percentile` | INP p75 |
| `metrics.FIRST_CONTENTFUL_PAINT_MS.percentile` | FCP p75 |

> Field data may be absent for low-traffic sites — only lab data will be available in that case.

---

### Tool: GTmetrix

Triggers when: user shares `gtmetrix.com/reports/` URL.

**Cannot auto-fetch** — GTmetrix reports are JS-rendered and the API is paid. Ask the user:
> "GTmetrix reports can't be fetched automatically. Please paste or describe: GTmetrix Grade, LCP, TBT, CLS, TTFB, and the Top Issues list with their Impact ratings."

**Extract from pasted GTmetrix data:**
| GTmetrix field | Maps to |
|---------------|---------|
| GTmetrix Grade (A–F) | Overall health proxy |
| Performance % | Lighthouse score equivalent |
| Structure % | Best practices score |
| LCP (ms or s) | LCP metric |
| TBT (ms) | TBT / INP proxy |
| CLS | CLS metric |
| TTFB (ms) | TTFB metric |
| Total Load Time | Load time |
| Total Page Size | Bundle size proxy |
| Top Issues with Impact: High/Medium/Low | Map to Check IDs below |

**GTmetrix issue → Check ID mapping:**
| GTmetrix Issue | Check ID |
|---------------|---------|
| "Serve scaled images" / "Specify image dimensions" | IMG-1, IMG-5 |
| "Use efficient image format" / "Optimize images" | IMG-3 (transformImage) |
| "Defer parsing of JavaScript" | BUNDLE-2, THIRD-1 |
| "Minify / Compress JS or CSS" | COMPRESS-1 |
| "Leverage browser caching" | CACHE-1 |
| "Avoid render-blocking resources" | THIRD-1, BUNDLE-2 |
| "Reduce initial server response time" | SERVER-1 |
| "Preload key requests" | LCP-1, PRECONNECT-1 |
| "Avoid enormous network payloads" | BUNDLE-1, COMPRESS-1 |
| "Avoid an excessive DOM size" | REACT-2, BUNDLE-2 |
| "Eliminate layout shifts" | CLS-1, IMG-1 |
| "Use passive listeners" | REACT-4 |

---

### Tool: WebPageTest (WPT)

Triggers when: user shares `webpagetest.org/result/` URL.

**Fetch:** Extract the test ID from the URL and try the JSON endpoint using WebFetch:
```
https://www.webpagetest.org/result/{TEST_ID}/?f=json
```
If the JSON endpoint is inaccessible, ask the user to export/paste the JSON or share the key metrics.

**Parse `data.median.firstView`:**
| JSON path | Metric |
|-----------|--------|
| `TTFB` | TTFB in ms |
| `render` | Start Render in ms |
| `SpeedIndex` | Speed Index |
| `loadTime` | Total load time |
| `TotalBlockingTime` | TBT (INP proxy) |
| `CumulativeLayoutShift` | CLS |
| `chromeUserTiming.LargestContentfulPaint` | LCP in ms |
| `chromeUserTiming.firstContentfulPaint` | FCP in ms |
| `requests` (array) | Individual resource waterfall |
| `breakdown` | JS/CSS/image byte breakdown |

Also check `data.lighthouse` for Lighthouse audit data (same fields as PSI).

**WPT-specific signals → Check IDs:**
| WPT Signal | Check ID |
|-----------|---------|
| TTFB > 800ms | SERVER-1 |
| Render-blocking requests in waterfall (early red bars) | THIRD-1, BUNDLE-2 |
| Large image files in breakdown | IMG-3, IMG-5 |
| JS > 500KB in breakdown | BUNDLE-1, BUNDLE-2, COMPRESS-1 |
| No CDN / direct origin requests for assets | CACHE-1, PRECONNECT-1 |
| LCP element loads after 2.5s | LCP-1, IMG-2 |

---

### Tool: Lighthouse JSON export (from Chrome DevTools or CLI)

Triggers when: user pastes or shares a JSON file containing `lighthouseResult` or `categories.performance`.

**Parse identically to PSI** — Lighthouse is the same engine. Use the same JSON paths from the PSI section above. There is no `loadingExperience` field (no CrUX data in local Lighthouse runs).

---

### Tool: Any other tool (DebugBear, Calibre, SpeedVitals, Treo, etc.) or pasted text/screenshot

Ask the user to share whatever they have (scores, opportunities list, screenshot description). Extract these fields from what they provide:

| Field | Threshold |
|-------|-----------|
| Overall performance score | 0–100 |
| LCP | Good ≤ 2.5s |
| CLS | Good ≤ 0.1 |
| INP or FID | Good ≤ 200ms |
| TBT | Good ≤ 200ms |
| FCP | Good ≤ 1.8s |
| TTFB | Flag if > 800ms |
| List of flagged issues / recommendations | With impact/priority |

Map flagged issues to Check IDs using the opportunity → Check ID table below, then proceed to the scan.

---

### Universal: Opportunity → Check ID mapping (applies to all tools)

| Opportunity / Issue (any tool wording) | Check IDs |
|----------------------------------------|-----------|
| Image size / format / next-gen / WebP / optimize images | IMG-3, IMG-5 |
| Image dimensions missing / width+height | IMG-1 |
| Lazy load / defer offscreen images | IMG-4 |
| LCP image lazy loaded | IMG-2 |
| Preload LCP image / fetchpriority | LCP-1 |
| Render-blocking JS or CSS | THIRD-1, BUNDLE-2 |
| Unused JS / reduce JS payload | BUNDLE-1, BUNDLE-2 |
| Unused CSS | COMPRESS-1 |
| Enable compression / Brotli / gzip | COMPRESS-1 |
| Cache policy / browser caching / long cache TTL | CACHE-1 |
| Layout shifts / CLS | CLS-1, IMG-1 |
| JS execution time / main thread / long tasks | REACT-2, REACT-4 |
| DOM size / too many nodes | REACT-2, BUNDLE-2 |
| Preconnect to origins / DNS lookup | PRECONNECT-1 |
| Chained requests / request waterfall | REACT-5, HOOK-2 |
| Server response time / TTFB slow | SERVER-1 |
| Font display / FOIT / FOUT | FONT-1, FONT-2 |
| Passive event listeners | REACT-4 |
| Third-party scripts / tag managers | THIRD-1 |

---

### Present the parsed report (same format for all tools)

After extracting data from any tool, output this summary — always separating field (real users) from lab (simulated):

```
## [Tool name] Report — [url] — [mobile/desktop]

### Lab (simulated / tool-measured)
| Metric | Value | Status |
|--------|-------|--------|
| Performance Score | 54/100 | 🔴 Poor |
| LCP | 4.2s | 🔴 Poor (>2.5s) |
| CLS | 0.08 | 🟢 Good |
| TBT | 620ms | 🔴 Poor (>200ms) |
| FCP | 2.1s | 🟠 Needs improvement |
| TTFB | 310ms | 🟢 Good |

### Field (Real users — if available)
| Metric | p75 Value | Status |
|--------|-----------|--------|
| LCP | 5.8s | 🔴 Poor |
| INP | 380ms | 🔴 Poor |
| CLS | 0.12 | 🟠 Needs improvement |

### Biggest Blocker: LCP at 5.8s (field)
Tool flagged: "Preload LCP image", "Serve images in next-gen formats", "Defer offscreen images"
→ Prioritizing checks: LCP-1, IMG-2, IMG-3, IMG-5 — scanning Tier 1 files first.
```

> **Field vs Lab:** Field data = real users, what Google uses for ranking. Lab = simulated. If they differ, trust field. Lab is for debugging root cause.

Then proceed to Step 1 scan, running the mapped Check IDs first before the full check table.

---

### If NO — no existing audit

Acknowledge and move straight to Step 1 code scan. Offer:
> "No problem — I'll scan the code directly. If you share your site URL I can pull a live PageSpeed Insights report automatically, or you can run it free at pagespeed.web.dev after we're done."

---

## Mode 1 — Full Project Audit (default when no specific issue is given)

Run this when the user says "audit", "check my project", "what's slow", or similar.

### Fynd Page & Section Priority

Scan in this order — highest business impact first:

**Tier 1 — Scan First (critical user journeys)**
| Page / Section type | Why it matters |
|---------------------|---------------|
| `pages/home.jsx` or `index.jsx` | First impression, LCP baseline for all users |
| `pages/product-listing.jsx` (PLP) | High traffic, filter/sort INP, product card CLS |
| `pages/product-description.jsx` (PDP) | Conversion page, image LCP, add-to-cart INP |
| Sections: sliders / carousels | Heavy JS, layout shift on load, lazy image issues |
| Sections: product cards (grid/list) | Repeated images → CLS, srcset, transformImage |
| Sections: hero banners | #1 LCP element on most pages |

**Tier 2 — Scan Second**
| Area | Why |
|------|-----|
| `helper/` and `hooks/` (custom hooks) | SSR safety, fetch waterfalls, re-render triggers |
| `sections/` (remaining non-Tier-1 sections) | Code splitting, lazy load |
| `components/` (shared UI) | Re-render chains from shared state |

**Tier 3 — Scan Last**
| Area | Why |
|------|-----|
| `styles/` | CLS from animations, font-display |
| `config/` / `assets/` | Bundle config, compression, cache settings |

> **Rule:** Always complete Tier 1 before moving to Tier 2. Report Tier 1 issues separately so the user can see the highest-impact fixes first.

### Step 1: Scan the project

Use Glob + Grep. **Scan Tier 1 files first, then Tier 2, then Tier 3.** Within each tier, run all checks in parallel:

**Tier 1 files to target:**
- `theme/pages/home.jsx`, `theme/pages/index.jsx`
- `theme/pages/product-listing.jsx`
- `theme/pages/product-description.jsx`
- `theme/sections/*[Ss]lider*.jsx`, `theme/sections/*[Cc]arousel*.jsx`
- `theme/sections/*[Pp]roduct[Cc]ard*.jsx`, `theme/sections/*[Pp]roduct[Gg]rid*.jsx`, `theme/sections/*[Pp]roduct[Ll]ist*.jsx`
- `theme/sections/*[Bb]anner*.jsx`, `theme/sections/*[Hh]ero*.jsx`

**Tier 2 files to target:**
- `theme/helper/*.js`, `theme/helper/*.jsx`, `theme/hooks/*.js`, `theme/hooks/*.jsx`
- Remaining files in `theme/sections/` not matched above
- `theme/components/`

**Tier 3:**
- `theme/styles/`, `vite.config.*`, `theme/config/`

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
| HOOK-1 | Custom hooks in `helper/`/`hooks/` using `window`/`document`/`localStorage` without `typeof window !== 'undefined'` guard | SSR crash — hooks used in sections run server-side |
| HOOK-2 | Custom hooks with `useEffect` fetch chains (one fetch triggers another via state) without `Promise.all` or parallel pattern | LCP — waterfall fetches in hooks delay render just like component-level waterfalls |
| HOOK-3 | Custom hooks returning new object/array literal on every call (e.g. `return { data, loading }` without `useMemo`) used as dep in parent `useEffect` | INP — causes infinite re-render loops or stale-closure bugs in parent components |
| HOOK-4 | Custom hooks with heavy `.filter()`/`.sort()`/`.reduce()` on large arrays without `useMemo` inside the hook | INP — computation runs on every parent render, not just when inputs change |

### Step 2: Present the issue report

After scanning, output a structured report — DO NOT apply any fix yet. **Group issues by Tier first, then by severity within each tier.**

```
## Performance Audit Report — [project name]

### Summary
| Severity  | Tier 1 (Home/PLP/PDP + key sections) | Tier 2 (Hooks + other sections) | Tier 3 (Config/styles) |
|-----------|--------------------------------------|--------------------------------|------------------------|
| 🔴 Critical | N | N | N |
| 🟠 Major    | N | N | N |
| 🟡 Minor    | N | N | N |

---

### Tier 1 — Home / PLP / PDP / Sliders / Product Cards / Banners

#### 🔴 Critical Issues
| # | Check | File | Line | Issue | Metric |
|---|-------|------|------|-------|--------|
| 1 | SSR-1 | sections/HeroBanner.jsx | 12 | `window.innerWidth` used outside SSR guard | SSR crash |

#### 🟠 Major Issues
| # | Check | File | Line | Issue | Metric | KPI Risk | ICE Priority |
|---|-------|------|------|-------|--------|---------|-------------|
| 1 | LCP-1 | sections/HeroBanner.jsx | 5 | Hero image missing fetchpriority="high" | LCP −600ms | Conversion ↓ | **High** |
| 2 | IMG-3 | sections/ProductCard.jsx | 34 | Raw image URL — no transformImage | LCP −300ms | Bounce ↑ | **High** |

> **ICE scoring guide:** Impact (metric distance from threshold × user volume on route) × Confidence (evidence strength: trace/waterfall = high, code inspection only = medium) ÷ Effort (S=high, L=low). Fix highest ICE first.

#### 🟡 Minor Issues
| # | Check | File | Line | Issue | Metric |
|---|-------|------|------|-------|--------|
| 1 | IMG-4 | sections/ProductSlider.jsx | 22 | Below-fold img missing loading="lazy" | TTI |

---

### Tier 2 — Hooks / Other Sections / Components

#### 🔴 Critical
| # | Check | File | Line | Issue | Metric |
|---|-------|------|------|-------|--------|
| 1 | HOOK-1 | helper/useWindowSize.js | 3 | `window.innerWidth` in hook without SSR guard | SSR crash |

#### 🟠 Major / 🟡 Minor
(same table structure as Tier 1)

---

### Tier 3 — Config / Styles
(issues here if any)
```

After the report, ask:
> "Found N issues across [Tier 1: X, Tier 2: Y, Tier 3: Z]. Should I start fixing them? I'll work through Tier 1 first — Critical then Major. I'll ask your permission before each one. Minor issues I'll batch at the end."

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

## Mode 3 — Specific File Check (when user names a file in Step 0 Q3)

Use this when the user gives a filename like `HeroBanner.jsx`, `useProductFilter.js`, `product-listing.jsx`.

### Step A: Resolve the file in the Fynd folder structure

Map the filename to its likely Fynd location using this lookup:

| Filename pattern | Likely Fynd path | Protected? |
|-----------------|-----------------|------------|
| `home.jsx`, `index.jsx` | `theme/pages/home.jsx` or `theme/pages/index.jsx` | pages — do NOT rename |
| `product-listing.jsx` | `theme/pages/product-listing.jsx` | pages — do NOT rename |
| `product-description.jsx` | `theme/pages/product-description.jsx` | pages — do NOT rename |
| `*Slider*.jsx`, `*Carousel*.jsx` | `theme/sections/` | Safe to optimize |
| `*ProductCard*.jsx`, `*ProductGrid*.jsx` | `theme/sections/` | Safe to optimize |
| `*Banner*.jsx`, `*Hero*.jsx` | `theme/sections/` | Safe to optimize |
| `use*.js` / `use*.jsx` | `theme/helper/` or `theme/hooks/` | Safe — apply HOOK checks |
| Anything in `pages/` | `theme/pages/[filename]` | Do NOT rename or restructure |
| Anything in `components/` | `theme/components/[filename]` | Safe to optimize |

Tell the user: "Found `[filename]` at `[resolved path]`. This is a [Tier 1 / Tier 2] file — [protected / safe to optimize]."

### Step B: Read the file and run all relevant checks

- Run the full check table (IMG-*, SSR-*, REACT-*, BUNDLE-*, HOOK-*, etc.) against that single file
- For hook files (`use*.js`): focus on HOOK-1 through HOOK-4 + SSR-1, SSR-2, REACT-3, REACT-5
- For section files: focus on IMG-1–6, LCP-1, CLS-1, REACT-1–5, SSR-1, SSR-2, BUNDLE-2
- For page files: focus on LCP-1, SSR-1, REACT-5, PRECONNECT-1, SERVER-1

### Step C: Report and fix (same format as Mode 1 Step 2 + Step 3)

Output a compact report for just that file, then follow the same Critical → Major → Minor fix flow with permission gates.

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
