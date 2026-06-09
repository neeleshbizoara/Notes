# Section 2: React Hooks (12 Questions & Answers)

---

## Q16. What are the rules of hooks and why do they exist?

**Answer:**

There are **2 rules:**

**Rule 1: Only call hooks at the top level**
- Don't call hooks inside loops, conditions, or nested functions
- Always call them in the same order

**Rule 2: Only call hooks from React functions**
- Call from React functional components
- Call from custom hooks
- NEVER call from regular JavaScript functions

**Why these rules exist:**

React tracks hooks by their **call order** (like a list: hook #1, hook #2, hook #3). If you put a hook inside an `if` block, the order might change between renders, and React gets confused.

```jsx
// ❌ BAD - Hook inside condition
function AccountPage({ isLoggedIn }) {
  if (isLoggedIn) {
    const [user, setUser] = useState(null);  // Sometimes called, sometimes not!
  }
  const [balance, setBalance] = useState(0);
  // React expects: hook#1 = user, hook#2 = balance
  // But if isLoggedIn is false: hook#1 = balance  😱 WRONG!
}

// ✅ GOOD - Condition inside hook
function AccountPage({ isLoggedIn }) {
  const [user, setUser] = useState(null);      // Always hook #1
  const [balance, setBalance] = useState(0);    // Always hook #2

  useEffect(() => {
    if (isLoggedIn) {
      fetchUser().then(setUser);  // Condition INSIDE the hook
    }
  }, [isLoggedIn]);
}
```

**Install ESLint plugin to catch violations:**
```bash
npm install eslint-plugin-react-hooks
```

---

## Q17. Explain the difference between `useMemo` and `useCallback`. When would you use each?

**Answer:**

Both are for **performance optimization** by memoizing (caching) values.

- `useMemo` → caches the **result** of a function (a computed value)
- `useCallback` → caches the **function itself** (a reference)

```jsx
function TransactionDashboard({ transactions, filter }) {
 
  // useMemo: Caches the RESULT of the calculation
  // Only recalculates when transactions or filter changes
  const filteredTransactions = useMemo(() => {
    console.log('Filtering...');  // Only runs when dependencies change
    return transactions.filter(txn => txn.type === filter);
  }, [transactions, filter]);

  // useCallback: Caches the FUNCTION itself
  // Same function reference is returned unless dependencies change
  const handleDelete = useCallback((id) => {
    deleteTransaction(id);
  }, []);  // Function never changes

  return (
    <div>
      <TransactionList
        data={filteredTransactions}
        onDelete={handleDelete}  
        {/* Without useCallback, a NEW function is created every render,
            causing TransactionList to re-render unnecessarily */}
      />
    </div>
  );
}

// This only re-renders when its props actually change
const TransactionList = React.memo(({ data, onDelete }) => {
  return data.map(txn => (
    <div key={txn.id}>
      {txn.description} - ₹{txn.amount}
      <button onClick={() => onDelete(txn.id)}>Delete</button>
    </div>
  ));
});
```

**When to use:**

| Use `useMemo` when... | Use `useCallback` when... |
|---|---|
| Expensive calculations (sorting, filtering large data) | Passing functions to `React.memo` child components |
| Creating objects/arrays passed as props | Function is a dependency of `useEffect` |
| Derived data from state | Event handlers passed to many list items |

**When NOT to use (common mistake):**
```jsx
// ❌ Don't memoize simple/cheap operations
const fullName = useMemo(() => firstName + ' ' + lastName, [firstName, lastName]);
// Just do: const fullName = firstName + ' ' + lastName;
```

---

## Q18. What is the `useRef` hook? Explain a use case beyond DOM references.

**Answer:**

`useRef` creates a **mutable reference** that persists across renders without causing re-renders when changed.

**Use Case 1: DOM Reference (common)**
```jsx
function SearchInput() {
  const inputRef = useRef(null);

  useEffect(() => {
    inputRef.current.focus();  // Auto-focus on mount
  }, []);

  return <input ref={inputRef} placeholder="Search transactions..." />;
}
```

**Use Case 2: Storing previous value (beyond DOM)**
```jsx
function AccountBalance({ balance }) {
  const prevBalanceRef = useRef();

  useEffect(() => {
    prevBalanceRef.current = balance;  // Store after each render
  });

  const prevBalance = prevBalanceRef.current;

  return (
    <div>
      <p>Current Balance: ₹{balance}</p>
      <p>Previous Balance: ₹{prevBalance}</p>
      {balance > prevBalance && <span className="green">↑ Increased</span>}
      {balance < prevBalance && <span className="red">↓ Decreased</span>}
    </div>
  );
}
```

**Use Case 3: Storing interval/timeout IDs**
```jsx
function AutoRefreshDashboard() {
  const intervalRef = useRef(null);

  function startAutoRefresh() {
    intervalRef.current = setInterval(() => {
      fetchLatestData();
    }, 5000);
  }

  function stopAutoRefresh() {
    clearInterval(intervalRef.current);
  }

  useEffect(() => {
    startAutoRefresh();
    return () => clearInterval(intervalRef.current);  // Cleanup
  }, []);

  return <button onClick={stopAutoRefresh}>Stop Auto-Refresh</button>;
}
```

**Use Case 4: Tracking if component is mounted (avoid memory leaks)**
```jsx
function TransactionLoader() {
  const isMounted = useRef(true);

  useEffect(() => {
    fetchTransactions().then(data => {
      if (isMounted.current) {  // Only update state if still mounted
        setTransactions(data);
      }
    });

    return () => { isMounted.current = false; };
  }, []);
}
```

**Key difference: `useRef` vs `useState`:**

| `useRef` | `useState` |
|---|---|
| Changing `.current` does NOT re-render | Changing state DOES re-render |
| Value persists across renders | Value also persists |
| Good for "silent" data storage | Good for UI-affecting data |

---

## Q19. How does `useEffect` cleanup work? Explain the order of execution with multiple effects.

**Answer:**

**Cleanup** runs before the effect runs again and when the component unmounts.

```jsx
function ChatRoom({ roomId }) {
  useEffect(() => {
    console.log(`Connecting to room ${roomId}`);
    const connection = connectToRoom(roomId);

    return () => {
      console.log(`Disconnecting from room ${roomId}`);
      connection.close();  // CLEANUP: runs before next effect or unmount
    };
  }, [roomId]);
}

// If roomId changes from "general" to "trading":
// 1. "Disconnecting from room general"  ← cleanup of old effect
// 2. "Connecting to room trading"       ← new effect runs
```

**Order of execution with multiple effects:**
```jsx
function Dashboard() {
  useEffect(() => {
    console.log('Effect A: setup');
    return () => console.log('Effect A: cleanup');
  });

  useEffect(() => {
    console.log('Effect B: setup');
    return () => console.log('Effect B: cleanup');
  });

  console.log('Render');
}

// First render:
// "Render"
// "Effect A: setup"
// "Effect B: setup"

// Re-render:
// "Render"
// "Effect A: cleanup"   ← ALL cleanups run first (in order)
// "Effect B: cleanup"
// "Effect A: setup"     ← THEN all setups run (in order)
// "Effect B: setup"

// Unmount:
// "Effect A: cleanup"
// "Effect B: cleanup"
```

### Why is my `useEffect` cleanup running on every render → commit cycle?

If your cleanup function runs on **every single render-to-commit cycle**, React is repeatedly stopping the previous effect and starting a new one. This is normal **only when the effect is supposed to re-synchronize**. If it happens unexpectedly, usually one of these three things is happening.

**Lifecycle flow:**
```
1. Render Phase
   React calculates the next UI

2. Commit Phase
   React updates the real DOM

3. Cleanup Old Effect
   Runs with OLD props/state

4. Run New Effect
   Runs with NEW props/state
```

**Important mental model:** cleanup does **not** only mean unmount. It also runs before the next effect setup whenever dependencies change.

---

#### Cause 1: Missing dependency array — most common

If you omit the dependency array completely, React runs the effect after **every commit**. If the effect returns cleanup, that cleanup also runs before every next effect execution.

```jsx
// ❌ Broken: no dependency array
function PriceTicker() {
  useEffect(() => {
    console.log('Setup runs');

    return () => {
      console.log('Cleanup runs before EVERY next commit effect');
    };
  }); // 🚨 Missing dependency array
}
```

**Fix based on intention:**

```jsx
// ✅ Run setup once on mount, cleanup once on unmount
useEffect(() => {
  const connection = connectToMarketFeed();
  return () => connection.close();
}, []);

// ✅ Re-run only when userId changes
useEffect(() => {
  const subscription = subscribeToUser(userId);
  return () => subscription.unsubscribe();
}, [userId]);
```

**Interview line:** "No dependency array means after every commit. Empty dependency array means mount/unmount. A filled dependency array means mount plus re-run when one dependency changes."

---

#### Cause 2: Unstable object, array, or function dependencies

React compares dependencies using `Object.is`. Primitive values like strings and numbers compare by value, but objects, arrays, and functions compare by **reference**. If you create them inside the component body, they are new on every render.

```jsx
// ❌ Broken: options is a new object on every render
function ChatRoom({ roomId }) {
  const options = {
    serverUrl: 'wss://prices.bank.com',
    roomId,
  };

  useEffect(() => {
    const connection = connect(options);
    connection.open();
    return () => connection.close();
  }, [options]); // 🚨 Different reference every render

  return <h1>Room: {roomId}</h1>;
}
```

Even if `options.roomId` did not change, this is still true:

```jsx
Object.is(
  { roomId: 'general' },
  { roomId: 'general' }
); // false
```

**Better fix 1 — create object inside the effect and depend on primitives:**

```jsx
// ✅ Best: depend on primitive values
function ChatRoom({ roomId }) {
  useEffect(() => {
    const options = {
      serverUrl: 'wss://prices.bank.com',
      roomId,
    };

    const connection = connect(options);
    connection.open();
    return () => connection.close();
  }, [roomId]);
}
```

**Better fix 2 — move static values outside component:**

```jsx
const serverUrl = 'wss://prices.bank.com';

function ChatRoom({ roomId }) {
  useEffect(() => {
    const connection = connect({ serverUrl, roomId });
    connection.open();
    return () => connection.close();
  }, [roomId]);
}
```

**Last-resort fix — memoize references:**

```jsx
const options = useMemo(() => ({
  serverUrl: 'wss://prices.bank.com',
  roomId,
}), [roomId]);

useEffect(() => {
  const connection = connect(options);
  connection.open();
  return () => connection.close();
}, [options]);
```

**React 19 note:** The React Compiler can reduce some manual memoization in compiler-compatible code, but you should still understand dependency identity. For effects, the cleanest fix is often to move object/function creation **inside the effect** or depend on primitive values.

---

#### Cause 3: React Strict Mode in development

In development, `React.StrictMode` intentionally runs this sequence on first mount:

```
Setup → Cleanup → Setup
```

This is a stress test. React is checking whether your cleanup correctly reverses your setup.

```jsx
useEffect(() => {
  const ws = new WebSocket('wss://prices.bank.com/live');
  console.log('Connected');

  return () => {
    ws.close();
    console.log('Disconnected');
  };
}, []);
```

In development you may see:

```
Connected
Disconnected
Connected
```

**Do not "fix" this with a ref just to stop the second execution.** If the duplicate dev run reveals duplicate subscriptions, timers, or connections, the real fix is proper cleanup. This extra setup-cleanup-setup cycle does **not** happen in production builds.

---

#### Debugging checklist

| Symptom | Likely Cause | Fix |
|---|---|---|
| Cleanup runs on every render | No dependency array | Add `[]` or correct dependencies |
| Cleanup runs when unrelated input changes | Object/function dependency changes every render | Move object/function inside effect or depend on primitives |
| Cleanup runs twice only on first mount in dev | `React.StrictMode` | Keep cleanup correct; production is not double-mounted |
| Effect reconnects on every message/count update | Effect depends on state it updates | Use functional state update or split effects |
| Effect creates infinite loop | Effect sets state that changes its own dependencies | Add condition, refactor, or remove unnecessary effect |

**Senior interview summary:**
> "An effect is a synchronization process. React starts it after commit and stops the previous one before starting the next one if dependencies changed. If cleanup happens every render, I first check for a missing dependency array, unstable object/function dependencies, or Strict Mode's development-only remount behavior."

**Banking app real example — WebSocket + Analytics:**
```jsx
function TradingDashboard({ symbol }) {
  // Effect 1: WebSocket connection
  useEffect(() => {
    const ws = new WebSocket(`wss://prices.bank.com/${symbol}`);
    ws.onmessage = (e) => updatePrice(JSON.parse(e.data));
    return () => ws.close();
  }, [symbol]);

  // Effect 2: Analytics tracking
  useEffect(() => {
    trackPageView(`/trading/${symbol}`);
    // No cleanup needed for analytics
  }, [symbol]);

  // Effect 3: Document title
  useEffect(() => {
    document.title = `Trading - ${symbol}`;
    return () => { document.title = 'Banking App'; };
  }, [symbol]);
}
```

---

## Q20. Write a custom hook `useDebounce` for a search input in a financial data lookup.

**Answer:**

**Debounce** = Wait until user stops typing for a certain time, then execute. Prevents API calls on every keystroke.

```jsx
// Custom Hook: useDebounce
function useDebounce(value, delay = 500) {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    return () => clearTimeout(timer);  // Cancel timer if value changes again
  }, [value, delay]);

  return debouncedValue;
}
```

**Usage in Financial Data Lookup:**
```jsx
function StockSearch() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [loading, setLoading] = useState(false);

  const debouncedQuery = useDebounce(query, 300);

  useEffect(() => {
    if (!debouncedQuery) {
      setResults([]);
      return;
    }

    setLoading(true);
    fetch(`/api/stocks/search?q=${debouncedQuery}`)
      .then(res => res.json())
      .then(data => {
        setResults(data);
        setLoading(false);
      });
  }, [debouncedQuery]);  // Only fires when debounced value changes

  return (
    <div>
      <input
        placeholder="Search stocks (e.g., RELIANCE, TCS)..."
        value={query}
        onChange={(e) => setQuery(e.target.value)}
      />
      {loading && <p>Searching...</p>}
      {results.map(stock => (
        <div key={stock.symbol}>
          {stock.symbol} - {stock.name} - ₹{stock.price}
        </div>
      ))}
    </div>
  );
}
```

**How it works step by step:**
```
User types: "R" → timer starts (300ms)
User types: "RE" → old timer cancelled, new timer starts
User types: "REL" → old timer cancelled, new timer starts
User stops typing → 300ms passes → API called with "REL"
```

**Without debounce:** 3 API calls (R, RE, REL)
**With debounce:** 1 API call (REL) ✅

---
## Q21. What is `useReducer`? When would you prefer it over `useState`?

**Answer:**

`useReducer` is like `useState` but for complex state logic. It's inspired by Redux.

```jsx
// Syntax
const [state, dispatch] = useReducer(reducer, initialState);
```

**When to use `useReducer` over `useState`:**
- State has multiple related values
- Next state depends on previous state
- Complex state transitions
- State logic you want to test independently

**Banking example — Transaction Form:**
```jsx
const initialState = {
  amount: '',
  fromAccount: '',
  toAccount: '',
  note: '',
  loading: false,
  error: null,
  success: false
};

function transactionReducer(state, action) {
  switch (action.type) {
    case 'UPDATE_FIELD':
      return { ...state, [action.field]: action.value, error: null };
    case 'SUBMIT_START':
      return { ...state, loading: true, error: null };
    case 'SUBMIT_SUCCESS':
      return { ...initialState, success: true };
    case 'SUBMIT_ERROR':
      return { ...state, loading: false, error: action.error };
    case 'RESET':
      return initialState;
    default:
      return state;
  }
}

function TransferForm() {
  const [state, dispatch] = useReducer(transactionReducer, initialState);

  function handleChange(field, value) {
    dispatch({ type: 'UPDATE_FIELD', field, value });
  }

  async function handleSubmit() {
    dispatch({ type: 'SUBMIT_START' });
    try {
      await transferMoney(state);
      dispatch({ type: 'SUBMIT_SUCCESS' });
    } catch (err) {
      dispatch({ type: 'SUBMIT_ERROR', error: err.message });
    }
  }

  return (
    <form>
      <input
        value={state.amount}
        onChange={(e) => handleChange('amount', e.target.value)}
      />
      {state.loading && <Spinner />}
      {state.error && <Error message={state.error} />}
      {state.success && <p>Transfer successful!</p>}
      <button onClick={handleSubmit} disabled={state.loading}>
        Transfer
      </button>
    </form>
  );
}
```

**Comparison:**

| `useState` | `useReducer` |
|---|---|
| Simple, independent state values | Related state values |
| `setCount(count + 1)` | `dispatch({ type: 'INCREMENT' })` |
| Logic in component | Logic in reducer (testable separately) |
| Good for 1-3 state variables | Good for 4+ related state variables |

---

## Q22. Explain `useLayoutEffect` vs `useEffect`. When is `useLayoutEffect` necessary?

**Answer:**

| `useEffect` | `useLayoutEffect` |
|---|---|
| Runs **after** browser paints | Runs **before** browser paints |
| Non-blocking (doesn't delay paint) | Blocking (delays paint until done) |
| Use for most side effects | Use when you need to measure/change DOM before user sees it |

```
Render → DOM updated → useLayoutEffect → Browser Paints → useEffect
```

**When `useLayoutEffect` is necessary:**

```jsx
// ❌ useEffect causes a FLASH (flicker)
function Tooltip({ targetRef, text }) {
  const [position, setPosition] = useState({ top: 0, left: 0 });

  useEffect(() => {
    const rect = targetRef.current.getBoundingClientRect();
    setPosition({ top: rect.bottom, left: rect.left });
    // User sees tooltip at (0,0) briefly, then it jumps to correct position 😟
  }, []);

  return <div style={{ position: 'absolute', ...position }}>{text}</div>;
}

// ✅ useLayoutEffect - no flash!
function Tooltip({ targetRef, text }) {
  const [position, setPosition] = useState({ top: 0, left: 0 });

  useLayoutEffect(() => {
    const rect = targetRef.current.getBoundingClientRect();
    setPosition({ top: rect.bottom, left: rect.left });
    // Runs BEFORE paint, so user never sees wrong position ✅
  }, []);

  return <div style={{ position: 'absolute', ...position }}>{text}</div>;
}
```

**Rule of thumb:** Use `useEffect` by default. Only use `useLayoutEffect` when:
- You need to measure DOM elements (width, height, position)
- You need to adjust layout before user sees it
- You see visual flickers with `useEffect`

---

## Q23. How would you build a custom hook `useFetch` with caching, retry logic, and abort support?

**Answer:**

```jsx
const cache = new Map();

function useFetch(url, options = {}) {
  const { retries = 3, retryDelay = 1000, cacheTime = 60000 } = options;
 
  const [data, setData] = useState(null);
  const [error, setError] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const controller = new AbortController();  // For abort support
    let retryCount = 0;

    async function fetchData() {
      // Check cache first
      const cached = cache.get(url);
      if (cached && Date.now() - cached.timestamp < cacheTime) {
        setData(cached.data);
        setLoading(false);
        return;
      }

      setLoading(true);
      setError(null);

      async function attempt() {
        try {
          const response = await fetch(url, { signal: controller.signal });
         
          if (!response.ok) throw new Error(`HTTP ${response.status}`);
         
          const result = await response.json();
         
          // Store in cache
          cache.set(url, { data: result, timestamp: Date.now() });
         
          setData(result);
          setLoading(false);
        } catch (err) {
          if (err.name === 'AbortError') return;  // Ignore abort errors
         
          if (retryCount < retries) {
            retryCount++;
            console.log(`Retry ${retryCount}/${retries} for ${url}`);
            setTimeout(attempt, retryDelay * retryCount);  // Exponential backoff
          } else {
            setError(err.message);
            setLoading(false);
          }
        }
      }

      attempt();
    }

    fetchData();

    return () => controller.abort();  // Abort on unmount or URL change
  }, [url, retries, retryDelay, cacheTime]);

  return { data, error, loading };
}
```

**Usage:**
```jsx
function AccountDashboard() {
  const { data: accounts, loading, error } = useFetch('/api/accounts', {
    retries: 3,
    retryDelay: 1000,
    cacheTime: 30000  // Cache for 30 seconds
  });

  if (loading) return <Spinner />;
  if (error) return <ErrorMessage message={error} />;

  return (
    <div>
      {accounts.map(acc => (
        <AccountCard key={acc.id} account={acc} />
      ))}
    </div>
  );
}
```

---

## Q24. What are the pitfalls of stale closures in hooks? How do you solve them?

**Answer:**

A **stale closure** happens when a function "remembers" an old value of a variable because it was captured at the time the function was created.

```jsx
// ❌ Stale closure problem
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const interval = setInterval(() => {
      console.log(count);       // Always logs 0! Stale closure!
      setCount(count + 1);      // Always sets to 1! Never increments!
    }, 1000);

    return () => clearInterval(interval);
  }, []);  // Empty dependency = captures initial count (0) forever
}
```

**Solutions:**

**Solution 1: Functional update (best for setState)**
```jsx
useEffect(() => {
  const interval = setInterval(() => {
    setCount(prev => prev + 1);  // ✅ Uses latest value via callback
  }, 1000);
  return () => clearInterval(interval);
}, []);
```

**Solution 2: Add dependency (re-creates effect)**
```jsx
useEffect(() => {
  const interval = setInterval(() => {
    console.log(count);  // ✅ Now has latest count
    setCount(count + 1);
  }, 1000);
  return () => clearInterval(interval);
}, [count]);  // Re-runs effect when count changes
```

**Solution 3: useRef (for reading latest value without re-running effect)**
```jsx
function PriceTracker({ symbol }) {
  const [price, setPrice] = useState(0);
  const priceRef = useRef(price);
 
  useEffect(() => {
    priceRef.current = price;  // Always keep ref updated
  }, [price]);

  useEffect(() => {
    const ws = new WebSocket(`wss://prices.bank.com/${symbol}`);
    ws.onmessage = (e) => {
      const newPrice = JSON.parse(e.data).price;
      const oldPrice = priceRef.current;  // ✅ Always latest!
     
      if (Math.abs(newPrice - oldPrice) > 100) {
        alert('Big price movement!');
      }
      setPrice(newPrice);
    };
    return () => ws.close();
  }, [symbol]);
}
```

---


## Q25. Explain `useTransition` and `useDeferredValue` from React 18. Give a financial dashboard use case.

**Answer:**

Both are for **prioritizing updates** — keeping the UI responsive while heavy work happens.

**`useTransition`** — Marks a state update as low priority
```jsx
function StockScreener() {
  const [filter, setFilter] = useState('');
  const [filteredStocks, setFilteredStocks] = useState([]);
  const [isPending, startTransition] = useTransition();

  function handleFilterChange(e) {
    const value = e.target.value;
    setFilter(value);  // HIGH priority: update input immediately

    startTransition(() => {
      // LOW priority: can be interrupted
      const result = stocks.filter(s =>
        s.name.includes(value) || s.sector.includes(value)
      );
      setFilteredStocks(result);  // Heavy operation on 5000+ stocks
    });
  }

  return (
    <div>
      <input value={filter} onChange={handleFilterChange} />
      {isPending && <Spinner />}
      <StockTable data={filteredStocks} />  {/* 5000+ rows */}
    </div>
  );
}
```

**`useDeferredValue`** — Creates a deferred version of a value
```jsx
function TransactionSearch({ transactions }) {
  const [query, setQuery] = useState('');
  const deferredQuery = useDeferredValue(query);
  // query updates immediately, deferredQuery updates when React has time

  const filteredTxns = useMemo(() => {
    return transactions.filter(t =>
      t.description.toLowerCase().includes(deferredQuery.toLowerCase())
    );
  }, [deferredQuery, transactions]);

  return (
    <div>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <div style={{ opacity: query !== deferredQuery ? 0.6 : 1 }}>
        {/* Dims while filtering is catching up */}
        <TransactionList data={filteredTxns} />
      </div>
    </div>
  );
}
```

**Difference:**

| `useTransition` | `useDeferredValue` |
|---|---|
| You control which setState is low priority | React defers a value automatically |
| You wrap the setter: `startTransition(() => {...})` | You wrap the value: `useDeferredValue(value)` |
| Use when you own the state update | Use when you receive a value as prop |

---

## Q26. How do you test custom hooks? What tools/patterns do you use?

**Answer:**

Use `@testing-library/react`'s `renderHook` utility.

```jsx
// hook: useCounter.js
function useCounter(initialValue = 0) {
  const [count, setCount] = useState(initialValue);
  const increment = () => setCount(c => c + 1);
  const decrement = () => setCount(c => c - 1);
  const reset = () => setCount(initialValue);
  return { count, increment, decrement, reset };
}
```

```jsx
// test: useCounter.test.js
import { renderHook, act } from '@testing-library/react';
import useCounter from './useCounter';

test('should initialize with default value', () => {
  const { result } = renderHook(() => useCounter());
  expect(result.current.count).toBe(0);
});

test('should initialize with custom value', () => {
  const { result } = renderHook(() => useCounter(10));
  expect(result.current.count).toBe(10);
});

test('should increment', () => {
  const { result } = renderHook(() => useCounter());
  act(() => {
    result.current.increment();
  });
  expect(result.current.count).toBe(1);
});

test('should reset to initial value', () => {
  const { result } = renderHook(() => useCounter(5));
  act(() => {
    result.current.increment();
    result.current.increment();
    result.current.reset();
  });
  expect(result.current.count).toBe(5);
});
```

**Testing hooks with async operations:**
```jsx
test('useFetch should return data', async () => {
  global.fetch = jest.fn(() =>
    Promise.resolve({ ok: true, json: () => Promise.resolve({ balance: 50000 }) })
  );

  const { result } = renderHook(() => useFetch('/api/balance'));

  // Wait for async operation
  await waitFor(() => {
    expect(result.current.loading).toBe(false);
  });

  expect(result.current.data).toEqual({ balance: 50000 });
});
```

---

## Q27. What is `useId` hook in React 18? Why was it introduced?

**Answer:**

`useId` generates a **unique ID** that is consistent between server and client rendering (SSR-safe).

**Problem it solves:**
```jsx
// ❌ BAD - IDs might clash or differ between server and client
function LoginForm() {
  return (
    <div>
      <label htmlFor="email">Email</label>
      <input id="email" />  {/* What if two LoginForms on the same page? */}
    </div>
  );
}
```

```jsx
// ✅ GOOD - useId generates unique, SSR-safe IDs
function LoginForm() {
  const id = useId();

  return (
    <div>
      <label htmlFor={`${id}-email`}>Email</label>
      <input id={`${id}-email`} />

      <label htmlFor={`${id}-password`}>Password</label>
      <input id={`${id}-password`} type="password" />
    </div>
  );
}

// First instance: id = ":r1:" → ":r1:-email", ":r1:-password"
// Second instance: id = ":r2:" → ":r2:-email", ":r2:-password"
// No clashes! ✅
```

**Accessibility use case (aria-describedby):**
```jsx
function AmountField({ error }) {
  const id = useId();
 
  return (
    <div>
      <label htmlFor={`${id}-amount`}>Transfer Amount</label>
      <input
        id={`${id}-amount`}
        aria-describedby={error ? `${id}-error` : undefined}
      />
      {error && <span id={`${id}-error`} role="alert">{error}</span>}
    </div>
  );
}
```

**Important:** Don't use `useId` for list keys. It's for HTML `id` attributes only.

---


## Quick Revision Checklist

- [ ] Rules of hooks (top-level, React functions only)
- [ ] `useMemo` (cache value) vs `useCallback` (cache function)
- [ ] `useRef` for DOM refs, previous values, interval IDs
- [ ] `useEffect` cleanup order with multiple effects
- [ ] Why `useEffect` cleanup runs on every render/commit
- [ ] Custom hook: `useDebounce` pattern
- [ ] `useReducer` for complex related state
- [ ] `useLayoutEffect` runs before paint (for DOM measurements)
- [ ] `useFetch` with caching, retry, and abort
- [ ] Stale closures and 3 solutions
- [ ] `useTransition` vs `useDeferredValue`
- [ ] Testing hooks with `renderHook` and `act`
- [ ] `useId` for SSR-safe unique IDs




