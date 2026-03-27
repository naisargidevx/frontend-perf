# RUM — Real User Monitoring
## (web-vitals library, attribution, route segmentation)

Real User Monitoring (RUM) is how you measure what **actual visitors** experience. Lighthouse is lab data (controlled). CrUX/RUM is field data (real). You need both — but only field data confirms a fix landed for real users.

---

## Fix 1: Install `web-vitals` and Collect Core Web Vitals

```bash
npm install web-vitals
```

### Minimal setup (basic values)
```js
// src/analytics.js
import { onCLS, onINP, onLCP, onFCP, onTTFB } from 'web-vitals';

function sendToAnalytics({ name, value, rating }) {
  // 'good' | 'needs-improvement' | 'poor'
  fetch('/analytics', {
    method: 'POST',
    body: JSON.stringify({ metric: name, value: Math.round(value), rating }),
    keepalive: true, // survives page unload
  });
}

onCLS(sendToAnalytics);
onINP(sendToAnalytics);
onLCP(sendToAnalytics);
onFCP(sendToAnalytics);
onTTFB(sendToAnalytics);
```

### With attribution (recommended — tells you WHY a metric is bad)
```js
import { onCLS, onINP, onLCP } from 'web-vitals/attribution';

onLCP(({ name, value, rating, attribution }) => {
  sendToAnalytics({
    metric: name,
    value: Math.round(value),
    rating,
    // Which element was LCP?
    element: attribution.lcpEntry?.element,
    // What was the biggest delay?
    resourceLoadDelay: Math.round(attribution.resourceLoadDelay),
    ttfb: Math.round(attribution.timeToFirstByte),
  });
});

onINP(({ name, value, rating, attribution }) => {
  sendToAnalytics({
    metric: name,
    value: Math.round(value),
    rating,
    // Which interaction caused worst INP?
    eventType: attribution.interactionType,
    // Where was the time spent?
    inputDelay: Math.round(attribution.inputDelay),
    processingTime: Math.round(attribution.processingDuration),
    presentationDelay: Math.round(attribution.presentationDelay),
    // Which element was interacted with?
    element: attribution.interactionTargetElement,
  });
});
```

**Why attribution matters:** Without it, you know INP is 620ms. With it, you know it's a `button.add-to-cart` interaction where 540ms was input delay caused by a long task — pointing directly to the third-party script to fix.

---

## Fix 2: Send to Google Analytics 4

```js
import { onCLS, onINP, onLCP, onFCP, onTTFB } from 'web-vitals/attribution';

function sendToGA4({ name, value, rating, attribution }) {
  if (typeof gtag === 'undefined') return;
  gtag('event', name, {
    value: Math.round(name === 'CLS' ? value * 1000 : value),
    metric_rating: rating,
    metric_delta: Math.round(name === 'CLS' ? delta * 1000 : delta),
    // Include top attribution field per metric
    metric_attribution: JSON.stringify(
      name === 'LCP' ? { element: attribution.lcpEntry?.element } :
      name === 'INP' ? { eventType: attribution.interactionType, inputDelay: attribution.inputDelay } :
      name === 'CLS' ? { largestShiftTarget: attribution.largestShiftTarget } :
      {}
    ),
  });
}

onCLS(sendToGA4); onINP(sendToGA4); onLCP(sendToGA4);
onFCP(sendToGA4); onTTFB(sendToGA4);
```

Then in GA4 explore: segment by `metric_rating = 'poor'` to find your worst-performing users.

---

## Fix 3: SPA Route Segmentation

For SPAs, CWV is measured per page load. Route transitions are **soft navigations** — they don't reset the CWV observers by default. You need to pass a route identifier with each metric so you can segment by route in your analytics.

```js
import { onLCP, onINP, onCLS } from 'web-vitals/attribution';

// Get current route at measurement time
const getRoute = () => window.location.pathname.replace(/\/[0-9]+/g, '/:id');

function sendWithRoute(metric) {
  sendToAnalytics({
    ...metric,
    route: getRoute(),
    navigationType: metric.navigationType, // 'navigate' | 'reload' | 'back-forward' | 'prerender'
  });
}

onLCP(sendWithRoute);
onINP(sendWithRoute);
onCLS(sendWithRoute);
```

> **Important limitation:** For hard navigations (full page loads), this works perfectly. For soft navigations (React Router route changes), `onLCP` will fire once for the initial load — it won't automatically reset per route transition. Track soft navigation performance separately using `performance.mark` / `performance.measure` around route transitions.

### Soft navigation instrumentation for SPAs

```js
// In your router change handler (React Router example)
import { useEffect } from 'react';
import { useLocation } from 'react-router-dom';

function RoutePerformanceTracker() {
  const location = useLocation();

  useEffect(() => {
    const start = performance.now();
    const route = location.pathname;

    // Measure when the route is visually stable
    requestAnimationFrame(() => {
      requestAnimationFrame(() => {
        const duration = performance.now() - start;
        sendToAnalytics({ metric: 'route-transition', value: Math.round(duration), route });
      });
    });
  }, [location.pathname]);

  return null;
}
```

---

## Fix 4: Interpret CrUX (Free Field Data Without Any Code)

PageSpeed Insights shows **CrUX data** (Chrome UX Report) — real user data from Chrome browsers aggregated over 28 days. You don't need to instrument anything to get it.

```
https://pagespeed.web.dev → enter your URL → scroll to "Discover what your real users are experiencing"
```

| CrUX metric | What it tells you | Limitation |
|-------------|------------------|-----------|
| p75 LCP | 75th percentile LCP for real visitors | 28-day window; new releases take time to show |
| p75 INP | 75th percentile interaction responsiveness | Only available if enough Chrome users visit |
| p75 CLS | 75th percentile layout shift score | Not segmented by route |
| "Good" % | % of users in the green zone | Use this for stakeholder reporting |

**CrUX vs your own RUM:**
- CrUX: free, zero setup, 28-day lag, no route breakdown, only Chrome
- Your RUM (`web-vitals`): real-time, all browsers, route-level, attribution data, requires implementation

Use CrUX to confirm your overall health and your own RUM to debug specific routes.

---

## Fix 5: Alert on Field Regressions

Once you're collecting RUM data, set p75 alert thresholds:

| Metric | Warning | Critical |
|--------|---------|---------|
| LCP | > 2.5s | > 4.0s |
| INP | > 200ms | > 500ms |
| CLS | > 0.1 | > 0.25 |
| FCP | > 1.8s | > 3.0s |
| TTFB | > 800ms | > 1.8s |

Minimum sample size before alerting: 100 sessions per route per day (to avoid false alarms on low-traffic routes).

---

## Measuring

- **PageSpeed Insights** — free CrUX field data: `https://pagespeed.web.dev`
- **Chrome UX Report** — BigQuery dataset for bulk analysis
- **web-vitals** GitHub: `https://github.com/GoogleChrome/web-vitals`
- **DevTools → Performance Insights** — shows "Slow interactions" with INP breakdown locally
