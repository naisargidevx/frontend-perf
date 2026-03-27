# CLS — Cumulative Layout Shift

**Good:** ≤ 0.1 | **Needs improvement:** 0.1–0.25 | **Poor:** > 0.25

CLS measures unexpected visual shifts during page load. Each shift = (impact fraction × distance fraction). Avoid shifts in the first 500ms of interaction.

---

## Fix 1: Always Set Width + Height on Images

The #1 CLS culprit. Without dimensions, the browser reserves no space — content jumps when the image loads.

```html
<!-- Bad: no dimensions -->
<img src="/photo.jpg" alt="Photo" />

<!-- Good: explicit dimensions (aspect ratio preserved by CSS) -->
<img src="/photo.jpg" alt="Photo" width="800" height="600" />
```

```css
/* Modern CSS to maintain aspect ratio while being responsive */
img {
  width: 100%;
  height: auto;
}
```

**Next.js `<Image>`:** Always provide `width` + `height` (or use `fill` with a positioned parent).

---

## Fix 2: Reserve Space for Dynamic / Async Content

Ads, embeds, banners, or lazy-loaded components that appear after initial render push content down.

```css
/* Reserve the slot before content loads */
.ad-slot {
  min-height: 250px;  /* match your ad unit size */
  width: 300px;
}

.skeleton-card {
  height: 200px;
  background: #f0f0f0;
  border-radius: 8px;
}
```

```jsx
// Show a skeleton while async content loads
function ProductCard({ loading, data }) {
  if (loading) return <SkeletonCard />;  /* same dimensions as real card */
  return <Card data={data} />;
}
```

---

## Fix 3: Fix Web Font Layout Shifts (FOUT / FOIT)

Fonts swapping in after load cause text reflow if the fallback and web font have different metrics.

```html
<!-- Preload critical fonts -->
<link rel="preload" href="/fonts/inter.woff2" as="font" type="font/woff2" crossorigin />
```

```css
/* Use font-display: optional for non-critical fonts (no FOUT) */
@font-face {
  font-family: 'Inter';
  src: url('/fonts/inter.woff2') format('woff2');
  font-display: optional; /* or 'swap' if font must show */
}
```

**Size-adjust technique** — match fallback font metrics to web font to eliminate shift:
```css
@font-face {
  font-family: 'InterFallback';
  src: local('Arial');
  size-adjust: 107%;       /* tweak until it matches */
  ascent-override: 90%;
  descent-override: 22%;
  line-gap-override: 0%;
}

body {
  font-family: 'Inter', 'InterFallback', sans-serif;
}
```

---

## Fix 4: Avoid Inserting Content Above Existing Content

Inserting banners, cookie notices, or notifications above the fold after load shifts everything down.

```js
// Bad: inject at top after page load
document.body.insertBefore(banner, document.body.firstChild);

// Good: reserve space in HTML from the start
// OR inject below the fold
// OR use position: fixed/sticky so it doesn't affect document flow
```

---

## Fix 5: Animate Only Transform + Opacity

Animating `top`, `left`, `width`, `height`, or `margin` causes layout recalculations and CLS.

```css
/* Bad: causes layout shift */
.slide-in {
  animation: slideIn 0.3s ease;
}
@keyframes slideIn {
  from { margin-top: -100px; }
  to   { margin-top: 0; }
}

/* Good: GPU-composited, no layout shift */
.slide-in {
  animation: slideIn 0.3s ease;
}
@keyframes slideIn {
  from { transform: translateY(-100px); opacity: 0; }
  to   { transform: translateY(0);      opacity: 1; }
}
```

---

## Fix 6: iframe / Embed Aspect Ratio

Embeds (YouTube, maps, etc.) need a reserved aspect ratio container.

```html
<!-- Bad: no reserved space -->
<iframe src="https://www.youtube.com/embed/..." ></iframe>

<!-- Good: aspect-ratio box -->
<div style="position: relative; padding-bottom: 56.25%; height: 0;">
  <iframe
    src="https://www.youtube.com/embed/..."
    style="position: absolute; inset: 0; width: 100%; height: 100%;"
    loading="lazy"
  ></iframe>
</div>
```

```css
/* Modern CSS approach */
.video-wrapper {
  aspect-ratio: 16 / 9;
  width: 100%;
}
.video-wrapper iframe {
  width: 100%;
  height: 100%;
}
```

---

## Measuring CLS

**Chrome DevTools → Performance panel:**
- Look for purple "Layout Shift" markers
- Click a shift to see which elements moved and their scores

**web-vitals library:**
```js
import { onCLS } from 'web-vitals';
onCLS(({ value, entries }) => {
  console.log('CLS score:', value);
  entries.forEach(e => console.log('Shifted element:', e.sources));
});
```

**Layout Instability API (manual):**
```js
new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (!entry.hadRecentInput) {
      console.log('Layout shift:', entry.value, entry.sources);
    }
  }
}).observe({ type: 'layout-shift', buffered: true });
```
