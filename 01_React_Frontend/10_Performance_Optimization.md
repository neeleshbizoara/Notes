# Section 10: Performance Optimization (9 Questions & Answers)



---

## Q87. How do you identify and fix performance bottlenecks in a React application?

**Answer:**

**Step 1: Identify the problem using tools:**

```
Browser DevTools → Performance tab → Record → Interact → Stop → Analyze
```

| Tool | What It Shows |
|---|---|
| React DevTools Profiler | Which components re-render, how long each takes |
| Chrome Performance tab | JavaScript execution, layout, paint times |
| Lighthouse | Overall performance score, LCP, FID, CLS |
| `React.Profiler` component | Programmatic profiling in code |
| `why-did-you-render` library | Logs unnecessary re-renders |

**Step 2: Common bottlenecks and fixes:**

```jsx
// ❌ Bottleneck 1: Unnecessary re-renders
function Dashboard({ user }) {
  const transactions = getTransactions();  // Recalculates every render!
  return <TransactionList data={transactions} />;
}

// ✅ Fix: useMemo
function Dashboard({ user }) {
  const transactions = useMemo(() => getTransactions(), [user.accountId]);
  return <TransactionList data={transactions} />;
}
```

```jsx
// ❌ Bottleneck 2: Creating new objects/functions every render
function Parent() {
  return <ExpensiveChild style={{ padding: 20 }} onClick={() => handleClick()} />;
  // New style object + new function on EVERY render → child always re-renders
}

// ✅ Fix: Memoize
const style = { padding: 20 };  // Move outside or useMemo
function Parent() {
  const handleClick = useCallback(() => { ... }, []);
  return <ExpensiveChild style={style} onClick={handleClick} />;
}
```

```jsx
// ❌ Bottleneck 3: Large list rendering
function TransactionHistory({ transactions }) {
  return transactions.map(t => <TransactionRow key={t.id} data={t} />);
  // 10,000 DOM nodes = laggy scrolling
}

// ✅ Fix: Virtualization (only render visible rows)
import { FixedSizeList } from 'react-window';

function TransactionHistory({ transactions }) {
  return (
    <FixedSizeList height={600} itemCount={transactions.length} itemSize={60}>
      {({ index, style }) => (
        <TransactionRow style={style} data={transactions[index]} />
      )}
    </FixedSizeList>
  );
}
```

**Using React.Profiler in code:**
```jsx
<Profiler id="Dashboard" onRender={(id, phase, actualDuration) => {
  if (actualDuration > 16) {  // More than 1 frame (16ms)
    console.warn(`${id} took ${actualDuration}ms to render`);
  }
}}>
  <Dashboard />
</Profiler>
```

---

## Q88. Explain `React.memo`, `useMemo`, and `useCallback`. When do they hurt performance?

**Answer:**

| Tool | What It Memoizes | Purpose |
|---|---|---|
| `React.memo` | Entire component render | Skip re-rendering if props unchanged |
| `useMemo` | Computed value | Skip recalculating expensive values |
| `useCallback` | Function reference | Maintain stable function reference |

**When they HURT performance:**

```jsx
// ❌ HURT: Memoizing cheap operations
const fullName = useMemo(() => first + ' ' + last, [first, last]);
// The comparison of deps costs MORE than just doing the concatenation!

// ❌ HURT: Memoizing a component that always gets new props
const MemoizedList = React.memo(TransactionList);
// If parent passes a new array every time → memo comparison is wasted work

// ❌ HURT: useCallback without React.memo child
function Parent() {
  const handler = useCallback(() => doSomething(), []);
  return <Child onClick={handler} />;
  // If Child is NOT wrapped in React.memo, this useCallback is useless!
  // Child re-renders because Parent re-renders, regardless of stable function ref
}
```

**When they HELP:**
```jsx
// ✅ HELP: React.memo on expensive component
const TransactionRow = React.memo(({ data }) => {
  return (
    <div> {/* Complex rendering with formatting, icons, etc. */}
      <span>{data.description}</span>
      <span>{formatCurrency(data.amount)}</span>
      <StatusBadge status={data.status} />
    </div>
  );
});

// ✅ HELP: useMemo on expensive calculation
const chartData = useMemo(() => {
  return processTransactions(transactions);  // Aggregate 10,000 transactions
}, [transactions]);

// ✅ HELP: useCallback + React.memo combination
const handleClick = useCallback((id) => {
  dispatch(selectTransaction(id));
}, [dispatch]);

const MemoizedRow = React.memo(TransactionRow);
// Now MemoizedRow only re-renders when its props change
// And handleClick reference is stable, so it doesn't cause re-render
```

**Rule of thumb:** Don't memoize by default. Profile first, then optimize what's actually slow.

---

## Q88b. How do you prevent unnecessary re-renders of sibling components?

**Answer:**

When a parent re-renders, **all its children re-render too** — even siblings that don't care about the changed state. Here are three techniques to prevent this, in order of preference:

**Technique 1: Keep State Local (Zero Cost)**

Move state into the component that actually uses it, so siblings are never affected.

```jsx
// ❌ BAD — typing re-renders Header + ProductList
function Page() {
  const [search, setSearch] = useState('');
  return (
    <div>
      <Header />                          {/* 🔴 re-renders */}
      <ProductList />                     {/* 🔴 re-renders */}
      <input value={search} onChange={e => setSearch(e.target.value)} />
    </div>
  );
}

// ✅ GOOD — state isolated, siblings untouched
function SearchBox() {
  const [search, setSearch] = useState('');
  return <input value={search} onChange={e => setSearch(e.target.value)} />;
}

function Page() {
  return (
    <div>
      <Header />           {/* ✅ not affected */}
      <ProductList />      {/* ✅ not affected */}
      <SearchBox />
    </div>
  );
}
```

**Technique 2: Children Pattern — Lift Only What's Needed (Zero Cost)**

When the parent must own some state, extract the stateful logic into a wrapper and pass unrelated siblings as `children` so React skips them.

```jsx
// ❌ BAD — ScrollTracker state re-renders both children
function Layout() {
  const [scrollY, setScrollY] = useState(0);
  useEffect(() => {
    const handler = () => setScrollY(window.scrollY);
    window.addEventListener('scroll', handler);
    return () => window.removeEventListener('scroll', handler);
  }, []);
  return (
    <div>
      <Navbar position={scrollY} />
      <Sidebar />            {/* 🔴 re-renders on every scroll */}
      <MainContent />        {/* 🔴 re-renders on every scroll */}
    </div>
  );
}

// ✅ GOOD — extract the stateful part, pass siblings as children
function ScrollTracker({ children }) {
  const [scrollY, setScrollY] = useState(0);
  useEffect(() => {
    const handler = () => setScrollY(window.scrollY);
    window.addEventListener('scroll', handler);
    return () => window.removeEventListener('scroll', handler);
  }, []);
  return (
    <div>
      <Navbar position={scrollY} />
      {children}              {/* ✅ children are the SAME JSX reference */}
    </div>
  );
}

function Layout() {
  return (
    <ScrollTracker>
      <Sidebar />            {/* ✅ not re-rendered on scroll */}
      <MainContent />        {/* ✅ not re-rendered on scroll */}
    </ScrollTracker>
  );
}
```

**Why it works:** `children` is created in `Layout`, which doesn't re-render. The JSX reference stays the same, so React skips reconciling those subtrees.

**Step-by-step breakdown:**

1. `setScrollY` fires → `ScrollTracker` re-renders
2. But `children` (i.e., `<Sidebar />` and `<MainContent />`) were **created in `Layout`**, not in `ScrollTracker`
3. `Layout` has **no state**, so it **never re-renders** → the JSX objects for `<Sidebar />` and `<MainContent />` are the **exact same reference** as last time
4. React sees the same reference and says: "nothing changed here, skip it"

**Analogy:** Imagine you hand someone a sealed envelope (`children`). When that person redecorates their room (re-renders), the envelope you gave them hasn't changed — it's still the same sealed envelope. So they don't need to open and re-examine it. The `children` prop is just a variable passed in from the parent. If the parent (`Layout`) didn't re-render, that variable is the same object in memory, and React skips it entirely.

**Technique 3: React.memo (Shallow Compare Overhead)**

When you can't restructure, wrap the sibling to bail out of re-renders:

```jsx
const Sidebar = React.memo(function Sidebar({ items }) {
  console.log('Sidebar rendered');
  return <ul>{items.map(i => <li key={i.id}>{i.name}</li>)}</ul>;
});

function Page() {
  const [search, setSearch] = useState('');
  const items = useMemo(() => getItems(), []); // stable reference
  return (
    <div>
      <Sidebar items={items} />   {/* ✅ memo skips if items ref unchanged */}
      <input value={search} onChange={e => setSearch(e.target.value)} />
    </div>
  );
}
```

**Gotcha:** `React.memo` does shallow comparison. If you pass a new object/array every render, it defeats the purpose. Pair with `useMemo`/`useCallback`.

**Decision Flowchart:**
```
Sibling re-rendering unnecessarily?
  │
  ├─ Can state move INTO the component that uses it?
  │     YES → Technique 1: Localize state
  │
  ├─ Can you restructure so siblings are children?
  │     YES → Technique 2: Children pattern
  │
  └─ Neither possible?
        → Technique 3: React.memo + useMemo/useCallback
```

**Summary:**

| Technique | Cost | When to use |
|---|---|---|
| Localize state | Zero cost | State only needed in one component |
| Children pattern | Zero cost | Parent owns state but siblings are unrelated |
| React.memo | Shallow compare overhead | Can't restructure; sibling receives stable props |

**Rule:** Prefer restructuring (techniques 1 & 2) over memoization (technique 3). Restructuring solves the problem architecturally; memo is a patch.

---

## Q89. What are Web Vitals (LCP, FID, CLS)? How do you measure and improve them?

**Answer:**

| Metric | Full Name | What It Measures | Good | Poor |
|---|---|---|---|---|
| **LCP** | Largest Contentful Paint | When the biggest visible element finishes loading | < 2.5s | > 4s |
| **FID** | First Input Delay | Time from user's first click to browser's response | < 100ms | > 300ms |
| **CLS** | Cumulative Layout Shift | How much the page content shifts unexpectedly | < 0.1 | > 0.25 |
| **INP** | Interaction to Next Paint | Responsiveness to ALL interactions (replaced FID) | < 200ms | > 500ms |

**Measuring:**
```jsx
// Using web-vitals library
import { onLCP, onFID, onCLS, onINP } from 'web-vitals';

onLCP(metric => console.log('LCP:', metric.value));
onFID(metric => console.log('FID:', metric.value));
onCLS(metric => console.log('CLS:', metric.value));
onINP(metric => console.log('INP:', metric.value));
```

**Improving LCP (load faster):**
```jsx
// 1. Lazy load below-the-fold content
const TransactionHistory = lazy(() => import('./TransactionHistory'));

// 2. Preload critical resources
<link rel="preload" href="/fonts/bank-font.woff2" as="font" crossOrigin />

// 3. Optimize images
<img src="hero.webp" loading="eager" fetchPriority="high" />  {/* Above fold */}
<img src="promo.webp" loading="lazy" />  {/* Below fold */}

// 4. Server-side render the above-fold content
```

**Improving CLS (prevent layout shifts):**
```jsx
// 1. Always set dimensions on images
<img src="logo.png" width={200} height={50} />

// 2. Reserve space for dynamic content
.skeleton {
  height: 200px;  /* Same height as loaded content */
  background: #f0f0f0;
  animation: pulse 1.5s infinite;
}

// 3. Don't inject content above existing content dynamically
// ❌ Bad: Banner appears above content → pushes everything down
// ✅ Good: Reserve space for banner from the start
```

**Improving INP (respond faster):**
```jsx
// Use useTransition for non-urgent updates
const [isPending, startTransition] = useTransition();

function handleFilter(filterValue) {
  setFilter(filterValue);  // Urgent: update input immediately
  startTransition(() => {
    setFilteredData(applyFilter(data, filterValue));  // Non-urgent: can wait
  });
}
```

---

## Q90. How would you optimize a dashboard rendering 10,000+ rows of financial data?

**Answer:**

**Solution: Virtualization (only render visible rows)**

```jsx
// Using react-window (lightweight)
import { FixedSizeList as List } from 'react-window';

function TransactionTable({ transactions }) {
  const Row = ({ index, style }) => {
    const txn = transactions[index];
    return (
      <div style={style} className="transaction-row">
        <span>{txn.date}</span>
        <span>{txn.description}</span>
        <span className={txn.amount >= 0 ? 'credit' : 'debit'}>
          ₹{Math.abs(txn.amount).toLocaleString()}
        </span>
      </div>
    );
  };

  return (
    <List
      height={600}           // Visible area height
      itemCount={transactions.length}  // 10,000+ items
      itemSize={50}          // Each row height
      width="100%"
    >
      {Row}
    </List>
  );
  // Only ~12 rows are in the DOM at any time (600/50 = 12)
  // Instead of 10,000 DOM nodes!
}
```

**For variable height rows:**
```jsx
import { VariableSizeList } from 'react-window';

<VariableSizeList
  height={600}
  itemCount={transactions.length}
  itemSize={index => transactions[index].hasDetails ? 100 : 50}
>
  {Row}
</VariableSizeList>
```

**Additional optimizations:**
```jsx
// 1. Memoize row component
const Row = React.memo(({ data, index, style }) => {
  const txn = data[index];
  return <div style={style}>{txn.description}</div>;
});

// 2. Use itemData to avoid creating new objects
<List itemData={transactions}>
  {Row}
</List>

// 3. Pagination as alternative (simpler)
function PaginatedTable({ transactions }) {
  const [page, setPage] = useState(1);
  const pageSize = 50;
  const currentData = transactions.slice((page - 1) * pageSize, page * pageSize);
  // Only 50 rows in DOM
}
```

**Performance comparison:**

| Approach | DOM Nodes | Scroll Performance | Initial Load |
|---|---|---|---|
| Render all 10,000 | 10,000 | Terrible | Slow (seconds) |
| Virtualization | ~15 | Smooth | Instant |
| Pagination (50/page) | 50 | N/A (click to navigate) | Instant |

---

## Q91. Explain virtualization (react-window, react-virtualized). When is it needed?

**Answer:**

**Virtualization** = Only render items that are currently visible in the viewport. As user scrolls, old items are removed and new items are rendered.

```
Full list: [1] [2] [3] [4] [5] [6] [7] [8] [9] [10] ... [10000]

Viewport shows:        [4] [5] [6] [7] [8]
                     ↑ Only these 5 items are in the DOM! ↑

User scrolls down:     [7] [8] [9] [10] [11]
                     ↑ Items 4,5,6 removed, 9,10,11 added ↑
```

**react-window vs react-virtualized:**

| Feature | react-window | react-virtualized |
|---|---|---|
| Bundle size | ~6KB | ~35KB |
| API | Simple | Feature-rich |
| Features | Basic lists and grids | Tables, masonry, collection, multi-grid |
| Performance | Slightly faster | Good |
| Recommendation | Use by default | Use when you need extras |

**When virtualization is needed:**
- Lists with 500+ items
- Tables with many rows
- Infinite scroll feeds
- Large data grids

**When it's NOT needed:**
- Lists with < 100 items (just render them all)
- Static content that doesn't change
- Simple dropdowns

---

## Q92. What is the impact of bundle size on performance? How do you analyze and reduce it?

**Answer:**

**Impact:** Every KB of JavaScript must be downloaded, parsed, compiled, and executed. On 3G:
- 100KB JS = ~1 second load time
- 500KB JS = ~5 seconds load time
- 1MB JS = ~10+ seconds load time

**Analyze with webpack-bundle-analyzer:**
```bash
npm install webpack-bundle-analyzer --save-dev

# Add to webpack config:
const { BundleAnalyzerPlugin } = require('webpack-bundle-analyzer');
plugins: [new BundleAnalyzerPlugin()]

# Or with Create React App:
npx source-map-explorer build/static/js/*.js
```

**Common size reduction techniques:**

```jsx
// 1. Import only what you need
import { format } from 'date-fns';        // ✅ ~2KB
import moment from 'moment';               // ❌ ~300KB

import { debounce } from 'lodash-es';      // ✅ ~1KB (tree-shakeable)
import _ from 'lodash';                     // ❌ ~70KB

// 2. Dynamic imports for large libraries
const Chart = lazy(() => import('recharts'));  // Load only when needed

// 3. Replace heavy libraries with lighter ones
// moment.js (300KB) → date-fns (tree-shakeable) or dayjs (2KB)
// lodash (70KB) → lodash-es or native JS
// axios (14KB) → native fetch

// 4. Use production builds
// React dev: ~800KB → React prod: ~130KB

// 5. Compression
// Gzip reduces JS by ~70%
// 500KB → ~150KB over network
```

---

## Q93. How do you implement efficient pagination vs infinite scroll for transaction history?

**Answer:**

**Pagination:**
```jsx
function PaginatedTransactions({ accountId }) {
  const [page, setPage] = useState(1);
  const [totalPages, setTotalPages] = useState(0);
  const pageSize = 20;

  const { data, loading } = useFetch(
    `/api/transactions?account=${accountId}&page=${page}&size=${pageSize}`
  );

  return (
    <div>
      <TransactionList data={data?.transactions || []} />
     
      <div className="pagination">
        <button
          disabled={page === 1}
          onClick={() => setPage(p => p - 1)}
        >
          Previous
        </button>
        <span>Page {page} of {totalPages}</span>
        <button
          disabled={page === totalPages}
          onClick={() => setPage(p => p + 1)}
        >
          Next
        </button>
      </div>
    </div>
  );
}
```

**Infinite Scroll:**
```jsx
function InfiniteTransactions({ accountId }) {
  const [transactions, setTransactions] = useState([]);
  const [page, setPage] = useState(1);
  const [hasMore, setHasMore] = useState(true);
  const [loading, setLoading] = useState(false);
  const observerRef = useRef();
  const lastItemRef = useRef();

  // Load more when last item becomes visible
  useEffect(() => {
    if (loading || !hasMore) return;

    observerRef.current = new IntersectionObserver(entries => {
      if (entries[0].isIntersecting) {
        loadMore();
      }
    });

    if (lastItemRef.current) {
      observerRef.current.observe(lastItemRef.current);
    }

    return () => observerRef.current?.disconnect();
  }, [transactions, loading, hasMore]);

  async function loadMore() {
    setLoading(true);
    const response = await fetch(
      `/api/transactions?account=${accountId}&page=${page}&size=20`
    );
    const data = await response.json();
   
    setTransactions(prev => [...prev, ...data.transactions]);
    setHasMore(data.hasNextPage);
    setPage(p => p + 1);
    setLoading(false);
  }

  return (
    <div>
      {transactions.map((txn, index) => (
        <TransactionRow
          key={txn.id}
          data={txn}
          ref={index === transactions.length - 1 ? lastItemRef : null}
        />
      ))}
      {loading && <Spinner />}
      {!hasMore && <p>No more transactions</p>}
    </div>
  );
}
```

**When to use which:**

| Pagination | Infinite Scroll |
|---|---|
| User needs to jump to specific page | User browses sequentially |
| Important to know total count | Casual browsing |
| User needs "go back to page 3" | Social feeds, activity log |
| **Bank statement (user wants specific month)** | Recent transactions feed |
| Better for accessibility | Harder for accessibility |
| Better for SEO | Worse for SEO |

---

## Q94. What is the React Profiler? Walk me through debugging a slow component.

**Answer:**

**React DevTools Profiler** shows you exactly which components render, how long they take, and why they re-rendered.

**Step-by-step debugging:**

```
1. Open React DevTools → Profiler tab
2. Click "Record" button
3. Perform the slow action in your app
4. Click "Stop" button
5. Analyze the flame chart
```

**What you'll see:**
```
Flame chart:
┌──── App (2ms) ────────────────────────────────┐
│  ┌── Header (0.5ms) ──┐  ┌── Dashboard (45ms) ──────────────────┐  │
│  │                     │  │  ┌── AccountList (3ms) ┐  ┌── Chart (38ms) ──┐  │
│  │                     │  │  │                     │  │   ← SLOW!        │  │
│  └─────────────────────┘  └──────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────────┘
```

**Common findings and fixes:**

```jsx
// Finding 1: Component re-renders when props haven't changed
// Fix: React.memo
const Chart = React.memo(function Chart({ data }) {
  // Expensive chart rendering
});

// Finding 2: Component is slow because of expensive calculation in render
// Fix: useMemo
function Dashboard({ transactions }) {
  const chartData = useMemo(() =>
    aggregateByMonth(transactions), // Takes 30ms
    [transactions]
  );
}

// Finding 3: Parent re-render causes ALL children to re-render
// Fix: Move state down closer to where it's used
// Before: State in App → everything re-renders
// After: State in SearchBar → only SearchBar re-renders
```

**Programmatic Profiler component:**
```jsx
function onRenderCallback(id, phase, actualDuration, baseDuration, startTime, commitTime) {
  if (actualDuration > 16) {
    console.warn(`⚠️ Slow render: ${id} took ${actualDuration.toFixed(1)}ms (${phase})`);
    // Send to monitoring service in production
  }
}

<Profiler id="TransactionTable" onRender={onRenderCallback}>
  <TransactionTable data={data} />
</Profiler>
```

---

## Quick Revision Checklist

- [ ] Performance debugging tools (React Profiler, Chrome DevTools, Lighthouse)
- [ ] React.memo / useMemo / useCallback — when they help vs hurt
- [ ] Preventing sibling re-renders — localize state → children pattern → React.memo
- [ ] Web Vitals: LCP, FID/INP, CLS — how to measure and improve
- [ ] Virtualization for 10,000+ row tables (react-window)
- [ ] react-window (6KB, simple) vs react-virtualized (35KB, feature-rich)
- [ ] Bundle size analysis and reduction techniques
- [ ] Pagination vs Infinite Scroll trade-offs
- [ ] React Profiler step-by-step debugging process

