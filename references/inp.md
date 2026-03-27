# INP — Interaction to Next Paint
## (replaces FID as Core Web Vital since March 2024)

**Good:** ≤ 200ms | **Needs improvement:** 200–500ms | **Poor:** > 500ms

INP measures the latency of ALL interactions (click, keypress, tap) — specifically the time from user input to the next frame paint. Unlike FID which only measured the *first* interaction delay.

---

## How INP Breaks Down

```
INP = Input Delay + Processing Time + Presentation Delay
```

- **Input delay:** Time waiting for main thread to be free (long tasks block this)
- **Processing time:** Time running the event handler JS
- **Presentation delay:** Time for browser to render the updated frame

---

## Fix 1: Break Up Long Tasks

Any JS task > 50ms on the main thread blocks interactions. Use `scheduler.yield()` or `setTimeout` to yield back to the browser.

```js
// Bad: one long synchronous task
function processLargeList(items) {
  items.forEach(item => {
    heavyOperation(item); // 200ms total
  });
}

// Good: yield between chunks
async function processLargeList(items) {
  for (let i = 0; i < items.length; i++) {
    heavyOperation(items[i]);
    // yield every 50 items so browser can handle interactions
    if (i % 50 === 0) {
      await scheduler.yield?.() ?? new Promise(r => setTimeout(r, 0));
    }
  }
}
```

---

## Fix 2: Defer Non-Critical Event Handler Work

Don't do everything synchronously in click/keydown handlers. Defer work that doesn't need to happen before the next paint.

```js
// Bad: all work blocks the paint
button.addEventListener('click', () => {
  updateUI();          // needs to happen now
  logAnalytics();      // can wait
  syncToServer();      // can wait
  rebuildSearchIndex(); // definitely can wait
});

// Good: only update UI synchronously
button.addEventListener('click', () => {
  updateUI(); // renders in next frame

  // defer everything else
  setTimeout(() => {
    logAnalytics();
    syncToServer();
  }, 0);

  requestIdleCallback(() => {
    rebuildSearchIndex();
  });
});
```

---

## Fix 3: Avoid Forced Synchronous Layout (Layout Thrashing)

Reading layout properties (`offsetWidth`, `getBoundingClientRect`) after writing DOM forces synchronous layout.

```js
// Bad: layout thrash — read after write in a loop
elements.forEach(el => {
  el.style.width = el.parentElement.offsetWidth + 'px'; // write then read
});

// Good: batch reads first, then writes
const widths = elements.map(el => el.parentElement.offsetWidth); // all reads
elements.forEach((el, i) => {
  el.style.width = widths[i] + 'px'; // all writes
});
```

---

## Fix 4: Optimize React State Updates Causing INP

Heavy re-renders triggered by user interactions inflate INP.

```jsx
// Bad: expensive calculation blocks render
function SearchResults({ query }) {
  const results = filterItems(allItems, query); // runs on every keystroke
  return <List items={results} />;
}

// Good: memoize expensive computation
function SearchResults({ query }) {
  const results = useMemo(() => filterItems(allItems, query), [query]);
  return <List items={results} />;
}
```

```jsx
// Bad: input update re-renders whole tree
function Form() {
  const [value, setValue] = useState('');
  return (
    <>
      <HeavyComponent /> {/* re-renders on every keystroke */}
      <input onChange={e => setValue(e.target.value)} />
    </>
  );
}

// Good: isolate the state to avoid cascade
function Form() {
  return (
    <>
      <HeavyComponent />   {/* never re-renders */}
      <ControlledInput />  {/* state lives here only */}
    </>
  );
}
```

---

## Fix 5: Use `startTransition` for Non-Urgent Updates (React 18+)

Mark expensive state updates as "transitions" so React can keep the UI responsive.

```jsx
import { startTransition, useState } from 'react';

function SearchPage() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);

  function handleInput(e) {
    // Urgent: update the input value immediately
    setQuery(e.target.value);

    // Non-urgent: defer the expensive search results render
    startTransition(() => {
      setResults(search(e.target.value));
    });
  }

  return (
    <>
      <input value={query} onChange={handleInput} />
      <ResultsList results={results} />
    </>
  );
}
```

---

## Fix 6: Debounce / Throttle Rapid Events

```js
// Debounce: wait for user to stop typing
const handleSearch = debounce((query) => {
  fetchResults(query);
}, 300);

// Throttle: limit scroll/resize handlers
const handleScroll = throttle(() => {
  updateScrollIndicator();
}, 100);
```

---

## Measuring INP

**web-vitals library:**
```js
import { onINP } from 'web-vitals';
onINP(({ value, entries }) => {
  console.log('INP:', value + 'ms');
  const worst = entries[entries.length - 1];
  console.log('Worst interaction:', worst.name, worst.processingStart - worst.startTime);
});
```

**Chrome DevTools → Performance panel:**
- Interact with the page while recording
- Look for long "Event: click" or "Event: keydown" tasks in the main thread flame chart
- "Total blocking time" in the summary correlates with bad INP

**Chrome DevTools → Performance Insights:**
- "Slow interactions" section lists INP candidates with full breakdown
