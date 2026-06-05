# Section 1: React Core (25 Questions & Answers)



---

## Q1. Explain the Virtual DOM and React's reconciliation algorithm. How does React 18's concurrent rendering change this?

**Answer:**

The **Virtual DOM** is a lightweight JavaScript copy of the real DOM. When something changes in your app, React first updates this virtual copy, then compares it with the previous version (this comparing is called **"diffing"**), and finally updates only the changed parts in the real DOM. This process is called **reconciliation**.

**Simple Example:**
```
Imagine you have a list of 100 items. You change item #5.
- Without Virtual DOM: Browser re-renders all 100 items
- With Virtual DOM: React compares old vs new, finds only item #5 changed, updates only that one
```

**How reconciliation works:**
1. React creates a new Virtual DOM tree
2. Compares it with the old one (diffing)
3. Calculates the minimum number of changes needed
4. Updates only those parts in the real DOM (this is called "patching")

**React 18's Concurrent Rendering changes:**
- Before React 18: Rendering was **synchronous** — once React starts rendering, it cannot stop until it finishes
- React 18: Rendering is **interruptible** — React can pause, work on something more urgent (like user typing), and come back

```jsx
// React 18 concurrent feature example
import { useTransition } from 'react';

function SearchPage() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [isPending, startTransition] = useTransition();

  function handleChange(e) {
    setQuery(e.target.value);  // This updates immediately (urgent)
    
    startTransition(() => {
      setResults(filterData(e.target.value));  // This can be delayed (not urgent)
    });
  }

  return (
    <div>
      <input value={query} onChange={handleChange} />
      {isPending ? <p>Loading...</p> : <ResultsList results={results} />}
    </div>
  );
}
```

**How to explain in interview:** "Virtual DOM is like a draft copy. You make changes on the draft first, compare it with the original, and then only apply the differences. React 18 made this smarter by allowing React to pause and prioritize urgent updates like user input over heavy calculations."

---

## Q1b. Explain the React 18 Rendering Lifecycle — the two-phase model, where hooks fit, and what you should/shouldn't do in each phase.

**Answer:**

**One-line mental model:** React rendering is a two-phase process — **Render** (pure, interruptible, in memory) and **Commit** (DOM updates, effects) — and in React 18, rendering is concurrent, meaning it can pause, resume, or abandon work to keep the UI responsive.

---

### The Two Phases of Rendering (Core Interview Concept)

```
           RENDER PHASE                    COMMIT PHASE
    (Pure · Interruptible)            (DOM · Not Interruptible)
    ┌──────────────────────┐          ┌──────────────────────────┐
    │                      │          │                          │
    │  Run function comp.  │          │  Apply DOM mutations     │
    │  Calculate new JSX   │          │  Run useLayoutEffect     │
    │  Build WIP Fiber     │   ──►    │  Browser paints screen   │
    │  tree in memory      │          │  Run useEffect           │
    │                      │          │                          │
    │  ⚠️ NO side effects   │          │  ✅ Side effects go here  │
    │  ⚠️ NO DOM mutations   │          │                          │
    └──────────────────────┘          └──────────────────────────┘
         Can be paused,                   Cannot be interrupted.
         restarted, or                    Once started, the user
         discarded by React               sees the result.
```

**Phase 1 — Render Phase (Pure & Interruptible):**

What happens:
- React calls your function components (runs the function body)
- Calculates what the UI **should** look like
- Builds a **Work-In-Progress (WIP) Fiber tree** — a draft version in memory
- **No DOM updates happen here**
- Can be paused, restarted, or discarded by React

```
React keeps TWO trees:

  Current Tree              WIP Tree
  ┌──────────┐            ┌──────────┐
  │   App    │            │   App    │
  │  ┌────┐  │            │  ┌────┐  │
  │  │Head│  │            │  │Head│  │
  │  ├────┤  │    ──►     │  ├────┤  │  ← React builds this
  │  │Body│  │  (diffing)  │  │Body│  │    in memory first
  │  └────┘  │            │  └────┘  │
  └──────────┘            └──────────┘
  (what's on screen)      (draft — not yet visible)
```

Only when the WIP tree is fully ready does React move to the Commit phase.

**Phase 2 — Commit Phase (DOM + Effects):**

What happens (in this exact order):
1. React applies **minimal DOM mutations** (only what changed)
2. Runs `useLayoutEffect` callbacks (synchronously, **before** browser paints)
3. Browser **paints** the screen (user sees the update)
4. Runs `useEffect` callbacks (asynchronously, **after** paint)

This phase is **NOT interruptible** — once React starts committing, it finishes.

---

### Where Hooks Fit in the Lifecycle

```
  ┌─── RENDER PHASE ───┐    ┌────────── COMMIT PHASE ──────────┐
  │                     │    │                                   │
  │  useState           │    │  DOM updated                      │
  │  (reads current     │    │       │                           │
  │   state value)      │    │       ▼                           │
  │                     │    │  useLayoutEffect runs             │
  │  useMemo            │    │  (before paint — measure DOM)     │
  │  (cached calc)      │    │       │                           │
  │                     │    │       ▼                           │
  │  useCallback        │    │  Browser PAINTS screen            │
  │  (cached function)  │    │       │                           │
  │                     │    │       ▼                           │
  │  useContext         │    │  useEffect runs                   │
  │  (reads context)    │    │  (after paint — API calls, subs)  │
  │                     │    │                                   │
  └─────────────────────┘    └───────────────────────────────────┘
```

| Hook | Phase | What It Does |
|---|---|---|
| `useState` | Render | Reads current state; `setState` **schedules** a new render |
| `useMemo` | Render | Caches expensive calculated values |
| `useCallback` | Render | Caches function references |
| `useContext` | Render | Reads nearest context value |
| `useRef` | Render | Reads mutable ref (no re-render on change) |
| `useReducer` | Render | Reads state from reducer; dispatch schedules render |
| `useLayoutEffect` | Commit (before paint) | DOM measurements, scroll position |
| `useEffect` | Commit (after paint) | API calls, subscriptions, logging |

---

### Mount → Update → Unmount (Mental Timeline)

**Mounting (component appears for the first time):**
```
1. Render phase    → React runs your function, builds WIP tree
2. Commit phase    → DOM nodes inserted
3. useLayoutEffect → Runs (before paint)
4. Browser paint   → User sees the component
5. useEffect       → Runs (API calls, subscriptions)
```

**Updating (state/props/context changed):**
```
1. Render phase         → React re-runs your function, builds new WIP tree
2. Commit phase         → DOM diff applied (minimal mutations)
3. Old useEffect cleanup → Runs (cleans up previous effect)
4. useLayoutEffect      → Runs
5. Browser paint        → User sees the update
6. New useEffect        → Runs
```

**Unmounting (component removed):**
```
1. useEffect cleanup    → Runs (cancel subscriptions, abort fetches)
2. useLayoutEffect cleanup → Runs
3. DOM nodes removed
```

---

### Parent–Child Rendering Order

```
  Parent starts render
       │
       ▼
  Child 1 renders
       │
       ▼
  Child 2 renders
       │
       ▼
  Parent render completes
       │
       ▼
  COMMIT — entire tree at once (atomic)
```

React renders **top-down** (parent first, then children). But the **commit happens once for the entire tree** — this avoids partial UI states where the parent is updated but children aren't.

---

### Practical Rules: What to Do and NOT Do in Each Phase

**✅ Safe in render (function body):**
```jsx
function Dashboard({ accounts }) {
  // ✅ Compute derived values
  const totalBalance = accounts.reduce((sum, acc) => sum + acc.balance, 0);
  
  // ✅ Read props and state
  const [filter, setFilter] = useState('all');
  
  // ✅ Memoize expensive calculations
  const filtered = useMemo(() => 
    accounts.filter(a => filter === 'all' || a.type === filter),
    [accounts, filter]
  );

  return <AccountList accounts={filtered} total={totalBalance} />;
}
```

**❌ Never do in render (function body):**
```jsx
function Dashboard() {
  // ❌ API calls (side effect!)
  fetch('/api/accounts');

  // ❌ Subscriptions
  window.addEventListener('scroll', handler);

  // ❌ DOM mutations
  document.title = 'Dashboard';

  // ❌ Generating random values (not pure!)
  const id = Math.random();
}
```

**✅ Side effects belong in effects:**
```jsx
function Dashboard() {
  useEffect(() => {
    // ✅ API calls
    fetchAccounts();

    // ✅ Subscriptions
    const unsub = subscribeToUpdates();
    return () => unsub(); // ✅ Cleanup
  }, []);

  useLayoutEffect(() => {
    // ✅ DOM measurements (rare — only when you need size/position before paint)
    const height = ref.current.getBoundingClientRect().height;
  }, []);
}
```

---

### React 18 Concurrency: Priority-Based Updates

Not all updates are equal in React 18:

```
  HIGH PRIORITY (urgent)           LOW PRIORITY (can wait)
  ┌─────────────────────┐         ┌─────────────────────┐
  │ User typing          │         │ Fetched data render  │
  │ Button clicks        │         │ Heavy list filtering │
  │ Form input           │         │ Analytics updates    │
  └─────────────────────┘         └─────────────────────┘
        │                                 │
        └──── React handles these ────────┘
              FIRST, can pause
              the low-priority ones
```

React can:
- **Pause** a low-priority render mid-way
- **Handle** an urgent input event
- **Resume** (or discard and restart) the paused render

You don't need to use any API for this — React does it internally. But you can **explicitly** mark updates as low priority:

```jsx
import { startTransition } from 'react';

function handleFilter(value) {
  setFilterText(value);           // ← URGENT: update input immediately
  
  startTransition(() => {
    setFilteredResults(filter(data, value));  // ← LOW PRIORITY: can wait
  });
}
```

---

**Interview-ready summary (say this):**

> "In React 18, rendering is split into a **render phase** — where React calculates changes in memory using a work-in-progress tree — and a **commit phase** — where it applies DOM updates and runs effects. Rendering is now concurrent and interruptible, so React can prioritize urgent updates and keep the UI responsive. Side effects always run after commit, never during render."

**Golden rule interviewers love:**
> "Render decides **what**, commit applies **it**, effects react **to it**."

---

## Q2. What is the difference between controlled and uncontrolled components? When would you use each in a financial form?

**Answer:**

**Controlled Component:** React controls the form data. The input value is stored in state and updated through `onChange`.

**Uncontrolled Component:** The DOM itself handles the form data. You use `ref` to get values when needed.

```jsx
// CONTROLLED - React controls the value
function ControlledForm() {
  const [amount, setAmount] = useState('');

  return (
    <input 
      value={amount} 
      onChange={(e) => setAmount(e.target.value)} 
    />
  );
}

// UNCONTROLLED - DOM controls the value
function UncontrolledForm() {
  const amountRef = useRef();

  function handleSubmit() {
    console.log(amountRef.current.value);
  }

  return <input ref={amountRef} defaultValue="" />;
}
```

**When to use what in financial services:**

| Scenario | Use | Reason |
|---|---|---|
| Payment form | Controlled | Need real-time validation (e.g., amount cannot exceed balance) |
| Search filter | Uncontrolled | Simple, no validation needed |
| Loan EMI calculator | Controlled | Need to update calculations as user types |
| File upload (KYC docs) | Uncontrolled | File inputs are always uncontrolled in React |

**Financial form example (Controlled):**
```jsx
function TransferForm() {
  const [amount, setAmount] = useState('');
  const [error, setError] = useState('');

  function handleAmountChange(e) {
    const value = e.target.value;
    // Real-time validation for financial form
    if (value < 0) setError('Amount cannot be negative');
    else if (value > 1000000) setError('Max transfer limit exceeded');
    else setError('');
    setAmount(value);
  }

  return (
    <div>
      <input value={amount} onChange={handleAmountChange} />
      {error && <span className="error">{error}</span>}
    </div>
  );
}
```

**Interview tip:** "For financial applications, I almost always use controlled components because we need strict validation at every keystroke — for example, limiting transfer amounts, formatting currency in real-time, or enforcing input masks for account numbers."

---

## Q2b. Compare form handling patterns: Native Controlled vs Formik vs React Hook Form. When do you use each?

**Answer:**

| Aspect | Native Controlled | Formik | React Hook Form |
|---|---|---|---|
| Control model | 100% controlled (state-driven) | Controlled (abstracted) | Uncontrolled + refs |
| Re-renders on typing | High (every keystroke) | Medium–High | **Very low** |
| Validation | Manual | Built-in (Yup integration) | Built-in (Yup, Zod) |
| Performance | OK for small forms | OK for medium forms | **Best for large forms** |
| Bundle size | 0 KB (built-in) | ~13 KB | ~9 KB |
| Learning curve | Low | Medium | Medium |
| Enterprise usage | Small forms, filters | Legacy enterprise apps | **Modern enterprise apps** |

**Native Controlled (baseline):**
```jsx
function SearchBox() {
  const [query, setQuery] = useState('');
  return <input value={query} onChange={e => setQuery(e.target.value)} />;
}
// ✅ Simple, full React control
// ❌ Re-render on every keystroke, painful for large forms
// Use for: search boxes, filters, small forms
```

**Formik (controlled abstraction):**
```jsx
import { Formik, Form, Field, ErrorMessage } from 'formik';
import * as Yup from 'yup';

const schema = Yup.object({
  amount: Yup.number().min(1).max(1000000).required(),
  toAccount: Yup.string().length(12).required(),
});

function TransferForm() {
  return (
    <Formik
      initialValues={{ amount: '', toAccount: '' }}
      validationSchema={schema}
      onSubmit={(values) => processTransfer(values)}
    >
      <Form>
        <Field name="amount" type="number" />
        <ErrorMessage name="amount" component="span" className="error" />
        <Field name="toAccount" />
        <ErrorMessage name="toAccount" component="span" className="error" />
        <button type="submit">Transfer</button>
      </Form>
    </Formik>
  );
}
// ✅ Structured API, good validation, touched/errors tracking
// ❌ Every keystroke re-renders form — slow on 20+ field forms
// Use for: legacy apps, moderate-size forms
```

**React Hook Form (modern, high-performance):**
```jsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

const schema = z.object({
  amount: z.number().min(1).max(1000000),
  toAccount: z.string().length(12),
});

function TransferForm() {
  const { register, handleSubmit, formState: { errors } } = useForm({
    resolver: zodResolver(schema),
  });

  return (
    <form onSubmit={handleSubmit(processTransfer)}>
      <input {...register('amount', { valueAsNumber: true })} />
      {errors.amount && <span className="error">{errors.amount.message}</span>}
      
      <input {...register('toAccount')} />
      {errors.toAccount && <span className="error">{errors.toAccount.message}</span>}
      
      <button type="submit">Transfer</button>
    </form>
  );
}
// ✅ Inputs are uncontrolled (refs) → minimal re-renders
// ✅ Only re-renders on submit or error state change
// Use for: large enterprise forms, BFSI workflows, loan applications
```

**Why React Hook Form is faster (the key insight):**
```
Native Controlled:                React Hook Form:
  Type "a"  → re-render             Type "a"  → DOM updates directly
  Type "ab" → re-render             Type "ab" → DOM updates directly
  Type "abc"→ re-render             Type "abc"→ DOM updates directly
                                    Submit    → ONE re-render (validation)
  3 re-renders for 3 keystrokes     0 re-renders until submit!
```

**Interview tip:** "Formik is controlled and easier conceptually, but React Hook Form scales better because it relies on uncontrolled inputs and refs, reducing re-renders. For a 20-field loan application form, the difference is noticeable."

---

## Q2c. What are the best practices for connecting controlled forms to Redux?

**Answer:**

**The golden rule:** Keep typing state **local**, sync to Redux only on **meaningful events**.

**❌ Common mistake — storing every keystroke in Redux:**
```jsx
function SearchInput() {
  const dispatch = useDispatch();
  
  return (
    <input onChange={e => dispatch(setSearchQuery(e.target.value))} />
    // ❌ Redux store updates on EVERY keypress
    // ❌ Every useSelector subscriber re-checks
    // ❌ Terrible performance
  );
}
```

**✅ Correct pattern — local state + sync on meaningful events:**
```jsx
// Pattern 1: Sync on submit
function TransferForm() {
  const [amount, setAmount] = useState('');
  const [toAccount, setToAccount] = useState('');
  const dispatch = useDispatch();

  function handleSubmit(e) {
    e.preventDefault();
    dispatch(submitTransfer({ amount, toAccount }));  // ✅ Sync once on submit
  }

  return (
    <form onSubmit={handleSubmit}>
      <input value={amount} onChange={e => setAmount(e.target.value)} />
      <input value={toAccount} onChange={e => setToAccount(e.target.value)} />
      <button type="submit">Transfer</button>
    </form>
  );
}

// Pattern 2: Sync with debounce (for search/filter)
function SearchInput() {
  const [query, setQuery] = useState('');
  const dispatch = useDispatch();

  const debouncedSearch = useMemo(
    () => debounce((value) => dispatch(searchTransactions(value)), 300),
    [dispatch]
  );

  function handleChange(e) {
    setQuery(e.target.value);           // ✅ Local state for instant UI
    debouncedSearch(e.target.value);    // ✅ Redux only after user stops typing
  }

  return <input value={query} onChange={handleChange} />;
}

// Pattern 3: Sync on step change (multi-step form)
function LoanStep1({ onNext }) {
  const [localData, setLocalData] = useState({ name: '', pan: '' });
  const dispatch = useDispatch();

  function goNext() {
    dispatch(saveLoanStep1(localData));  // ✅ Sync when leaving the step
    onNext();
  }

  return (/* form fields using localData */);
}
```

**When form data SHOULD live in Redux:**

| Scenario | Why Redux |
|---|---|
| Multi-step forms (loan application) | Persist data across steps/routes |
| Draft saving across navigation | User navigates away and comes back |
| Shared form state | Two components need the same form data |
| Undo/redo support | Redux time-travel makes this trivial |
| Audit/compliance logging | Every state change is traceable |

**When form data should stay LOCAL:**

| Scenario | Why Local |
|---|---|
| Single-page form | No need for global state |
| Search/filter inputs | Only the search component cares |
| Validation-in-progress state | `isValid`, `touched` — UI-only concerns |
| Sensitive fields (passwords) | Don't persist in Redux/DevTools |

**Interview tip:** "I keep typing state local and sync to Redux only on meaningful events — submit, step change, or debounce. This keeps forms responsive and Redux state clean."

---

## Q2d. How do you optimize performance for controlled inputs in React 18?

**Answer:**

Controlled inputs can cause lag on large pages because every keystroke triggers a re-render. Here are four techniques to fix this, from simplest to most advanced:

**Technique 1: Localize state (most important):**
```jsx
// ❌ BAD — input state in parent re-renders entire page
function Page() {
  const [search, setSearch] = useState('');
  return (
    <div>
      <Header />           {/* 🔴 re-renders on every keystroke */}
      <TransactionTable /> {/* 🔴 re-renders on every keystroke */}
      <input value={search} onChange={e => setSearch(e.target.value)} />
    </div>
  );
}

// ✅ GOOD — extract input into its own component
function SearchBox({ onSearch }) {
  const [search, setSearch] = useState('');
  return (
    <input
      value={search}
      onChange={e => {
        setSearch(e.target.value);
        onSearch(e.target.value);
      }}
    />
  );
}
// Only SearchBox re-renders on typing. Header and Table are untouched.
```

**Technique 2: Debounce expensive downstream work:**
```jsx
function SearchBox() {
  const [query, setQuery] = useState('');
  const dispatch = useDispatch();

  const debouncedDispatch = useCallback(
    debounce((value) => dispatch(searchAPI(value)), 300),
    [dispatch]
  );

  return (
    <input
      value={query}
      onChange={e => {
        setQuery(e.target.value);         // ✅ Instant UI update
        debouncedDispatch(e.target.value); // ✅ API call only after 300ms pause
      }}
    />
  );
}
```

**Technique 3: `startTransition` for heavy rendering (React 18):**
```jsx
function FilterableTable({ data }) {
  const [filter, setFilter] = useState('');
  const [filteredData, setFilteredData] = useState(data);

  function handleChange(e) {
    setFilter(e.target.value);                    // ← URGENT: update input now

    startTransition(() => {
      setFilteredData(filterLargeDataset(data, e.target.value));  // ← LOW PRIORITY
    });
  }

  return (
    <div>
      <input value={filter} onChange={handleChange} />
      {/* Table renders when React has time, input stays responsive */}
      <table>{filteredData.map(/* ... */)}</table>
    </div>
  );
}
```

**Technique 4: `React.memo` for siblings (when restructuring isn't possible):**
```jsx
const TransactionTable = React.memo(function TransactionTable({ data }) {
  return <table>{data.map(/* ... */)}</table>;
});

const Header = React.memo(function Header() {
  return <header>Banking Dashboard</header>;
});

// Now even if parent re-renders, these skip if their props haven't changed
```

**Decision flow:**
```
Input causes lag?
  │
  ├─ Can you move input state into its own component?
  │     YES → Technique 1 (localize state) — zero cost
  │
  ├─ Is the lag from an API call or dispatch?
  │     YES → Technique 2 (debounce) — 300ms delay on the expensive part
  │
  ├─ Is the lag from rendering a huge list/table?
  │     YES → Technique 3 (startTransition) — React prioritizes input
  │
  └─ Siblings re-rendering unnecessarily?
        YES → Technique 4 (React.memo) — shallow compare cost
```

**Interview tip:** "Controlled inputs are fast if state is localized, expensive work is deferred with debounce or startTransition, and sibling re-renders are prevented with composition or memo. I optimize in that order — localize first, memo last."

---

## Q3. Explain React Fiber architecture. Why was it introduced?

**Answer:**

**React Fiber** is the internal engine of React (introduced in React 16). It completely rewrote how React processes updates.

**Problem with old React (Stack Reconciler):**
- It worked like a **stack** — once it started rendering, it had to finish everything in one go
- If your component tree was big, the browser would freeze (no user interaction possible)

**Fiber solves this by:**
- Breaking rendering work into small **units of work** (called "fibers")
- Each component gets its own fiber node
- React can **pause**, **resume**, and **prioritize** work

**Simple analogy:**
```
Old React = Reading a 500-page book in one sitting (can't stop)
Fiber React = Reading in chapters (can stop, do something urgent, come back)
```

**Key features of Fiber:**
1. **Incremental rendering** — split work into chunks
2. **Priority-based updates** — user clicks are more important than data fetching
3. **Ability to pause and resume** work
4. **Better error handling** — error boundaries became possible

```jsx
// Fiber enables features like Suspense
function App() {
  return (
    <Suspense fallback={<Loading />}>
      <AccountDashboard />  {/* Can pause while loading */}
    </Suspense>
  );
}
```

**Interview tip:** "Fiber is the reason React can handle complex UIs like financial dashboards with real-time data without freezing the browser. It allows React to prioritize user interactions over background updates."

---

## Q4. What are React Server Components? How do they differ from SSR?

**Answer:**

| Feature | SSR (Server Side Rendering) | React Server Components (RSC) |
|---|---|---|
| Where it runs | Server renders full HTML, then client takes over | Some components stay on server, some on client |
| JavaScript sent | Full app JS sent to client | Only client component JS sent |
| Interactivity | Full page is interactive after hydration | Only client components are interactive |
| Data fetching | Usually fetch on server, pass as props | Direct database/API access in server components |

**SSR Example (Traditional):**
```jsx
// Everything runs on server first, then hydrates on client
// ALL JavaScript is sent to the browser
export async function getServerSideProps() {
  const data = await fetch('https://api.bank.com/accounts');
  return { props: { accounts: data } };
}

function AccountPage({ accounts }) {
  return <AccountList accounts={accounts} />;
}
```

**React Server Components:**
```jsx
// This component NEVER goes to the browser - zero JS sent
// server-component.jsx
async function AccountSummary() {
  const accounts = await db.query('SELECT * FROM accounts');  // Direct DB access!
  return (
    <div>
      {accounts.map(acc => <p key={acc.id}>{acc.name}: ₹{acc.balance}</p>)}
    </div>
  );
}

// This component IS sent to browser - it needs interactivity
// client-component.jsx
'use client';
function TransferButton() {
  const [open, setOpen] = useState(false);
  return <button onClick={() => setOpen(true)}>Transfer Money</button>;
}
```

**Interview tip:** "Server Components are great for financial apps because sensitive data processing stays on the server. The account summary can query the database directly without exposing any API endpoints to the client."

---

## Q5. How does React's batching work in React 18 vs earlier versions?

**Answer:**

**Batching** = React groups multiple state updates together and does only ONE re-render instead of multiple.

**Before React 18:**
```jsx
function handleClick() {
  setName('Neelesh');   // Does NOT re-render yet
  setAge(35);           // Does NOT re-render yet
  // React batches these — only ONE re-render happens here
}

// BUT in async code, batching DID NOT work:
function handleClick() {
  fetch('/api/user').then(() => {
    setName('Neelesh');   // Re-render #1 😟
    setAge(35);           // Re-render #2 😟
    // Two separate re-renders! Not batched!
  });
}
```

**React 18 (Automatic Batching):**
```jsx
// NOW batching works EVERYWHERE - events, promises, timeouts, etc.
function handleClick() {
  fetch('/api/user').then(() => {
    setName('Neelesh');   // Does NOT re-render yet
    setAge(35);           // Does NOT re-render yet
    // Only ONE re-render! Batched! ✅
  });
}

// Even in setTimeout:
setTimeout(() => {
  setName('Neelesh');
  setAge(35);
  // Only ONE re-render! ✅
}, 1000);
```

**If you DON'T want batching (rare case):**
```jsx
import { flushSync } from 'react-dom';

function handleClick() {
  flushSync(() => {
    setName('Neelesh');
  });
  // Re-render happens here

  flushSync(() => {
    setAge(35);
  });
  // Re-render happens here
}
```

**Interview tip:** "Automatic batching in React 18 is a big performance win for financial dashboards where we update multiple pieces of state after an API call — like updating account balance, transaction list, and notifications together in one render."

---

## Q6. What is the difference between `createElement` and JSX? What happens during JSX compilation?

**Answer:**

**JSX** is just syntactic sugar (shortcut) for `React.createElement()`. They produce the exact same result.

```jsx
// What you write (JSX):
const element = <h1 className="title">Hello, Neelesh</h1>;

// What it compiles to (createElement):
const element = React.createElement('h1', { className: 'title' }, 'Hello, Neelesh');

// Both produce this JavaScript object:
{
  type: 'h1',
  props: {
    className: 'title',
    children: 'Hello, Neelesh'
  }
}
```

**More complex example:**
```jsx
// JSX:
<div>
  <AccountCard name="Savings" balance={50000} />
  <AccountCard name="Current" balance={120000} />
</div>

// Compiles to:
React.createElement('div', null,
  React.createElement(AccountCard, { name: 'Savings', balance: 50000 }),
  React.createElement(AccountCard, { name: 'Current', balance: 120000 })
);
```

**Compilation process:**
1. You write JSX
2. **Babel** (a compiler) converts JSX to `createElement` calls
3. In React 17+, it uses a new JSX transform (`jsx()` function from `react/jsx-runtime`) — you don't even need to `import React` anymore

```jsx
// React 17+ new transform:
import { jsx as _jsx } from 'react/jsx-runtime';
const element = _jsx('h1', { className: 'title', children: 'Hello' });
```

---


## Q7. Explain error boundaries. How would you implement a global error boundary for a banking application?

**Answer:**

**Error Boundaries** are React components that catch JavaScript errors in their child component tree and display a fallback UI instead of crashing the whole app.

**Important:** Error boundaries only work with **class components** (no hook equivalent yet).

```jsx
class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error) {
    // Update state to show fallback UI
    return { hasError: true, error };
  }

  componentDidCatch(error, errorInfo) {
    // Log error to monitoring service (important for banking apps!)
    logErrorToService(error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback || <h1>Something went wrong</h1>;
    }
    return this.props.children;
  }
}
```

**Banking application — Global Error Boundary:**
```jsx
function BankingApp() {
  return (
    // Level 1: App-level boundary (catches everything)
    <ErrorBoundary fallback={<FullPageError />}>
      <Header />
      <main>
        {/* Level 2: Section-level boundary */}
        <ErrorBoundary fallback={<p>Account section unavailable</p>}>
          <AccountDashboard />
        </ErrorBoundary>

        {/* Separate boundary so if transfers crash, accounts still work */}
        <ErrorBoundary fallback={<p>Transfer section unavailable</p>}>
          <TransferSection />
        </ErrorBoundary>

        <ErrorBoundary fallback={<p>Could not load transactions</p>}>
          <TransactionHistory />
        </ErrorBoundary>
      </main>
    </ErrorBoundary>
  );
}
```

**What error boundaries DON'T catch:**
- Event handlers (use try-catch inside them)
- Async code (promises)
- Server-side rendering
- Errors in the error boundary itself

**Interview tip:** "In a banking app, I use nested error boundaries so one section crashing doesn't bring down the whole application. If the transaction history fails, the user can still see their account balance and make transfers."

---

## Q8. What are portals in React? Give a real-world use case.

**Answer:**

**Portals** let you render a component's output into a different DOM node, outside its parent hierarchy, while keeping all React behavior (events, context) intact.

```jsx
import { createPortal } from 'react-dom';

function Modal({ children, isOpen }) {
  if (!isOpen) return null;

  return createPortal(
    <div className="modal-overlay">
      <div className="modal-content">
        {children}
      </div>
    </div>,
    document.getElementById('modal-root')  // Renders here, not inside parent
  );
}
```

**HTML structure:**
```html
<body>
  <div id="root">
    <!-- Your app renders here -->
    <div class="account-page">
      <!-- Modal component is HERE in React tree -->
    </div>
  </div>
  <div id="modal-root">
    <!-- But modal renders HERE in DOM -->
  </div>
</body>
```

**Why use portals?**
- **Modals/Dialogs** — avoid CSS z-index and overflow issues
- **Tooltips** — render above everything without CSS hacks
- **Dropdown menus** — escape overflow:hidden containers

**Banking use case — Transaction Confirmation Modal:**
```jsx
function TransferPage() {
  const [showConfirm, setShowConfirm] = useState(false);

  return (
    <div className="transfer-form">
      <h2>Transfer Money</h2>
      <input placeholder="Amount" />
      <button onClick={() => setShowConfirm(true)}>Transfer</button>

      {/* Even though this is inside transfer-form,
          it renders at document body level */}
      <Modal isOpen={showConfirm}>
        <h3>Confirm Transfer</h3>
        <p>Are you sure you want to transfer ₹50,000?</p>
        <button onClick={handleConfirm}>Yes, Transfer</button>
        <button onClick={() => setShowConfirm(false)}>Cancel</button>
      </Modal>
    </div>
  );
}
```

**Key point:** Even though the modal renders outside the parent DOM, React events still bubble up through the React tree (not the DOM tree). So a click inside the portal can be caught by the parent React component.

---

## Q9. How does React's key prop work internally? What happens if you use array index as key?

**Answer:**

The **`key`** prop helps React identify which items in a list have changed, been added, or removed. It's used during the diffing process.

**How it works internally:**
1. React compares old and new lists by their keys
2. Same key = same element (just update props if needed)
3. New key = new element (create it)
4. Missing key = removed element (destroy it)

**Problem with index as key:**
```jsx
// ❌ BAD - Using index as key
function TransactionList({ transactions }) {
  return transactions.map((txn, index) => (
    <TransactionItem key={index} data={txn} />
  ));
}

// What happens when you delete the first transaction:
// Before: [0: TxnA, 1: TxnB, 2: TxnC]
// After:  [0: TxnB, 1: TxnC]
// React thinks: key 0 changed (TxnA → TxnB), key 1 changed (TxnB → TxnC), key 2 removed
// Result: React updates ALL items instead of just removing one! 😟
```

```jsx
// ✅ GOOD - Using unique ID as key
function TransactionList({ transactions }) {
  return transactions.map((txn) => (
    <TransactionItem key={txn.transactionId} data={txn} />
  ));
}

// What happens when you delete the first transaction:
// Before: [id1: TxnA, id2: TxnB, id3: TxnC]
// After:  [id2: TxnB, id3: TxnC]
// React thinks: id1 removed, id2 and id3 unchanged
// Result: React only removes TxnA! ✅
```

**When index as key is OK:**
- List is static (never changes)
- List is never reordered or filtered
- Items have no state

**Interview tip:** "In a financial app showing transaction history, using index as key would cause bugs — if a new transaction comes in at the top, all items would re-render and any expanded/selected state would shift to the wrong transaction."

---

## Q10. What are the differences between class components and functional components beyond syntax?

**Answer:**

| Feature | Class Components | Functional Components |
|---|---|---|
| Syntax | `class App extends React.Component` | `function App()` or `const App = () =>` |
| State | `this.state` and `this.setState` | `useState` hook |
| Lifecycle | `componentDidMount`, `componentDidUpdate`, etc. | `useEffect` hook |
| `this` keyword | Required (common source of bugs) | Not needed |
| Performance | Slightly heavier (class instance created) | Lighter |
| Error Boundaries | Supported | NOT supported (still need class) |
| Code reuse | HOCs, render props | Custom hooks (much cleaner) |
| Closure behavior | Always reads latest `this.props` | Captures props at render time |

**The closure difference is important:**
```jsx
// Class Component - always reads LATEST props
class ClassMessage extends React.Component {
  showMessage = () => {
    setTimeout(() => {
      alert(this.props.message);  // Reads current props (might have changed!)
    }, 3000);
  };
}

// Functional Component - captures props at render time
function FunctionalMessage({ message }) {
  const showMessage = () => {
    setTimeout(() => {
      alert(message);  // Reads the message from when button was clicked
    }, 3000);
  };
}
```

**Interview tip:** "Today I use functional components for everything except error boundaries. The main advantage isn't just cleaner syntax — it's that custom hooks allow much better code reuse than HOCs or render props ever did."

---

## Q11. Explain Strict Mode in React. What double-invocations does it cause and why?

**Answer:**

**Strict Mode** is a development-only tool that helps find potential problems in your app. It does NOT affect production builds.

```jsx
// Enable for entire app
root.render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

**What Strict Mode does (double invocations):**

1. **Renders components twice** — to find side effects in render
2. **Runs effects twice** (mount → unmount → mount) — to find cleanup bugs
3. **Runs reducers/initializers twice** — to ensure they're pure

```jsx
function AccountBalance() {
  console.log('Rendering!');  // You'll see this TWICE in dev
 
  useEffect(() => {
    console.log('Effect mounted');    // Runs
    const ws = new WebSocket('wss://prices.bank.com');
   
    return () => {
      console.log('Effect cleanup');  // Runs
      ws.close();
    };
    // Strict Mode: mount → cleanup → mount (to test cleanup works)
  }, []);

  return <div>Balance: ₹50,000</div>;
}
```

**Why this is useful:**
```jsx
// ❌ BUG that Strict Mode catches:
useEffect(() => {
  const ws = new WebSocket('wss://prices.bank.com');
  // Forgot cleanup! Strict Mode will show duplicate connections
}, []);

// ✅ Correct:
useEffect(() => {
  const ws = new WebSocket('wss://prices.bank.com');
  return () => ws.close();  // Proper cleanup
}, []);
```

**Strict Mode also warns about:**
- Legacy string refs
- Deprecated lifecycle methods (`componentWillMount`, etc.)
- Legacy Context API usage

---

## Q12. What is prop drilling and what are the solutions?

**Answer:**

**Prop drilling** = passing data through many levels of components that don't need it, just to get it to a deeply nested component.

```jsx
// ❌ Prop Drilling Problem:
function App() {
  const [user, setUser] = useState({ name: 'Neelesh', role: 'admin' });
  return <Dashboard user={user} />;
}

function Dashboard({ user }) {  // Doesn't use user, just passes it
  return <Sidebar user={user} />;
}

function Sidebar({ user }) {  // Doesn't use user, just passes it
  return <UserProfile user={user} />;
}

function UserProfile({ user }) {  // Finally uses it!
  return <p>Welcome, {user.name}</p>;
}
```

**Solutions:**

**1. Context API (Best for simple shared data like theme, user, language):**
```jsx
const UserContext = createContext();

function App() {
  const [user, setUser] = useState({ name: 'Neelesh', role: 'admin' });
  return (
    <UserContext.Provider value={user}>
      <Dashboard />  {/* No prop passing needed! */}
    </UserContext.Provider>
  );
}

function UserProfile() {
  const user = useContext(UserContext);  // Gets data directly!
  return <p>Welcome, {user.name}</p>;
}
```

**2. Redux (Best for complex app-wide state):**
```jsx
function UserProfile() {
  const user = useSelector(state => state.auth.user);
  return <p>Welcome, {user.name}</p>;
}
```

**3. Component Composition (Often overlooked but powerful):**
```jsx
function App() {
  const user = { name: 'Neelesh' };
  return (
    <Dashboard>
      <Sidebar>
        <UserProfile user={user} />  {/* Pass directly! */}
      </Sidebar>
    </Dashboard>
  );
}
```

**Interview tip:** "I choose the solution based on the use case — Context for theme/auth, Redux for complex business state like transaction data, and composition when the component tree allows it."

---

## Q12b. State Architecture Decision Guide — Lifting State vs Prop Drilling vs Context vs Store. How do you decide?

**Answer:**

This is the question that separates seniors from juniors. Interviewers don't want to hear which API you know — they want to see **how you think about state location**.

---

### 1. Lifting State — It's a Consequence, Not a Goal

**What juniors say:** "I lift state to the parent."
**What seniors say:** "I lift state only when multiple siblings truly depend on the **same source of truth**. If I'm lifting more than 2-3 levels, that's an architectural smell — not a React solution."

```
  Think of it like a shared electricity meter:

  ┌─────────────────────────────────────────────┐
  │              Parent (owns state)             │
  │                    │                         │
  │           ┌───────┴───────┐                  │
  │           ▼               ▼                  │
  │     ┌──────────┐   ┌──────────┐             │
  │     │  Input   │   │  Preview │             │
  │     │ (writes) │   │  (reads) │             │
  │     └──────────┘   └──────────┘             │
  │                                              │
  │  Both NEED the same value → lift to parent   │
  │  This is correct ✅                          │
  └─────────────────────────────────────────────┘
```

```jsx
// ✅ Good use of lifting state — siblings share data
function TransferPage() {
  const [amount, setAmount] = useState('');
  return (
    <>
      <AmountInput value={amount} onChange={setAmount} />
      <TransferPreview amount={amount} />
      {/* Both NEED amount → parent owns it. Correct. */}
    </>
  );
}
```

**When lifting state goes wrong:**
```
  ❌ BAD — forced to lift 4 levels up:

  App (owns user state)
   └── Dashboard (passes user ↓)          ← doesn't use it
        └── Sidebar (passes user ↓)       ← doesn't use it
             └── Menu (passes user ↓)     ← doesn't use it
                  └── Avatar (finally uses user!)

  Four levels of passing for one consumer = architectural smell
  Solution: Context or store, NOT more lifting
```

**Key rule:** Lifting state = good when 1-2 levels. Beyond that, it's prop drilling in disguise.

---

### 2. Prop Drilling — When It's Fine vs When It's a Problem

**What juniors say:** "Prop drilling is bad."
**What seniors say:** "Prop drilling is acceptable when it's **shallow and explicit**."

```
  ✅ ACCEPTABLE (2 levels, direct relationship):

  Page
   └── TransactionTable  (receives transactions)
        └── TransactionRow  (receives single transaction)

  This is fine — it's component API design, not a problem.
```

```
  ❌ PROBLEM (3+ levels, intermediaries don't care):

  App
   └── Layout              (passes theme ↓)  ← doesn't care
        └── Main           (passes theme ↓)  ← doesn't care
             └── Sidebar   (passes theme ↓)  ← doesn't care
                  └── Card (finally uses theme)

  3 components exist just to forward a prop they never read.
```

**Quick test to know if prop drilling is a problem:**

| Check | Fine | Problem |
|---|---|---|
| How deep? | 1-2 levels | 3+ levels |
| Does intermediary use the prop? | Yes | No (just forwards it) |
| Are components tightly related? | Yes (parent-child) | No (distant cousins) |
| Would renaming the prop break many files? | 1-2 files | 5+ files |

**Interview line:** "Prop drilling is not a React problem — **excessive** prop drilling is a design problem."

---

### 3. React Context — Use Carefully, Not by Default

**What it's good for:** Low-frequency, cross-cutting concerns that many components need but that rarely change.

```
  ✅ CONTEXT IS PERFECT FOR:          ❌ CONTEXT IS BAD FOR:

  Theme (light/dark)                  Fast-changing form state
  Auth session / user role            Typing / drag events
  Locale / i18n language              Large mutable objects
  Feature flags / RBAC                High-frequency updates
  App-level config                    (10+ times per second)
```

**The performance trap interviewers love to ask about:**

```
  When Context value changes, EVERY consumer re-renders:

  ThemeContext.Provider value={theme}
       │
       ├── Header      ← re-renders ✅ (uses theme)
       ├── Sidebar     ← re-renders ❌ (only uses user, but still re-renders!)
       └── Dashboard   ← re-renders ❌ (doesn't use theme at all!)

  Why? Because ALL children of the Provider re-render
  when the context value reference changes.
```

**The fix — split contexts:**
```jsx
// ❌ BAD — one giant context for everything
const AppContext = createContext();
<AppContext.Provider value={{ user, theme, notifications, locale }}>
  {/* Changing theme re-renders EVERYTHING */}
</AppContext.Provider>

// ✅ GOOD — separate contexts by change frequency
const ThemeContext = createContext();     // Changes rarely
const AuthContext = createContext();      // Changes on login/logout
const NotificationContext = createContext(); // Changes often

// Now changing theme doesn't touch auth consumers
```

**Proper Context pattern with memoization:**
```jsx
const AuthContext = createContext(null);

function AuthProvider({ children }) {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(null);

  // ✅ Memoize the value object so consumers don't re-render
  // unless user or token actually changed
  const value = useMemo(() => ({
    user,
    token,
    login: (credentials) => { /* ... */ },
    logout: () => { setUser(null); setToken(null); },
  }), [user, token]);

  return (
    <AuthContext.Provider value={value}>
      {children}
    </AuthContext.Provider>
  );
}

// Usage — clean, no prop drilling
function Avatar() {
  const { user } = useContext(AuthContext);
  return <img src={user?.avatar} alt={user?.name} />;
}
```

---

### 4. useReducer + Context — The "Built-in Store" (No Redux Needed)

When you need **predictable state transitions** across a feature but don't need Redux's ecosystem (middleware, DevTools, huge team conventions):

```jsx
// A shopping cart that multiple components need to read and update

// Step 1: Define the reducer (predictable transitions)
const cartReducer = (state, action) => {
  switch (action.type) {
    case 'ADD_ITEM':
      return { ...state, items: [...state.items, action.item] };
    case 'REMOVE_ITEM':
      return { ...state, items: state.items.filter(i => i.id !== action.id) };
    case 'CLEAR':
      return { ...state, items: [] };
    default:
      return state;
  }
};

// Step 2: Create context + provider
const CartContext = createContext();

function CartProvider({ children }) {
  const [state, dispatch] = useReducer(cartReducer, { items: [] });

  // Memoize to prevent unnecessary re-renders
  const value = useMemo(() => ({ state, dispatch }), [state]);

  return (
    <CartContext.Provider value={value}>
      {children}
    </CartContext.Provider>
  );
}

// Step 3: Use anywhere — reads and writes without prop drilling
function CartIcon() {
  const { state } = useContext(CartContext);
  return <span>🛒 {state.items.length}</span>;
}

function ProductCard({ product }) {
  const { dispatch } = useContext(CartContext);
  return (
    <button onClick={() => dispatch({ type: 'ADD_ITEM', item: product })}>
      Add to Cart
    </button>
  );
}
```

```
  How it works — just like Redux, minus the middleware:

  dispatch({ type: 'ADD_ITEM', item })
       │
       ▼
  cartReducer(currentState, action)
       │
       ▼
  New state returned
       │
       ▼
  Context value updates → consumers re-render
```

**When to use useReducer + Context vs Redux:**

| useReducer + Context | Redux |
|---|---|
| Feature-scoped state (cart, wizard, filter) | App-wide state shared by many features |
| Well-defined state transitions | Complex async side effects (thunks, sagas) |
| Small-to-medium apps | Large apps with many developers |
| Want zero extra dependencies | Need DevTools, time-travel, middleware |
| Testable logic (reducers are pure) | Need to combine state from many sources |

**Senior insight:** "useReducer + Context is essentially Redux without middleware and DevTools. For a feature like a multi-step loan wizard, it's cleaner than adding Redux just for one flow."

---

### 5. Global Store (Redux / Zustand) — When You Actually Need It

**Interviewer trap:** "Why not just use Context everywhere?"

**Answer:** "Context is for **distribution**, not state management at scale."

Context becomes painful when:
- Many distant components need the same data
- Updates are frequent (every second, every keystroke)
- You need time-travel debugging
- Async flows need centralized management
- Large teams need predictable, enforceable patterns

```
  ✅ Examples that justify a global store:

  ┌─────────────────────────────────────────────────┐
  │  Header (shows user name + balance)             │
  │  Sidebar (shows account list)                   │
  │  Dashboard (shows transactions)                 │
  │  Transfer Form (needs accounts + balance)       │
  │  Notification Bell (real-time alerts)           │
  │                                                 │
  │  ALL need overlapping data, updated frequently  │
  │  → Redux/Zustand is the right call              │
  └─────────────────────────────────────────────────┘
```

---

### 6. The Complete Decision Matrix (Interview Gold)

```
  Is the state local to ONE component?
  │
  YES → useState / useReducer (keep it local)
  │
  NO → Shared by siblings, limited scope?
       │
       YES → Lift state to parent (1-2 levels max)
       │
       NO → Prop drilling going 3+ levels?
            │
            YES → Is it low-frequency, cross-cutting data?
            │     │
            │     YES → Context API (theme, auth, locale)
            │     │
            │     NO → Is it feature-scoped with clear transitions?
            │           │
            │           YES → useReducer + Context ("built-in store")
            │           │
            │           NO → App-wide, high-frequency,
            │                 needs middleware/DevTools?
            │                 │
            │                 YES → Redux / Zustand
            │
            └── Is it server-sourced async data?
                │
                YES → React Query / RTK Query (NOT Redux for fetch caching)
```

**Quick reference table:**

| State Type | Tool | Example |
|---|---|---|
| UI / ephemeral | `useState` / `useReducer` | Modal open/close, form input, accordion |
| Shared siblings | Lift state | Currency converter, synced inputs |
| Cross-cutting config | Context | Theme, auth, locale, feature flags |
| Feature-scoped logic | `useReducer` + Context | Shopping cart, multi-step wizard |
| App-wide global | Redux / Zustand | User profile, permissions, dashboards |
| Server/async data | React Query / RTK Query | Transactions, account balances, search results |

---

### 7. The 16-Year Experience Closing Statement (Use This in Interview)

> "Over the years, I've learned that **state location matters more than state tool**. I start local, lift only when necessary, introduce Context for cross-cutting concerns, and bring in a store only when complexity justifies the cost. Most React performance problems are **architectural, not API-related**."

This line sounds senior and naturally ends the discussion.

---

## Q13. How would you handle a complex multi-step form (e.g., loan application) in React?

**Answer:**

For a multi-step loan application form, I would use this approach:

```jsx
// Step 1: Define form steps and shared state
function LoanApplication() {
  const [step, setStep] = useState(1);
  const [formData, setFormData] = useState({
    // Step 1: Personal Details
    fullName: '', panNumber: '', dateOfBirth: '',
    // Step 2: Employment Details
    employer: '', salary: '', experience: '',
    // Step 3: Loan Details
    loanAmount: '', tenure: '', purpose: '',
    // Step 4: Documents
    documents: []
  });

  function updateForm(fields) {
    setFormData(prev => ({ ...prev, ...fields }));
  }

  function nextStep() {
    if (validateStep(step)) setStep(s => s + 1);
  }

  function prevStep() {
    setStep(s => s - 1);
  }

  return (
    <div>
      <StepIndicator currentStep={step} totalSteps={4} />
     
      {step === 1 && (
        <PersonalDetails data={formData} update={updateForm} />
      )}
      {step === 2 && (
        <EmploymentDetails data={formData} update={updateForm} />
      )}
      {step === 3 && (
        <LoanDetails data={formData} update={updateForm} />
      )}
      {step === 4 && (
        <DocumentUpload data={formData} update={updateForm} />
      )}

      <div className="buttons">
        {step > 1 && <button onClick={prevStep}>Back</button>}
        {step < 4 ? (
          <button onClick={nextStep}>Next</button>
        ) : (
          <button onClick={handleSubmit}>Submit Application</button>
        )}
      </div>
    </div>
  );
}
```

**For larger forms, use `useReducer`:**
```jsx
const formReducer = (state, action) => {
  switch (action.type) {
    case 'UPDATE_FIELD':
      return { ...state, [action.field]: action.value };
    case 'SET_STEP':
      return { ...state, currentStep: action.step };
    case 'RESET':
      return initialState;
    default:
      return state;
  }
};
```

**Key considerations for financial forms:**
- Save progress to localStorage (in case user accidentally closes tab)
- Validate each step before allowing "Next"
- Show a summary/review step before final submission
- Mask sensitive fields (PAN, Aadhaar)

---

## Q14. What are Higher-Order Components (HOCs)? Are they still relevant with hooks?

**Answer:**

A **HOC** is a function that takes a component and returns a new component with extra functionality.

```jsx
// HOC that adds authentication check
function withAuth(WrappedComponent) {
  return function AuthenticatedComponent(props) {
    const isLoggedIn = checkAuth();
   
    if (!isLoggedIn) {
      return <Redirect to="/login" />;
    }
   
    return <WrappedComponent {...props} />;
  };
}

// Usage:
const ProtectedDashboard = withAuth(Dashboard);
const ProtectedTransfers = withAuth(TransferPage);
```

**Are HOCs still relevant?**

| HOCs | Custom Hooks |
|---|---|
| Wrap component (adds nesting) | Used inside component (no nesting) |
| Can be confusing (wrapper hell) | Cleaner and more readable |
| Can intercept rendering | Cannot intercept rendering |
| Props can clash | No props clash |

**Most HOC use cases are replaced by hooks:**
```jsx
// ❌ Old way (HOC):
const EnhancedComponent = withAuth(withTheme(withLogging(MyComponent)));

// ✅ New way (Hooks):
function MyComponent() {
  const auth = useAuth();
  const theme = useTheme();
  useLogging('MyComponent');
  // ... clean and simple
}
```

**When HOCs are still useful:**
- When you need to wrap rendering logic (like adding a loading spinner around any component)
- When working with third-party libraries that expect HOCs
- Error boundaries (no hook alternative)

---

## Q15. Explain the concept of lifting state up. When is it appropriate vs using global state?

**Answer:**

**Lifting state up** = moving state from a child component to the closest common parent when two or more siblings need to share data.

```jsx
// ❌ Before: Each component has its own state (can't sync)
function CurrencyConverter() {
  return (
    <div>
      <RupeeInput />    {/* Has its own state */}
      <DollarInput />   {/* Has its own state, can't sync with Rupee */}
    </div>
  );
}

// ✅ After: State lifted to parent
function CurrencyConverter() {
  const [amount, setAmount] = useState('');
  const [currency, setCurrency] = useState('INR');

  const inrAmount = currency === 'INR' ? amount : amount * 83;
  const usdAmount = currency === 'USD' ? amount : amount / 83;

  return (
    <div>
      <CurrencyInput
        label="INR"
        value={inrAmount}
        onChange={(val) => { setAmount(val); setCurrency('INR'); }}
      />
      <CurrencyInput
        label="USD"
        value={usdAmount}
        onChange={(val) => { setAmount(val); setCurrency('USD'); }}
      />
    </div>
  );
}
```

**When to lift state vs use global state:**

| Lift State Up | Use Global State (Redux/Context) |
|---|---|
| 2-3 siblings share data | Many components across app need data |
| Data is localized to one feature | Data is app-wide (user, theme, cart) |
| Simple parent-child relationship | Deep component nesting |
| Example: Two inputs that sync | Example: User authentication status |

**Interview tip:** "I start with lifting state up because it's simpler. If I find myself lifting state too many levels or the same data is needed in unrelated parts of the app, that's when I move to Context or Redux."

---

## Q15b. Draw and explain the complete React rendering lifecycle diagram — from setState to screen update.

**Answer:**

**The full journey of a state update:**

```
  setState() called
       │
       ▼
  ┌────────────────────────────────────────────────────────────┐
  │                    RENDER PHASE                            │
  │                (Pure · Interruptible)                      │
  │                                                            │
  │   1. React schedules a render                              │
  │   2. Runs your function component (top → down)             │
  │   3. Calls useState → gets current state                   │
  │   4. Calls useMemo/useCallback → returns cached values     │
  │   5. Returns JSX → React builds WIP Fiber tree             │
  │   6. React diffs WIP tree vs Current tree                  │
  │                                                            │
  │   ⏸️  Can be PAUSED here (React 18 concurrency)             │
  │   🔄  Can be RESTARTED if a higher priority update arrives  │
  │   ❌  Can be DISCARDED if the update is outdated            │
  └────────────────────────┬───────────────────────────────────┘
                           │
                           ▼
  ┌────────────────────────────────────────────────────────────┐
  │                    COMMIT PHASE                            │
  │             (DOM · NOT Interruptible)                      │
  │                                                            │
  │   7. React applies minimal DOM mutations (insert/update/   │
  │      remove only what changed)                             │
  │   8. useLayoutEffect runs (sync, before paint)             │
  │   9. Browser PAINTS → user sees the update                 │
  │  10. useEffect runs (async, after paint)                   │
  │                                                            │
  │   🔒  Cannot be interrupted — runs to completion            │
  └────────────────────────────────────────────────────────────┘
```

**Mapping to a real banking scenario:**

```jsx
function AccountDashboard() {
  // ── RENDER PHASE (steps 1-6) ──────────────────────
  const [account, setAccount] = useState(null);        // Step 3
  const formattedBalance = useMemo(                    // Step 4
    () => formatCurrency(account?.balance),
    [account?.balance]
  );
  // Step 5: Returns JSX ↓

  // ── COMMIT PHASE (steps 7-10) ─────────────────────
  useLayoutEffect(() => {                              // Step 8
    // Measure DOM before paint (e.g., calculate chart dimensions)
    const width = chartRef.current.offsetWidth;
  }, []);

  useEffect(() => {                                    // Step 10
    // After paint — fetch data, subscribe to websocket
    const ws = new WebSocket('wss://accounts.bank.com');
    ws.onmessage = (e) => setAccount(JSON.parse(e.data));
    return () => ws.close();
  }, []);

  return (
    <div>
      <h2>Balance: {formattedBalance}</h2>
      <div ref={chartRef}><BalanceChart /></div>
    </div>
  );
}
```

**What happens when user clicks "Refresh Balance":**
```
User clicks button
  → onClick handler calls setAccount(newData)
    → React SCHEDULES a new render
      → RENDER PHASE: React re-runs AccountDashboard()
        → useMemo recalculates formattedBalance (if balance changed)
        → New JSX returned → WIP tree built → diff with current tree
      → COMMIT PHASE:
        → DOM updated (only the <h2> text changes)
        → useLayoutEffect runs
        → Browser paints
        → useEffect runs (cleanup of old effect first, then new one)
```

**Interview tip:** "The key insight is that everything in my function body runs during the render phase — so it must be pure. All side effects go into useEffect (after paint) or useLayoutEffect (before paint). React 18 can pause the render phase mid-way to handle urgent user input, which is why purity matters even more now."

---

## Q15c. How does React's rendering lifecycle connect to Redux re-renders? When does a Redux state change trigger a component re-render?

**Answer:**

Redux integrates with React's rendering lifecycle through `useSelector`. Understanding the flow prevents unnecessary re-renders in large apps.

**The Redux → React rendering flow:**

```
  dispatch(action)
       │
       ▼
  Reducer returns NEW state
       │
       ▼
  Redux Store updated
       │
       ▼
  Redux notifies ALL useSelector() subscribers
       │
       ▼
  Each useSelector() runs its selector function
       │
       ├── Selected value SAME (===) → ❌ No re-render
       │
       └── Selected value DIFFERENT → ✅ Triggers re-render
                                            │
                                            ▼
                                      React RENDER PHASE
                                      (same two-phase model)
```

**Common trap — creating new references every render:**
```jsx
// ❌ BAD — creates a new array every time Redux notifies
function TransactionList() {
  const transactions = useSelector(state =>
    state.transactions.filter(t => t.amount > 1000)  // New array every time!
  );
  // This component re-renders on EVERY Redux state change,
  // even changes in completely unrelated slices!
}

// ✅ GOOD — use Reselect memoized selector
import { createSelector } from '@reduxjs/toolkit';

const selectLargeTransactions = createSelector(
  state => state.transactions,
  transactions => transactions.filter(t => t.amount > 1000)  
  // Only recalculates if state.transactions changes
);

function TransactionList() {
  const transactions = useSelector(selectLargeTransactions);
  // Only re-renders when the actual transactions array changes
}
```

**Where Redux fits in the React lifecycle:**

| Step | What Happens |
|---|---|
| 1. `dispatch(action)` | Reducer runs (synchronous, pure) |
| 2. Store updated | New state reference created |
| 3. `useSelector` checks | Compares old vs new selected value (`===`) |
| 4. If different → render | React enters **Render Phase** for that component |
| 5. Commit Phase | DOM updated, effects run |

**Multiple dispatches and batching (React 18):**
```jsx
function handleTransfer() {
  dispatch(debitAccount(amount));      // State change 1
  dispatch(creditAccount(amount));     // State change 2
  dispatch(addTransaction(details));   // State change 3
 
  // React 18: All three are BATCHED → only ONE re-render! ✅
  // React 17: Each dispatch caused a separate re-render! ❌
}
```

**Performance rules with Redux:**
1. **Select only what you need** — `useSelector(s => s.user.name)` not `useSelector(s => s.user)`
2. **Use memoized selectors** for derived/filtered data
3. **Split state into small slices** — changing one slice won't notify components subscribed to another
4. **Don't store derived data** — compute it in selectors

**Interview tip:** "Redux re-renders follow the same two-phase model as any React render. The key difference is that `useSelector` acts as the gatekeeper — it runs the selector on every store change and only triggers a render if the selected value's reference changed. That's why memoized selectors from Reselect are critical for performance."

---

## Q15d. Why do effects sometimes run twice in StrictMode? Explain the exact mechanism.

**Answer:**

In development mode, React StrictMode **intentionally mounts, unmounts, then re-mounts** every component. This causes `useEffect` to run twice. This is **by design** — it's not a bug.

**What StrictMode does (exact sequence):**
```
  Normal Mount (without StrictMode):
  ┌────────────────────────────────┐
  │ 1. Render phase                │
  │ 2. Commit phase (DOM inserted) │
  │ 3. useEffect runs              │
  └────────────────────────────────┘

  StrictMode Mount (development only):
  ┌────────────────────────────────┐
  │ 1. Render phase                │  ← Runs TWICE
  │ 2. Commit phase (DOM inserted) │
  │ 3. useEffect runs              │  ← Effect #1
  │ 4. useEffect CLEANUP runs      │  ← Simulates unmount
  │ 5. useEffect runs AGAIN        │  ← Effect #2 (re-mount)
  └────────────────────────────────┘
```

**Why React does this — to catch cleanup bugs:**

```jsx
// ❌ BUG that StrictMode catches:
useEffect(() => {
  const ws = new WebSocket('wss://prices.bank.com');
  ws.onmessage = (e) => setPrice(JSON.parse(e.data));
  // No cleanup! StrictMode shows TWO connections open
}, []);

// ✅ Correct — StrictMode-safe:
useEffect(() => {
  const ws = new WebSocket('wss://prices.bank.com');
  ws.onmessage = (e) => setPrice(JSON.parse(e.data));
  return () => ws.close();  // Cleanup! First connection closed, second one works fine
}, []);
```

**Real-world example — what goes wrong without cleanup:**
```jsx
// Without cleanup, StrictMode shows the problem:
useEffect(() => {
  const interval = setInterval(() => {
    fetchBalance();  // Called TWICE as often — two intervals running!
  }, 5000);
}, []);

// With cleanup, StrictMode proves it's safe:
useEffect(() => {
  const interval = setInterval(() => {
    fetchBalance();
  }, 5000);
  return () => clearInterval(interval);  // First interval cleared, second takes over
}, []);
```

**The key insight — idempotent effects:**

StrictMode tests whether your effect is **idempotent** (safe to run multiple times). If mount → unmount → mount produces the same result as a single mount, your effect is correct.

```
Good effect:     mount(cleanup) → mount = same as → mount
Bad effect:      mount(no cleanup) → mount = TWO subscriptions, TWO intervals, etc.
```

**What double-fires in StrictMode vs what doesn't:**

| What | Double-fires? | Why |
|---|---|---|
| Function body (render) | ✅ Yes | Tests render purity |
| `useState` initializer | ✅ Yes | Tests initializer purity |
| `useEffect` | ✅ Yes (mount→cleanup→mount) | Tests cleanup correctness |
| `useLayoutEffect` | ✅ Yes | Same as useEffect |
| `useReducer` reducer | ✅ Yes | Tests reducer purity |
| Event handlers | ❌ No | Not called during render |
| `useRef` | ❌ No | Persists across re-renders |

**Important:** This only happens in **development mode**. Production builds run everything exactly once.

**Interview tip:** "StrictMode double-fires effects to simulate a real-world scenario: a user navigates away and comes back (like React Router remounting a component). If your effect doesn't clean up properly, you'd get duplicate subscriptions, duplicate API calls, or memory leaks. StrictMode surfaces these bugs during development so you fix them before production."

---

## Q15d-ii. Common interview trap questions on React 18 concurrency (with correct answers).

**Answer:**

These are specific traps interviewers use to test if you truly understand concurrency vs just memorized API names.

**Trap 1: "Does concurrent rendering mean everything is async?"**

> ❌ Wrong: "Yes, React 18 makes all rendering asynchronous."
>
> ✅ Correct: "No. Concurrency means **interruptibility and prioritization**, not async execution. The render phase can be paused, but the commit phase still applies DOM updates **synchronously**. React never shows a half-committed tree."

**Trap 2: "Does `startTransition` delay state updates?"**

> ❌ Wrong: "Yes, it delays the setState call."
>
> ✅ Correct: "No. The **state update happens immediately** — the Redux store or React state is updated right away. `startTransition` tells React that **rendering** this update is low priority. React may defer, pause, or restart the rendering — but the state itself is already updated."

```jsx
startTransition(() => {
  setResults(filterHugeList(query));  
  // State is set NOW. React decides WHEN to render it.
});
```

**Trap 3: "Can React show partial UI during concurrent rendering?"**

> ❌ Wrong: "Yes, that's what incremental rendering means."
>
> ✅ Correct: "No. React only commits a **fully completed tree**. During the render phase, React may build partial work in memory, but it **never commits incomplete UI** to the DOM. Users always see a consistent, atomic update."

**Trap 4: "Is concurrent rendering always enabled in React 18?"**

> ❌ Wrong: "Yes, just use `createRoot` and everything is concurrent."
>
> ✅ Correct: "No. `createRoot` **enables** concurrency, but React only **uses** concurrent features when you opt in — via `startTransition`, `useDeferredValue`, or `<Suspense>`. Without these, React 18 renders synchronously, just like React 17."

**Trap 5: "Does Redux break with concurrent rendering?"**

> ❌ Wrong: "Yes, Redux dispatches conflict with React's scheduling."
>
> ✅ Correct: "No. Redux state updates are **synchronous** (reducer runs, store updates immediately). React 18 controls **when and how the UI renders** that state. Redux works fine because `useSelector` integrates with React's rendering cycle. React may delay rendering, but the store is always consistent."

```jsx
// This works perfectly in React 18:
function Search() {
  const dispatch = useDispatch();

  function onChange(value) {
    dispatch(setSearchQuery(value));         // urgent — renders immediately
    startTransition(() => {
      dispatch(fetchResults(value));         // non-urgent — rendering can wait
    });
  }
}
// Store: updated for both dispatches immediately
// UI: query input updates first, results render when React is ready
```

**Trap 6: "Why can effects run multiple times?"**

> ❌ Wrong: "It's a bug in React 18."
>
> ✅ Correct: "Because React may **start, pause, and restart** renders. Effects must be resilient to re-running. In StrictMode (dev only), React deliberately double-fires effects to surface cleanup bugs. In production, effects run once per commit."

**Quick recall table for speed rounds:**

| Trap Question | One-word Answer | Explanation |
|---|---|---|
| Is concurrent rendering async? | **No** | It's interruptible, not asynchronous |
| Does `startTransition` delay state? | **No** | State updates immediately; rendering is deferred |
| Can React show partial UI? | **No** | Only fully completed trees are committed |
| Is concurrency always on? | **No** | Opt-in via transitions, Suspense, useDeferredValue |
| Does Redux break? | **No** | Redux is sync; React controls rendering schedule |
| Why do effects run twice? | **StrictMode** | Dev-only double-fire to catch missing cleanup |

---

## Q15e. One-liner rapid fire answers for React Core concepts (interview speed round).

**Answer:**

| Question | One-liner Answer |
|---|---|
| What is Virtual DOM? | A lightweight JS copy of the real DOM that React diffs to minimize actual DOM updates. |
| What is reconciliation? | The algorithm React uses to diff old and new Virtual DOM trees and apply minimal changes. |
| What is React Fiber? | React's internal engine that breaks rendering into interruptible units of work (fibers). |
| What are the two rendering phases? | **Render** (pure, calculates changes in memory) and **Commit** (applies DOM updates, runs effects). |
| What is concurrent rendering? | React 18's ability to pause, resume, or abandon renders to keep the UI responsive. |
| What is automatic batching? | React 18 groups multiple `setState` calls into one re-render, even inside promises and timeouts. |
| Controlled vs Uncontrolled? | Controlled = React state drives the input value. Uncontrolled = DOM holds the value via `ref`. |
| What is a portal? | Renders a component into a different DOM node while keeping React event bubbling intact. |
| What is an error boundary? | A class component that catches JS errors in its child tree and shows a fallback UI. |
| Why not use index as key? | Index keys cause wrong elements to update when the list is reordered, filtered, or items are added/removed. |
| What is prop drilling? | Passing props through many intermediate components just to reach a deeply nested one. |
| What is lifting state up? | Moving state to the closest common parent so sibling components can share it. |
| What are HOCs? | Functions that take a component and return a new one with added behavior (e.g., `withAuth(Page)`). |
| What is `startTransition`? | Marks a state update as low-priority so React can handle urgent input first. |
| Why does StrictMode double-render? | To catch impure renders and missing effect cleanups during development. |
| What is the children pattern? | Passing components as `children` to avoid re-rendering them when the wrapper's state changes. |
| What is SSR vs RSC? | SSR renders full HTML on server then hydrates. RSC keeps some components server-only (zero client JS). |
| What is `flushSync`? | Forces React to commit a state update immediately, bypassing batching. |
| When to use `useLayoutEffect`? | When you need to measure or mutate the DOM **before** the browser paints (e.g., tooltips, scroll position). |
| What is the WIP tree? | The Work-In-Progress Fiber tree — a draft React builds in memory before committing to the DOM. |

**How to use in an interview:** These one-liners are your **first sentence**. After the one-liner, pause — let the interviewer decide if they want depth. If they do, expand with examples and trade-offs.

---

## Quick Revision Checklist

- [ ] Virtual DOM, diffing, reconciliation
- [ ] Controlled vs Uncontrolled components
- [ ] React Fiber architecture
- [ ] Server Components vs SSR
- [ ] Automatic Batching in React 18
- [ ] JSX compilation process
- [ ] Error Boundaries (class components only)
- [ ] Portals and use cases
- [ ] Key prop and why index-as-key is bad
- [ ] Class vs Functional components
- [ ] Strict Mode double-invocations
- [ ] Prop drilling solutions
- [ ] State architecture decision: local → lift → Context → useReducer+Context → Redux → React Query
- [ ] Multi-step form patterns
- [ ] HOCs vs Custom Hooks
- [ ] Lifting State Up vs Global State
- [ ] React 18 two-phase rendering: Render (pure, interruptible) vs Commit (DOM, effects)
- [ ] WIP Fiber tree, mount/update/unmount timeline, hooks in lifecycle
- [ ] Render phase diagram: setState → render → commit → paint → effects
- [ ] Redux re-renders: useSelector gatekeeper, memoized selectors, batched dispatches
- [ ] StrictMode double-fires: mount → cleanup → mount to catch cleanup bugs
- [ ] One-liner rapid fire answers for speed rounds
- [ ] Concurrency trap questions: not async, no partial UI, opt-in, Redux safe
- [ ] Form patterns: Native vs Formik vs React Hook Form (perf comparison)
- [ ] Controlled forms + Redux: local state → sync on submit/debounce/step
- [ ] Controlled input performance: localize state → debounce → startTransition → memo
