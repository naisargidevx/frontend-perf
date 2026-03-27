# Bundle & JavaScript Performance

Large JS bundles = slow TTI, high TBT, bad INP. Reduce parse/execute time on the main thread.

---

## Fix 1: Analyze Your Bundle First

Never optimize blindly. Find the actual large packages.

```bash
# Vite
npm i -D rollup-plugin-visualizer
# Add to vite.config.js:
# import { visualizer } from 'rollup-plugin-visualizer';
# plugins: [visualizer({ open: true })]

# Next.js
npm i -D @next/bundle-analyzer
# In next.config.js:
# const withBundleAnalyzer = require('@next/bundle-analyzer')({ enabled: true });
# module.exports = withBundleAnalyzer({});

# CRA / Webpack
npx webpack-bundle-analyzer stats.json
```

**What to look for:** Packages taking > 50KB gzipped that you might replace or lazy load.

---

## Fix 2: Replace Heavy Dependencies

| Heavy library | Lightweight alternative | Savings |
|---------------|------------------------|---------|
| `moment` (330KB) | `date-fns` or `dayjs` | ~280KB |
| `lodash` (full) | `lodash-es` with tree-shaking | ~50KB |
| `axios` | `ky` or native `fetch` | ~30KB |
| `react-icons` (all) | Only import used icons | 100–500KB |
| Full `antd` / `mui` | Import only used components | 200–600KB |

```js
// Bad: imports all icons (~500KB)
import { FiSearch, FiUser } from 'react-icons/fi';

// Good for react-icons: use direct path imports
import FiSearch from 'react-icons/fi/FiSearch';
import FiUser from 'react-icons/fi/FiUser';
```

---

## Fix 3: Tree Shaking — Use Named ES Module Imports

Tree shaking only works with ES modules (`import`/`export`), not CommonJS (`require`).

```js
// Bad: imports entire library, prevents tree shaking
import _ from 'lodash';
import * as dateFns from 'date-fns';

// Good: named imports — bundler removes unused code
import { debounce, throttle } from 'lodash-es';
import { format, parseISO } from 'date-fns';
```

Make sure your bundler is in production mode — tree shaking only runs in production builds.

---

## Fix 4: Dynamic Imports for Non-Critical Features

```js
// Bad: heavy PDF generator loaded on page init
import { generatePDF } from './pdfGenerator';

// Good: load only when user triggers the action
async function handleExport() {
  const { generatePDF } = await import('./pdfGenerator');
  generatePDF(data);
}
```

```jsx
// Lazy-load heavy React components
const RichEditor = lazy(() => import('./RichTextEditor'));
const DataGrid = lazy(() => import('./DataGrid'));
```

---

## Fix 5: Enable Compression (Gzip / Brotli)

Ensure your server/CDN compresses JS assets. Brotli gives ~15-25% better compression than gzip.

```nginx
# Nginx: enable brotli + gzip
brotli on;
brotli_comp_level 6;
brotli_types text/javascript application/javascript;

gzip on;
gzip_types text/javascript application/javascript;
```

**Vite:** generates pre-compressed `.br` files with `vite-plugin-compression`:
```js
import viteCompression from 'vite-plugin-compression';
plugins: [viteCompression({ algorithm: 'brotliCompress' })]
```

---

## Fix 6: Long-Term Caching with Content Hashing

```js
// vite.config.js
export default {
  build: {
    rollupOptions: {
      output: {
        // Vendor chunk: rarely changes → long cache lifetime
        manualChunks: {
          vendor: ['react', 'react-dom'],
          router: ['react-router-dom'],
          ui: ['@radix-ui/react-dialog', '@radix-ui/react-dropdown-menu'],
        },
        // Hash in filename for cache busting
        entryFileNames: 'assets/[name]-[hash].js',
        chunkFileNames: 'assets/[name]-[hash].js',
      },
    },
  },
};
```

```
Cache-Control: public, max-age=31536000, immutable
# Only for hashed filenames — safe to cache forever
```

---

## Fix 7: Remove Unused Code (Dead Code Elimination)

```bash
# Find unused exports
npx ts-prune                    # TypeScript projects
npx depcheck                    # Unused npm dependencies
```

```js
// tsconfig.json: strict mode catches unused variables
{
  "compilerOptions": {
    "noUnusedLocals": true,
    "noUnusedParameters": true
  }
}
```

---

## Fix 8: Module Federation / Micro-Frontends (Large Apps)

For very large applications, split into independently deployable chunks.

```js
// webpack.config.js (Module Federation)
new ModuleFederationPlugin({
  name: 'shell',
  remotes: {
    checkout: 'checkout@https://checkout.example.com/remoteEntry.js',
    catalog: 'catalog@https://catalog.example.com/remoteEntry.js',
  },
});
```

---

## Measuring Bundle Impact

**Bundle size tracking in CI:**
```bash
# bundlesize npm package
npx bundlesize
# In package.json:
# "bundlesize": [{ "path": "./dist/*.js", "maxSize": "150kB" }]
```

**Size Limit:**
```bash
npx size-limit
# .size-limit.json:
# [{ "path": "dist/index.js", "limit": "50 KB" }]
```

**Quick check in browser:**
```js
// Paste in DevTools console
performance.getEntriesByType('resource')
  .filter(r => r.initiatorType === 'script')
  .map(r => ({ url: r.name.split('/').pop(), kb: Math.round(r.transferSize / 1024) }))
  .sort((a, b) => b.kb - a.kb)
  .forEach(r => console.log(r.kb + 'KB', r.url));
```
