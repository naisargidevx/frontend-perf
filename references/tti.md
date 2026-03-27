# TTI — Time to Interactive
## (also: TBT — Total Blocking Time)

**TTI Good:** ≤ 3.8s | **Needs improvement:** 3.8–7.3s | **Poor:** > 7.3s
**TBT Good:** ≤ 200ms | **Needs improvement:** 200–600ms | **Poor:** > 600ms

TTI measures when the page becomes reliably interactive — main thread quiet for 5s with no long tasks > 50ms. TBT (Total Blocking Time) is the lab-measurable proxy Lighthouse uses.

---

## Fix 1: Code Splitting — Route-Level

The most impactful TTI fix. Don't ship the whole app bundle on the first load.

**React + React Router:**
```jsx
import { lazy, Suspense } from 'react';
import { BrowserRouter, Routes, Route } from 'react-router-dom';

// Bad: eager imports
import HomePage from './pages/HomePage';
import DashboardPage from './pages/DashboardPage';
import SettingsPage from './pages/SettingsPage';

// Good: lazy route splitting
const HomePage = lazy(() => import('./pages/HomePage'));
const DashboardPage = lazy(() => import('./pages/DashboardPage'));
const SettingsPage = lazy(() => import('./pages/SettingsPage'));

function App() {
  return (
    <BrowserRouter>
      <Suspense fallback={<PageSkeleton />}>
        <Routes>
          <Route path="/" element={<HomePage />} />
          <Route path="/dashboard" element={<DashboardPage />} />
          <Route path="/settings" element={<SettingsPage />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  );
}
```

**Next.js** — pages are automatically code-split by route.

---

## Fix 2: Defer Third-Party Scripts

Analytics, chat widgets, A/B testing scripts are the #1 cause of long tasks.

```html
<!-- Bad: blocks main thread during parse -->
<script src="https://cdn.analytics.com/tracker.js"></script>

<!-- Good: defer until after page is interactive -->
<script src="https://cdn.analytics.com/tracker.js" defer></script>

<!-- Better: async (loads in parallel, executes when ready) -->
<script src="https://cdn.analytics.com/tracker.js" async></script>

<!-- Best: load after user interaction or idle -->
<script>
  // Load after first user interaction
  const load3P = () => {
    const s = document.createElement('script');
    s.src = 'https://cdn.widget.com/chat.js';
    document.head.appendChild(s);
    ['click', 'scroll', 'keydown'].forEach(e => document.removeEventListener(e, load3P));
  };
  ['click', 'scroll', 'keydown'].forEach(e => document.addEventListener(e, load3P, { once: true }));
</script>
```

---

## Fix 3: Reduce JavaScript Bundle Size

```bash
# Analyze your bundle
npx vite-bundle-visualizer          # Vite
npx webpack-bundle-analyzer         # Webpack
npx @next/bundle-analyzer           # Next.js
```

Common quick wins:
- Replace `moment.js` → `date-fns` (saves ~200KB gzipped)
- Replace `lodash` import with named imports: `import debounce from 'lodash/debounce'`
- Replace icon library bulk import with individual icons
- Remove polyfills for modern browsers (check browserslist config)

```js
// Bad: imports entire lodash (~70KB gzipped)
import _ from 'lodash';
const result = _.debounce(fn, 300);

// Good: imports only what you need (~2KB)
import debounce from 'lodash/debounce';
const result = debounce(fn, 300);
```

---

## Fix 4: Lazy Load Heavy Components

```jsx
import { lazy, Suspense, useState } from 'react';

// Heavy chart library — don't load until needed
const Chart = lazy(() => import('./Chart'));
const RichTextEditor = lazy(() => import('./RichTextEditor'));

function Dashboard() {
  const [showChart, setShowChart] = useState(false);

  return (
    <div>
      <button onClick={() => setShowChart(true)}>Show Chart</button>
      {showChart && (
        <Suspense fallback={<div>Loading chart...</div>}>
          <Chart data={data} />
        </Suspense>
      )}
    </div>
  );
}
```

---

## Fix 5: Move Work Off Main Thread (Web Workers)

CPU-intensive tasks (parsing, encryption, image processing) should run in a worker.

```js
// worker.js
self.onmessage = ({ data }) => {
  const result = heavyCalculation(data); // runs off main thread
  self.postMessage(result);
};

// main.js
const worker = new Worker(new URL('./worker.js', import.meta.url));
worker.postMessage(largeDataset);
worker.onmessage = ({ data }) => {
  setState(data); // update UI with result
};
```

**Comlink** simplifies worker communication:
```js
import * as Comlink from 'comlink';
const worker = Comlink.wrap(new Worker('./worker.js'));
const result = await worker.heavyCalculation(data);
```

---

## Fix 6: Preload Critical Resources, Prefetch Future Pages

```html
<!-- Preload: resources needed for current page -->
<link rel="preload" href="/critical.js" as="script" />
<link rel="preload" href="/hero.webp" as="image" fetchpriority="high" />

<!-- Prefetch: resources likely needed on next navigation -->
<link rel="prefetch" href="/dashboard-bundle.js" as="script" />

<!-- Preconnect: establish connection to third-party domains early -->
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
<link rel="preconnect" href="https://cdn.analytics.com" />
```

---

## Measuring TTI / TBT

**Lighthouse:** "Total Blocking Time" is the best lab proxy for TTI. Aim for < 200ms.

**Chrome DevTools → Performance panel:**
- Long red tasks in the main thread timeline are "long tasks" (> 50ms)
- "Total blocking time" in the summary = sum of time beyond 50ms for each long task

**Long Tasks API:**
```js
new PerformanceObserver((list) => {
  list.getEntries().forEach(entry => {
    console.log('Long task:', entry.duration.toFixed(0) + 'ms', entry.attribution);
  });
}).observe({ type: 'longtask', buffered: true });
```
