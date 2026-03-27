# React Performance

React-specific patterns that cause slow renders, high INP, and inflated TTI.

---

## Fix 1: Memoize Expensive Computations

```jsx
import { useMemo } from 'react';

// Bad: runs filterItems on every render
function ProductList({ products, filter }) {
  const filtered = filterItems(products, filter); // expensive
  return <List items={filtered} />;
}

// Good: only recalculates when inputs change
function ProductList({ products, filter }) {
  const filtered = useMemo(
    () => filterItems(products, filter),
    [products, filter]
  );
  return <List items={filtered} />;
}
```

**When to use `useMemo`:** Computation takes > ~1ms, or returns a new array/object used as a dependency elsewhere.
**When NOT to use `useMemo`:** Simple property access, string concatenation — memoization overhead outweighs savings.

---

## Fix 2: Prevent Unnecessary Child Re-Renders

```jsx
import { memo, useCallback } from 'react';

// Bad: new function reference on every render → child always re-renders
function Parent() {
  const [count, setCount] = useState(0);
  const handleClick = () => doSomething(); // new ref each render

  return <Child onClick={handleClick} />;
}

// Good: stable reference with useCallback
function Parent() {
  const [count, setCount] = useState(0);
  const handleClick = useCallback(() => doSomething(), []); // stable

  return <Child onClick={handleClick} />;
}

// Wrap Child in memo to skip re-render when props unchanged
const Child = memo(function Child({ onClick }) {
  return <button onClick={onClick}>Click me</button>;
});
```

**Rule:** `useCallback` + `memo` only help when: (a) the component is actually slow, and (b) props are truly stable. Profile first.

---

## Fix 3: Virtualize Long Lists

Rendering 1000+ items kills performance. Only render what's visible.

```jsx
// Bad: renders all 10,000 rows
function UserList({ users }) {
  return (
    <ul>
      {users.map(user => <UserRow key={user.id} user={user} />)}
    </ul>
  );
}

// Good: react-window renders only ~20 visible rows
import { FixedSizeList } from 'react-window';

function UserList({ users }) {
  return (
    <FixedSizeList
      height={600}
      itemCount={users.length}
      itemSize={60}
      width="100%"
    >
      {({ index, style }) => (
        <UserRow style={style} user={users[index]} />
      )}
    </FixedSizeList>
  );
}
```

**Libraries:** `react-window` (lightweight), `react-virtual` (TanStack, headless), `react-virtuoso` (auto-sizing).

---

## Fix 4: Avoid State that Causes Waterfall Re-Renders

```jsx
// Bad: single state object causes full re-render on any field change
function Form() {
  const [form, setForm] = useState({ name: '', email: '', bio: '' });
  return (
    <input onChange={e => setForm(prev => ({ ...prev, name: e.target.value }))} />
    // Every keystroke re-renders the entire Form tree
  );
}

// Good: split state so only the changed field's subtree re-renders
function Form() {
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');
  // Or: use useReducer for complex forms
}
```

---

## Fix 5: Lazy Load Heavy Components

```jsx
import { lazy, Suspense } from 'react';

// Heavy third-party: charts, editors, maps — don't load upfront
const Chart = lazy(() => import('recharts').then(m => ({ default: m.LineChart })));
const MapView = lazy(() => import('./MapView'));

function Dashboard({ showMap }) {
  return (
    <>
      <Suspense fallback={<ChartSkeleton />}>
        <Chart data={data} />
      </Suspense>
      {showMap && (
        <Suspense fallback={<MapSkeleton />}>
          <MapView />
        </Suspense>
      )}
    </>
  );
}
```

---

## Fix 6: Use React Profiler to Find Actual Bottlenecks

Don't guess — profile first.

```jsx
import { Profiler } from 'react';

function onRender(id, phase, actualDuration) {
  if (actualDuration > 16) { // > 1 frame at 60fps
    console.warn(`Slow render: ${id} (${phase}) took ${actualDuration.toFixed(1)}ms`);
  }
}

<Profiler id="ProductList" onRender={onRender}>
  <ProductList items={items} />
</Profiler>
```

**React DevTools Profiler:** Record interactions and see a flame chart of which components took longest to render.

---

## Fix 7: Context Performance — Avoid Unnecessary Re-Renders

Every consumer of a context re-renders when the context value changes.

```jsx
// Bad: one context for everything — every consumer re-renders on any change
const AppContext = createContext();
function AppProvider({ children }) {
  const [user, setUser] = useState(null);
  const [theme, setTheme] = useState('light');
  const [cart, setCart] = useState([]);
  return <AppContext.Provider value={{ user, theme, cart, setUser, setTheme, setCart }}>{children}</AppContext.Provider>;
}

// Good: split contexts by update frequency
const UserContext = createContext();
const ThemeContext = createContext();
const CartContext = createContext();

// Or: use a state manager like Zustand for fine-grained subscriptions
import { create } from 'zustand';
const useStore = create(set => ({
  user: null,
  theme: 'light',
  cart: [],
  setUser: (user) => set({ user }),
}));
// Components only re-render when their subscribed slice changes
const user = useStore(state => state.user);
```

---

## Fix 8: Transition Non-Urgent UI Updates (React 18+)

```jsx
import { useTransition, useState } from 'react';

function SearchPage() {
  const [query, setQuery] = useState('');
  const [isPending, startTransition] = useTransition();

  return (
    <>
      <input
        value={query}
        onChange={e => {
          setQuery(e.target.value); // urgent: update input immediately
          startTransition(() => {
            performSearch(e.target.value); // non-urgent: can be interrupted
          });
        }}
      />
      {isPending && <Spinner />}
    </>
  );
}
```

---

## Measuring React Performance

**React DevTools Profiler** (browser extension):
1. Open React DevTools → Profiler tab
2. Click "Record"
3. Interact with the slow part of the UI
4. Click "Stop" — inspect the flame chart for long renders

**Inline profiling:**
```js
console.time('render');
// trigger render
console.timeEnd('render');
```

**why-did-you-render** library:
```js
// In dev only: logs unnecessary re-renders
import React from 'react';
import whyDidYouRender from '@welldone-software/why-did-you-render';
whyDidYouRender(React, { trackAllPureComponents: true });
```
