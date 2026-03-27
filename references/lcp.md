# LCP — Largest Contentful Paint

**Good:** ≤ 2.5s | **Needs improvement:** 2.5–4s | **Poor:** > 4s

LCP measures when the largest visible element (image, video poster, or text block) finishes rendering in the viewport.

---

## Most Common LCP Elements
- Hero `<img>` or `<picture>`
- CSS `background-image` on a hero section
- Large `<h1>` or text block above the fold
- Video poster frame

---

## Fix 1: Preload the LCP Image

The single highest-impact LCP fix. Without a preload hint, the browser discovers the image only after parsing HTML+CSS.

```html
<!-- In <head> — add fetchpriority="high" -->
<link rel="preload" as="image" href="/hero.webp" fetchpriority="high" />

<!-- For responsive images, use imagesrcset -->
<link
  rel="preload"
  as="image"
  imagesrcset="/hero-480.webp 480w, /hero-800.webp 800w, /hero-1200.webp 1200w"
  imagesizes="(max-width: 600px) 480px, (max-width: 1024px) 800px, 1200px"
  fetchpriority="high"
/>
```

**Next.js:** use `priority` prop on `<Image>` — it auto-adds the preload link.
```jsx
// before
<Image src="/hero.webp" width={1200} height={600} alt="Hero" />
// after
<Image src="/hero.webp" width={1200} height={600} alt="Hero" priority />
```

---

## Fix 2: Serve Images in Modern Formats

WebP is ~25-35% smaller than JPEG; AVIF is ~50% smaller.

```html
<picture>
  <source srcset="/hero.avif" type="image/avif" />
  <source srcset="/hero.webp" type="image/webp" />
  <img src="/hero.jpg" alt="Hero" width="1200" height="600" />
</picture>
```

---

## Fix 3: Eliminate Render-Blocking Resources

Render-blocking CSS/JS delays LCP by holding up the first paint.

```html
<!-- Bad: blocks rendering -->
<link rel="stylesheet" href="/styles.css" />
<script src="/app.js"></script>

<!-- Good: non-blocking JS -->
<script src="/app.js" defer></script>
<!-- Good: preload + async font -->
<link rel="preload" href="/font.woff2" as="font" type="font/woff2" crossorigin />
```

---

## Fix 4: Reduce TTFB (Server Response Time)

LCP can't start until the first byte arrives. TTFB > 800ms is a red flag.

- Add CDN caching for static assets
- Enable HTTP/2 or HTTP/3
- Use `Cache-Control: public, max-age=31536000, immutable` for hashed assets
- For SSR: use ISR/SSG where possible instead of full server render

---

## Fix 5: Avoid Lazy Loading the LCP Image

`loading="lazy"` on the LCP image delays it — only use lazy load for below-the-fold images.

```html
<!-- Bad: lazy loading the hero -->
<img src="/hero.jpg" loading="lazy" />

<!-- Good: eager (default) for above-the-fold -->
<img src="/hero.jpg" loading="eager" fetchpriority="high" />
```

---

## Fix 6: CSS Background Images as LCP Element

The browser can't preload a CSS background image without a hint because it doesn't know which element will be LCP until layout.

```html
<!-- Add a preload specifically for the bg image -->
<link rel="preload" as="image" href="/hero-bg.webp" fetchpriority="high" />
```

Or better: switch from CSS `background-image` to an `<img>` element for the LCP candidate — it's more predictable.

---

## Measuring LCP

**Chrome DevTools:**
- Performance panel → look for "LCP" marker on the timeline
- Click the LCP marker to see which element triggered it

**web-vitals library:**
```js
import { onLCP } from 'web-vitals';
onLCP(({ value, entries }) => {
  console.log('LCP:', value, entries[0].element);
});
```

**Lighthouse:** Run in incognito mode; check "Largest Contentful Paint" in the metrics section.

**CrUX / PageSpeed Insights:** Real-user LCP data from Chrome users — more representative than lab tests.
