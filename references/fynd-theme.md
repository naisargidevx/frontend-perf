# Fynd Theme — Performance Rules & Safe Patterns

Fynd themes are SSR React apps with a strict folder contract. Performance fixes must never break the platform's rendering pipeline.

---

## Theme Folder Structure (DO NOT restructure)

```
themes/
├── index.jsx                  ← PROTECTED: entry bootstrap — do not rename/move
├── pages/                     ← PROTECTED: system pages ONLY — no arbitrary files
│   ├── product-listing.jsx
│   ├── product-description.jsx
│   └── ... (exact system names)
├── sections/                  ← reusable section components
├── custom-templates/          ← custom page templates (/c/page-name routes)
├── components/                ← shared components
├── assets/                    ← images, stylesheets, scripts
├── styles/
├── helper/
├── constants/
├── page-layouts/
└── config/
    ├── settings_data.json     ← PROTECTED: global theme settings structure
    └── settings_schema.json   ← PROTECTED: schema structure
```

### Files/Folders You Must NEVER Modify or Rename

| File | Why Protected |
|------|--------------|
| `index.jsx` | Theme bootstrap — FDK reads this as entry point |
| `pages/*.jsx` file names | System page names are exact strings FDK expects |
| `config/settings_data.json` structure | Theme editor reads this shape |
| `config/settings_schema.json` structure | Editor schema contract |

### Safe Folders to Optimize Freely

- `components/` — full freedom
- `sections/` — safe, but keep Component + settings exports
- `custom-templates/` — safe
- `styles/` — safe
- `helper/` — safe
- `assets/` — safe

---

## Performance Targets (Fynd Submission Requirements)

| Metric | Target |
|--------|--------|
| GTmetrix grade | **A** |
| Page speed | **< 1.4 seconds** |
| LCP | ≤ 2.5s |
| CLS | ≤ 0.1 |
| INP | ≤ 200ms |

---

## Critical: SSR-Safe Code

Fynd themes render server-side first. Any direct `window`/`document` access breaks SSR.

```jsx
// BAD — breaks server render
const width = window.innerWidth;
document.getElementById('root');

// GOOD — SSR-safe guard
if (typeof window !== 'undefined') {
  const width = window.innerWidth;
}

// GOOD — React hook (runs only client-side)
useEffect(() => {
  const width = window.innerWidth; // safe inside useEffect
}, []);
```

**Applies everywhere:** components, sections, custom-templates, helper utilities.

---

## Image Optimization: Use `transformImage`

Fynd provides a `transformImage` utility that leverages Pixelbin for responsive resizing. Always use it — never hardcode image URLs directly.

```jsx
// BAD — fixed URL, no optimization
<img src={product.image} alt={product.name} />

// GOOD — use transformImage for responsive delivery
import { transformImage } from 'fdk-core/utils';

<img
  src={transformImage(product.image, { width: 800, format: 'webp' })}
  srcSet={`
    ${transformImage(product.image, { width: 400, format: 'webp' })} 400w,
    ${transformImage(product.image, { width: 800, format: 'webp' })} 800w,
    ${transformImage(product.image, { width: 1200, format: 'webp' })} 1200w
  `}
  sizes="(max-width: 480px) 100vw, (max-width: 1024px) 50vw, 800px"
  loading="lazy"
  width="800"
  height="600"
  alt={product.name}
/>
```

**LCP image** (hero/first image above fold): do NOT use `loading="lazy"` — use `loading="eager"`.

```jsx
// Hero image: eager + high priority
<img
  src={transformImage(heroImage, { width: 1200, format: 'webp' })}
  loading="eager"
  fetchpriority="high"
  width="1200"
  height="600"
  alt="Hero"
/>
```

---

## Section Code Splitting (Safe Pattern)

Sections load individually in Fynd — each section should be independently splittable.

```jsx
// In index.jsx bootstrap (safe code splitting)
const Sections = {
  'hero-banner': React.lazy(() => import('./sections/HeroBanner')),
  'product-grid': React.lazy(() => import('./sections/ProductGrid')),
  'testimonials': React.lazy(() => import('./sections/Testimonials')),
};

// Wrap section renderer with Suspense
<Suspense fallback={<SectionSkeleton />}>
  {SectionComponent && <SectionComponent {...props} />}
</Suspense>
```

**Do not** put lazy imports inside the `settings` export — only the Component export can be lazy.

---

## Section Component Pattern (Required Exports)

Every section must export both `Component` and `settings`. Do not remove either.

```jsx
// sections/HeroBanner.jsx

// Component export — optimize freely inside here
export function Component({ props, blocks, preset, globalConfig }) {
  const heroImage = props.image?.value;
  const title = props.title?.value;

  return (
    <section>
      {heroImage && (
        <img
          src={transformImage(heroImage, { width: 1200, format: 'webp' })}
          loading="eager"
          fetchpriority="high"
          width="1200"
          height="600"
          alt={title || 'Hero'}
        />
      )}
      <h1>{title}</h1>
    </section>
  );
}

// settings export — REQUIRED, do not remove or rename
export const settings = {
  label: 'Hero Banner',
  props: [
    { id: 'image', type: 'image_picker', label: 'Hero Image' },
    { id: 'title', type: 'text', label: 'Title', default: 'Welcome' },
  ],
  blocks: [],
};
```

---

## serverFetch — Data Fetching for SSR

Use `serverFetch` static method on page components for server-side data. Do not fetch in component body for critical data.

```jsx
// pages/product-listing.jsx
export default function ProductListing({ serverData }) {
  const products = serverData?.products ?? [];
  return <ProductGrid products={products} />;
}

// Runs on server before render — performance critical
ProductListing.serverFetch = async ({ router, store, fpi }) => {
  const data = await fpi.executeGQL(PRODUCT_LIST_QUERY);
  return { products: data.products };
};

// Optional: redirect non-logged-in users
ProductListing.authGuard = true;
```

**Performance note:** Keep `serverFetch` lean — it blocks the first render. Defer non-critical data to client-side `useEffect`.

---

## Canvas Rules (Avoid Breaking Section Layout)

- Partners define canvases, sellers customize content within them
- If you add a new canvas key that doesn't match existing section values → **sections migrate to default canvas** (layout breaks)
- Do not rename or delete canvas keys in `settings_schema.json`
- Safe: add new sections, modify section internals, add canvas keys with matching section assignments

---

## Performance Anti-Patterns Specific to Fynd

| Anti-pattern | Why it breaks Fynd | Fix |
|-------------|-------------------|-----|
| `window.*` at module level | Breaks SSR | Wrap in `typeof window !== 'undefined'` |
| Arbitrary files in `pages/` | FDK throws errors | Move to `components/` or `helper/` |
| Renaming system page files | FDK can't find routes | Never rename `pages/*.jsx` |
| Removing `settings` export from section | Editor breaks | Always export both `Component` + `settings` |
| `loading="lazy"` on hero image | Delays LCP | Use `loading="eager" fetchpriority="high"` |
| Raw image URLs without `transformImage` | No CDN resize/WebP | Always use `transformImage` |
| Heavy sync work in `serverFetch` | Delays TTFB | Defer non-critical work to client |
| Direct DOM manipulation outside useEffect | SSR crash | Use `useEffect` + `useRef` |

---

## Safe Performance Wins for Fynd Themes

These are safe to apply without risking theme structure:

1. **Add `transformImage` + srcset** to all `<img>` tags in sections and components
2. **Add `width` + `height`** to all images (prevents CLS)
3. **Add `loading="lazy"`** to below-fold images in sections
4. **Wrap heavy sections** in `React.lazy` + `Suspense`
5. **Add `useMemo`** for expensive filter/sort in product listing pages
6. **Add `fetchpriority="high"`** to the hero/LCP image
7. **Guard `window`/`document`** with SSR checks
8. **Move non-critical `serverFetch` data** to client-side `useEffect`
9. **Virtualize** long product lists with `react-window`
10. **Debounce search** input handlers

---

## FDK CLI Commands

```bash
fdk theme serve    # local preview at localhost:5001
fdk theme sync     # sync changes to platform
fdk theme open     # open live preview
```

Always test locally with `fdk theme serve` before syncing. The FDK server mimics SSR behavior.
