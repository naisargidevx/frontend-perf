# Server-Timing — TTFB Diagnosis and Backend Performance

TTFB > 800ms is a hard cap on how fast FCP and LCP can be. But "slow TTFB" is not actionable — you need to know *which backend phase* is slow. `Server-Timing` exposes server-side metrics to the browser, turning a single TTFB number into a breakdown of db / app / cache / cdn.

---

## Fix 1: Add Server-Timing Headers

The header syntax:
```
Server-Timing: <name>;dur=<ms>;desc="<label>"
```

Multiple phases:
```
Server-Timing: db;dur=410, cache;dur=20;desc="MISS", app;dur=180, total;dur=620
```

### Express / Node.js

```js
app.get('/product/:id', async (req, res) => {
  const t = {};

  t.cacheStart = performance.now();
  const cached = await cache.get(`product:${req.params.id}`);
  t.cacheDur = performance.now() - t.cacheStart;

  let product;
  if (cached) {
    product = cached;
  } else {
    t.dbStart = performance.now();
    product = await db.products.findById(req.params.id);
    t.dbDur = performance.now() - t.dbStart;

    await cache.set(`product:${req.params.id}`, product, 300);
  }

  t.renderStart = performance.now();
  const html = await renderTemplate('product', { product });
  t.renderDur = performance.now() - t.renderStart;

  // Build Server-Timing header
  const timings = [
    `cache;dur=${t.cacheDur.toFixed(1)};desc="${cached ? 'HIT' : 'MISS'}"`,
    t.dbDur ? `db;dur=${t.dbDur.toFixed(1)}` : null,
    `render;dur=${t.renderDur.toFixed(1)}`,
  ].filter(Boolean).join(', ');

  res.setHeader('Server-Timing', timings);
  res.send(html);
});
```

### Next.js App Router (React Server Component)

```tsx
// app/product/[id]/page.tsx
import { headers } from 'next/headers';

export default async function ProductPage({ params }) {
  const start = Date.now();

  const cacheStart = Date.now();
  // ... cache lookup ...
  const cacheDur = Date.now() - cacheStart;

  const dbStart = Date.now();
  const product = await db.getProduct(params.id);
  const dbDur = Date.now() - dbStart;

  // Next.js 15+ supports headers() in RSC
  // Alternatively, set in middleware or route handlers
  const headersList = headers();

  return (
    <>
      {/* Add via response headers in middleware for full SSR timing */}
      <ProductDetail product={product} />
    </>
  );
}
```

```ts
// middleware.ts — cleaner pattern for App Router
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  const response = NextResponse.next();
  // Timing gets added by API routes; middleware can add edge timing
  response.headers.set('Server-Timing', 'edge;dur=1;desc="CDN edge"');
  return response;
}
```

### Next.js API Route (Pages Router)

```ts
// pages/api/product/[id].ts
import type { NextApiRequest, NextApiResponse } from 'next';

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  const dbStart = Date.now();
  const product = await db.getProduct(req.query.id as string);
  const dbDur = Date.now() - dbStart;

  res.setHeader('Server-Timing', `db;dur=${dbDur}`);
  res.json(product);
}
```

---

## Fix 2: Read Server-Timing in DevTools

1. Open Chrome DevTools → **Network** tab
2. Click the HTML document request (first request)
3. Click **Timing** tab → scroll to **Server Timing** section

You'll see each phase as a labeled bar:
```
cache   ▓░░░  20ms (MISS)
db      ████████████████  410ms
render  ████  50ms
```

**What to look for:**
- `db` > 200ms → query optimization, connection pool, or caching needed
- `cache` with `desc=MISS` → caching not working; check TTL and cache key
- `render` > 100ms → template/component render is slow; consider streaming SSR or caching rendered fragments
- All phases small but TTFB still high → CDN overhead, DNS, TLS, or network path issue

---

## Fix 3: Read Server-Timing in JavaScript (PerformanceServerTiming API)

```js
// In browser — reads Server-Timing values programmatically
const [navEntry] = performance.getEntriesByType('navigation');

navEntry.serverTiming.forEach(({ name, duration, description }) => {
  console.log(`${name}: ${duration}ms (${description})`);
  // Send to RUM for field-level TTFB subpart tracking
  sendToAnalytics({ metric: `server-${name}`, value: Math.round(duration) });
});
```

Combine with your `web-vitals` TTFB data to get full picture:
```js
import { onTTFB } from 'web-vitals/attribution';

onTTFB(({ value, attribution }) => {
  const serverTimings = performance.getEntriesByType('navigation')[0]?.serverTiming ?? [];
  const dbTime = serverTimings.find(t => t.name === 'db')?.duration ?? 0;

  sendToAnalytics({
    metric: 'TTFB',
    value: Math.round(value),
    // Subparts from attribution
    waitingTime: Math.round(attribution.waitingDuration),
    // Subparts from server
    dbTime: Math.round(dbTime),
  });
});
```

---

## Fix 4: Add Cache-Control to HTML Responses

A slow TTFB on every page load often means the HTML is not cached at the CDN. Fix the caching strategy:

```js
// Express — cache SSR HTML at CDN for 60s, stale-while-revalidate for 5 min
res.setHeader('Cache-Control', 'public, max-age=60, stale-while-revalidate=300');

// Static assets (hashed filenames) — cache forever
res.setHeader('Cache-Control', 'public, max-age=31536000, immutable');
```

```js
// next.config.js — cache HTML pages
async headers() {
  return [
    {
      source: '/product/:id',
      headers: [
        { key: 'Cache-Control', value: 'public, max-age=60, stale-while-revalidate=300' },
      ],
    },
    {
      source: '/_next/static/:path*',
      headers: [
        { key: 'Cache-Control', value: 'public, max-age=31536000, immutable' },
      ],
    },
  ];
}
```

**Decision tree for TTFB:**
```
TTFB > 800ms?
├─ Server-Timing: db > 300ms  → Add DB query caching (Redis) or query optimization
├─ Server-Timing: cache = MISS → Fix cache key / TTL; consider ISR/SSG for the route
├─ Server-Timing: render > 200ms → Streaming SSR; server component data parallelism
└─ All phases < 50ms but TTFB still high → CDN misconfiguration; check origin vs edge hits
```

---

## Measuring TTFB

**web-vitals:**
```js
import { onTTFB } from 'web-vitals';
onTTFB(({ value }) => console.log('TTFB:', value + 'ms'));
```

**Chrome DevTools → Network tab:**
- Click the first HTML request → Timing tab → "Waiting for server response" = TTFB

**PageSpeed Insights:** "Reduce initial server response time" opportunity shows TTFB and flags if > 600ms.
