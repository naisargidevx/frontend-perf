# FCP — First Contentful Paint
## (also: TTFB — Time to First Byte, Critical Rendering Path)

**FCP Good:** ≤ 1.8s | **Needs improvement:** 1.8–3s | **Poor:** > 3s
**TTFB Good:** ≤ 800ms | **Needs improvement:** 800ms–1.8s | **Poor:** > 1.8s

FCP marks when any content (text, image, SVG) first appears. It's blocked by render-blocking resources and TTFB.

---

## Fix 1: Eliminate Render-Blocking CSS

CSS in `<link>` blocks rendering until fully downloaded and parsed.

```html
<!-- Bad: large stylesheet blocks FCP -->
<link rel="stylesheet" href="/styles.css" /> <!-- 200KB -->

<!-- Fix 1: Inline critical CSS, defer the rest -->
<style>
  /* Only above-the-fold critical styles */
  body { margin: 0; font-family: sans-serif; }
  .hero { background: #000; color: #fff; padding: 60px; }
</style>
<!-- Non-critical: load async -->
<link rel="preload" href="/styles.css" as="style" onload="this.onload=null;this.rel='stylesheet'" />
<noscript><link rel="stylesheet" href="/styles.css" /></noscript>
```

**Automated critical CSS extraction:**
```bash
npm i -D critical
# Extracts and inlines critical CSS automatically
```

---

## Fix 2: Eliminate Render-Blocking Scripts

```html
<!-- Bad: blocks HTML parsing -->
<script src="/app.js"></script>

<!-- Good options: -->
<script src="/app.js" defer></script>   <!-- runs after HTML parsed, in order -->
<script src="/app.js" async></script>   <!-- runs as soon as downloaded, out of order -->

<!-- Rule: use defer for scripts that depend on DOM; async for independent scripts (analytics) -->
```

---

## Fix 3: Reduce TTFB

TTFB > 800ms is a hard cap on how fast FCP can be.

**Static sites / Next.js SSG:**
```js
// next.config.js: add Cache-Control headers for SSG pages
headers: async () => [{
  source: '/(.*)',
  headers: [{
    key: 'Cache-Control',
    value: 'public, max-age=3600, stale-while-revalidate=86400',
  }],
}]
```

**CDN edge caching:** Deploy to Vercel, Netlify, or Cloudflare Workers — TTFB < 50ms from edge.

**SSR optimization:**
- Use ISR (Next.js Incremental Static Regeneration) instead of per-request SSR
- Cache database queries with Redis/Memcached
- Use `React.cache()` (Next.js App Router) for deduplicated data fetching

---

## Fix 4: Optimize Web Font Loading

Fonts are a common FCP blocker.

```html
<!-- Step 1: Preconnect to font provider -->
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />

<!-- Step 2: Preload the most critical font weight -->
<link
  rel="preload"
  href="https://fonts.gstatic.com/s/inter/v13/inter-regular.woff2"
  as="font"
  type="font/woff2"
  crossorigin
/>
```

```css
/* Step 3: font-display strategy */
@font-face {
  font-family: 'Inter';
  src: url('/fonts/inter.woff2') format('woff2');
  font-display: swap;     /* show fallback immediately, swap when loaded */
  /* font-display: optional; — never shifts, uses fallback if not cached */
}
```

**Next.js — use `next/font`** (zero-layout-shift font loading built in):
```jsx
import { Inter } from 'next/font/google';
const inter = Inter({ subsets: ['latin'], display: 'swap' });
export default function Layout({ children }) {
  return <html className={inter.className}>{children}</html>;
}
```

---

## Fix 5: Server-Side Rendering (SSR) vs Client-Side Rendering (CSR)

CSR apps show a blank page until JS downloads, parses, and executes — terrible FCP.

```
CSR FCP timeline:
[TTFB] → [HTML (empty)] → [JS download] → [JS parse] → [React render] → [FCP] ← too late

SSR/SSG FCP timeline:
[TTFB] → [HTML (content)] → [FCP] ← immediate
```

**Migration path for CSR apps:**
1. Next.js App Router (easiest for React)
2. Vite + vite-plugin-ssr
3. Astro (for mostly-static content)

---

## Fix 6: Reduce Initial HTML Payload

```html
<!-- Don't inline large scripts/data in HTML -->
<!-- Bad: 500KB of JSON in HTML -->
<script>window.__INITIAL_DATA__ = { /* huge JSON */ };</script>

<!-- Good: fetch from API after hydration -->
<!-- Or: use Next.js generateStaticParams / server components to stream data -->
```

---

## Measuring FCP / TTFB

**web-vitals library:**
```js
import { onFCP, onTTFB } from 'web-vitals';
onFCP(({ value }) => console.log('FCP:', value + 'ms'));
onTTFB(({ value }) => console.log('TTFB:', value + 'ms'));
```

**Chrome DevTools → Network tab:**
- Click the first HTML document request
- "Waiting for server response" = TTFB
- Set throttling to "Slow 3G" to simulate real-world conditions

**Lighthouse:** "First Contentful Paint" in metrics; "Eliminate render-blocking resources" in opportunities.
