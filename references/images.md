# Image Optimization

Images are typically 50-70% of page weight and the #1 LCP bottleneck. Fix these first.

---

## Checklist (apply in order)

1. Correct format (WebP/AVIF over JPEG/PNG)
2. Correct size (serve at display size, not larger)
3. Correct loading strategy (`lazy` vs `eager` + `fetchpriority`)
4. Correct dimensions declared (`width` + `height` to prevent CLS)
5. Responsive srcset (different sizes for different viewports)
6. CDN + caching

---

## Fix 1: Use Modern Formats

| Format | Best for | Savings vs JPEG |
|--------|----------|-----------------|
| **WebP** | Photos, general | ~25-35% smaller |
| **AVIF** | Photos, illustrations | ~45-55% smaller |
| **SVG** | Icons, logos, illustrations | Infinitely scalable |
| **PNG** | Transparency needed (use WebP instead) | — |

```html
<picture>
  <source srcset="/hero.avif" type="image/avif" />
  <source srcset="/hero.webp" type="image/webp" />
  <img src="/hero.jpg" alt="Hero" width="1200" height="600" />
</picture>
```

**Next.js `<Image>` does this automatically** — it serves WebP/AVIF to supported browsers.

---

## Fix 2: Responsive Images with srcset + sizes

Don't serve a 2400px image to a 400px mobile screen.

```html
<img
  src="/photo-800.webp"
  srcset="
    /photo-400.webp  400w,
    /photo-800.webp  800w,
    /photo-1200.webp 1200w,
    /photo-1600.webp 1600w
  "
  sizes="
    (max-width: 480px) 100vw,
    (max-width: 1024px) 50vw,
    800px
  "
  alt="Photo"
  width="800"
  height="533"
/>
```

**Rule of thumb for `sizes`:** Match the CSS layout. If the image is full-width on mobile and 50% on desktop, use `(max-width: 768px) 100vw, 50vw`.

---

## Fix 3: Lazy Load Below-the-Fold Images

```html
<!-- Above-the-fold (LCP candidate): eager + high priority -->
<img src="/hero.webp" loading="eager" fetchpriority="high" width="1200" height="600" alt="Hero" />

<!-- Below-the-fold: lazy load -->
<img src="/card-photo.webp" loading="lazy" width="400" height="300" alt="Card" />
```

**Never** use `loading="lazy"` on the LCP image — it delays the most important paint.

---

## Fix 4: Next.js `<Image>` Component

Next.js `<Image>` handles format conversion, srcset generation, lazy loading, and dimension reservation automatically.

```jsx
import Image from 'next/image';

// LCP image: use priority
<Image
  src="/hero.jpg"
  alt="Hero"
  width={1200}
  height={600}
  priority           // adds preload + fetchpriority="high"
  quality={80}       // default 75; tune for quality/size balance
/>

// Below the fold: lazy (default)
<Image
  src="/card.jpg"
  alt="Card"
  width={400}
  height={300}
  // loading="lazy" is the default
/>

// Fill a container (no fixed width/height needed)
<div style={{ position: 'relative', height: '400px' }}>
  <Image
    src="/background.jpg"
    alt="Background"
    fill
    style={{ objectFit: 'cover' }}
    sizes="(max-width: 768px) 100vw, 50vw"
  />
</div>
```

---

## Fix 5: Set Explicit Width + Height (Prevents CLS)

```html
<!-- Bad: no dimensions → layout shift when image loads -->
<img src="/photo.webp" alt="Photo" />

<!-- Good: dimensions reserved immediately -->
<img src="/photo.webp" alt="Photo" width="800" height="600" />
```

```css
/* CSS to keep images responsive while preserving aspect ratio */
img {
  max-width: 100%;
  height: auto;
}
```

---

## Fix 6: Avoid Images for Decorative Content

Use CSS instead of images where possible:
- Gradients → `background: linear-gradient(...)`
- Simple shapes → CSS `border-radius`, `clip-path`
- Icons → SVG inline or icon font
- Patterns → CSS `background-image` with SVG data URI

---

## Fix 7: Image CDN Configuration

A proper image CDN (Cloudinary, Imgix, CloudFront + Lambda@Edge) resizes, formats, and caches on the fly.

```html
<!-- Cloudinary example: auto format + auto quality + width -->
<img
  src="https://res.cloudinary.com/demo/image/upload/f_auto,q_auto,w_800/sample.jpg"
  alt="Sample"
  width="800"
  height="600"
/>
```

Key CDN directives:
- `f_auto` / `format=auto` — serve WebP/AVIF automatically
- `q_auto` / `quality=auto` — smart quality compression
- `w_<N>` / `width=<N>` — resize to exact display size

---

## Fix 8: Compression Settings (Build Tools)

**Vite + vite-imagetools:**
```js
// vite.config.js
import { imagetools } from 'vite-imagetools';
export default { plugins: [imagetools()] };

// Usage in component
import heroUrl from './hero.jpg?format=webp&width=1200';
```

**Next.js — custom image loader for external CDN:**
```js
// next.config.js
module.exports = {
  images: {
    loader: 'cloudinary',
    path: 'https://res.cloudinary.com/demo/image/upload/',
    formats: ['image/avif', 'image/webp'],
    deviceSizes: [640, 750, 828, 1080, 1200, 1920],
  },
};
```

---

## Measuring Image Impact

**Chrome DevTools → Network tab:**
- Filter by "Img"
- Check size column — anything > 200KB for a displayed image is suspicious
- Check "Initiator" to see where late-discovered images come from

**Lighthouse:**
- "Properly size images" — flags images served larger than displayed
- "Serve images in next-gen formats" — flags JPEG/PNG when WebP/AVIF possible
- "Defer offscreen images" — flags missing `loading="lazy"`

**Quick size audit:**
```js
// Paste in DevTools console to audit all images on page
performance.getEntriesByType('resource')
  .filter(r => r.initiatorType === 'img')
  .map(r => ({ url: r.name.split('/').pop(), kb: Math.round(r.transferSize / 1024) }))
  .sort((a, b) => b.kb - a.kb)
  .forEach(r => console.log(r.kb + 'KB', r.url));
```
