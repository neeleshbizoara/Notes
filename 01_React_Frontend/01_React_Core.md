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

**Interview tip:** "I keep typing state local and sync to Redux only on meaningful events — submit, step change, or debounce. This keeps forms responsive and Redux state clea