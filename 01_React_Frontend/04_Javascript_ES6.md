# Section 4: JavaScript ES6+ (12 Questions & Answers)



---

## Q40. Explain closures with a practical example. How are they used in React hooks internally?

**Answer:**

A **closure** is a function that remembers and has access to variables from its outer scope, even after the outer function has finished executing.

```javascript
// Simple closure
function createCounter() {
  let count = 0;  // This variable is "enclosed"
 
  return {
    increment: () => ++count,
    getCount: () => count
  };
}

const counter = createCounter();
counter.increment();  // 1
counter.increment();  // 2
counter.getCount();   // 2
// `count` is not accessible from outside, but the returned functions can access it
```

**How closures work in React hooks:**
```jsx
function useState(initialValue) {
  let _val = initialValue;  // Variable in closure

  function state() {
    return _val;
  }

  function setState(newVal) {
    _val = newVal;  // Closure allows access to _val
    render();       // Trigger re-render
  }

  return [state, setState];
}

// React's actual implementation is more complex, but closures are the foundation
```

**Practical banking example:**
```javascript
function createBankAccount(initialBalance) {
  let balance = initialBalance;  // Private variable via closure
 
  return {
    deposit(amount) {
      if (amount > 0) balance += amount;
      return balance;
    },
    withdraw(amount) {
      if (amount > 0 && amount <= balance) balance -= amount;
      else console.log('Insufficient balance');
      return balance;
    },
    getBalance() {
      return balance;
    }
  };
}

const account = createBankAccount(10000);
account.deposit(5000);    // 15000
account.withdraw(3000);   // 12000
account.getBalance();     // 12000
// account.balance → undefined (can't access directly — private!)
```

---

## Q41. What is the event loop? Explain microtasks vs macrotasks with examples.

**Answer:**

The **event loop** is how JavaScript handles asynchronous code despite being single-threaded.

```
 ┌──────────────────────────┐
 │       Call Stack          │  ← Executes synchronous code
 └──────────┬───────────────┘
            │
 ┌──────────▼───────────────┐
 │    Microtask Queue        │  ← Promises, queueMicrotask, MutationObserver
 │    (Higher Priority)      │     Runs AFTER current task, BEFORE next macrotask
 └──────────┬───────────────┘
            │
 ┌──────────▼───────────────┐
 │    Macrotask Queue        │  ← setTimeout, setInterval, I/O, UI events
 │    (Lower Priority)       │     Runs one at a time
 └──────────────────────────┘
```

**Example:**
```javascript
console.log('1. Start');

setTimeout(() => {
  console.log('2. setTimeout (macrotask)');
}, 0);

Promise.resolve().then(() => {
  console.log('3. Promise (microtask)');
});

queueMicrotask(() => {
  console.log('4. queueMicrotask (microtask)');
});

console.log('5. End');

// Output:
// 1. Start           ← synchronous
// 5. End             ← synchronous
// 3. Promise         ← microtask (runs before macrotask)
// 4. queueMicrotask  ← microtask
// 2. setTimeout      ← macrotask (runs last)
```

**Real-world impact:**
```javascript
// Banking scenario: price update processing
function processPriceUpdate(newPrice) {
  console.log('Received price:', newPrice);

  // Microtask: Update internal state immediately
  Promise.resolve().then(() => {
    updatePriceDisplay(newPrice);
    console.log('Display updated');
  });

  // Macrotask: Log analytics (can wait)
  setTimeout(() => {
    logPriceChange(newPrice);
    console.log('Analytics logged');
  }, 0);

  console.log('Processing done');
}

// Output:
// Received price: 1500
// Processing done
// Display updated     ← Important UI update happens first (microtask)
// Analytics logged    ← Less important happens later (macrotask)
```

---

## Q42. Explain Promise chaining, `Promise.all`, `Promise.allSettled`, `Promise.race`, and `Promise.any`.

**Answer:**

**Promise Chaining:**
```javascript
// Each .then() returns a new promise
fetchUser('Neelesh')
  .then(user => fetchAccounts(user.id))      // Returns promise
  .then(accounts => fetchBalance(accounts[0]))  // Returns promise
  .then(balance => console.log('Balance:', balance))
  .catch(err => console.error('Error:', err));  // Catches ANY error in the chain
```

**Promise.all — All must succeed:**
```javascript
// Fetch multiple account balances at the same time
// If ANY one fails, the whole thing fails
const balances = await Promise.all([
  fetchBalance('ACC-001'),
  fetchBalance('ACC-002'),
  fetchBalance('ACC-003')
]);
// balances = [50000, 120000, 30000]  (all succeeded)
// If ACC-002 fails → entire Promise.all rejects!
```

**Promise.allSettled — Get results of all, even if some fail:**
```javascript
// Better for dashboard where partial data is okay
const results = await Promise.allSettled([
  fetchBalance('ACC-001'),
  fetchBalance('ACC-002'),  // This might fail
  fetchBalance('ACC-003')
]);

// results = [
//   { status: 'fulfilled', value: 50000 },
//   { status: 'rejected', reason: 'Network error' },  // Failed but others still returned
//   { status: 'fulfilled', value: 30000 }
// ]

// Show what we have, show error for what failed
results.forEach((result, i) => {
  if (result.status === 'fulfilled') showBalance(i, result.value);
  else showError(i, result.reason);
});
```

**Promise.race — First to finish wins (success or failure):**
```javascript
// Timeout pattern for API calls
const result = await Promise.race([
  fetchAccountData(),
  new Promise((_, reject) =>
    setTimeout(() => reject('Request timeout'), 5000)
  )
]);
// If API responds in 3 seconds → result = API data
// If API takes more than 5 seconds → throws 'Request timeout'
```

**Promise.any — First SUCCESS wins (ignores failures):**
```javascript
// Try multiple servers, use whichever responds first successfully
const data = await Promise.any([
  fetchFromPrimaryServer(),
  fetchFromBackupServer1(),
  fetchFromBackupServer2()
]);
// Uses the first server that succeeds
// Only fails if ALL three fail (AggregateError)
```

**Summary:**

| Method | Behavior | Fails when |
|---|---|---|
| `Promise.all` | Wait for ALL | ANY one fails |
| `Promise.allSettled` | Wait for ALL (with status) | Never fails |
| `Promise.race` | First to finish (success or fail) | First one fails |
| `Promise.any` | First to succeed | ALL fail |

---

## Q43. What is the difference between `var`, `let`, and `const`? Explain temporal dead zone.

**Answer:**

| Feature | `var` | `let` | `const` |
|---|---|---|---|
| Scope | Function scope | Block scope | Block scope |
| Hoisting | Hoisted + initialized as `undefined` | Hoisted but NOT initialized (TDZ) | Hoisted but NOT initialized (TDZ) |
| Re-declaration | Yes | No | No |
| Re-assignment | Yes | Yes | No |

```javascript
// Scope difference
function example() {
  if (true) {
    var x = 10;    // Available in entire function
    let y = 20;    // Only available inside this if block
    const z = 30;  // Only available inside this if block
  }
  console.log(x);  // 10 ✅
  console.log(y);  // ReferenceError ❌
  console.log(z);  // ReferenceError ❌
}
```

**Temporal Dead Zone (TDZ):**
The TDZ is the time between when a variable is hoisted and when it's initialized. You cannot access the variable during this time.

```javascript
// var: NO TDZ (hoisted and initialized as undefined)
console.log(a);  // undefined (not an error!)
var a = 5;

// let/const: HAS TDZ
console.log(b);  // ReferenceError: Cannot access 'b' before initialization
let b = 5;

// TDZ example:
{
  // TDZ for `balance` starts here
  console.log(balance);  // ReferenceError (TDZ!)
  let balance = 50000;   // TDZ ends here
  console.log(balance);  // 50000 ✅
}
```

**Common interview trap — `var` in loops:**
```javascript
// ❌ var in loop (classic closure bug)
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
// Output: 3, 3, 3  (all share the same `i`)

// ✅ let in loop (each iteration gets its own `i`)
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
// Output: 0, 1, 2 ✅
```

---

## Q44. Explain prototypal inheritance in JavaScript. How does `class` syntax relate to it?

**Answer:**

JavaScript doesn't have traditional class-based inheritance. It uses **prototypal inheritance** — objects inherit directly from other objects.

```javascript
// Every object has a hidden [[Prototype]] link
const account = {
  type: 'Generic',
  getType() { return this.type; }
};

const savingsAccount = Object.create(account);  // Inherits from account
savingsAccount.type = 'Savings';
savingsAccount.interestRate = 4;

savingsAccount.getType();  // 'Savings' (own property)
// JavaScript looks: savingsAccount.getType → not found → looks in prototype (account) → found!
```

**Prototype chain:**
```
savingsAccount → account → Object.prototype → null
```

**`class` is just syntactic sugar over prototypes:**
```javascript
// ES6 class syntax
class BankAccount {
  constructor(holder, balance) {
    this.holder = holder;
    this.balance = balance;
  }
  deposit(amount) { this.balance += amount; }
}

class SavingsAccount extends BankAccount {
  constructor(holder, balance, interestRate) {
    super(holder, balance);
    this.interestRate = interestRate;
  }
  addInterest() {
    this.balance += this.balance * this.interestRate / 100;
  }
}

// Under the hood, this is the same as:
function BankAccount(holder, balance) {
  this.holder = holder;
  this.balance = balance;
}
BankAccount.prototype.deposit = function(amount) {
  this.balance += amount;
};

// new keyword: creates object → links prototype → calls constructor
const acc = new SavingsAccount('Neelesh', 50000, 4);
acc.deposit(5000);     // Found on BankAccount.prototype
acc.addInterest();     // Found on SavingsAccount.prototype
```

---

## Q45. What are generators and iterators? Give a practical use case.

**Answer:**

**Iterator** - An object with a `next()` method that returns `{ value, done }`.

**Generator** - A function that can pause and resume execution using `yield`.

```javascript
// Generator function (notice the *)
function* idGenerator() {
  let id = 1;
  while (true) {
    yield id++;  // Pauses here, returns value, resumes when next() called
  }
}

const gen = idGenerator();
gen.next();  // { value: 1, done: false }
gen.next();  // { value: 2, done: false }
gen.next();  // { value: 3, done: false }
// Can keep going forever!
```

**Practical use case — Paginated transaction loading:**
```javascript
function* transactionPaginator(accountId, pageSize = 20) {
  let page = 1;
  let hasMore = true;

  while (hasMore) {
    const response = yield fetch(
      `/api/transactions?account=${accountId}&page=${page}&size=${pageSize}`
    ).then(r => r.json());

    hasMore = response.hasNextPage;
    page++;
  }
}

// Usage:
const paginator = transactionPaginator('ACC-001');
const page1 = paginator.next();       // Fetches page 1
const page2 = paginator.next(data1);  // Fetches page 2
// User clicks "Load More" → paginator.next(previousData) → fetches next page
```

**Generators in Redux Saga:**
```javascript
function* watchTransferSaga() {
  yield takeEvery('TRANSFER_REQUEST', handleTransfer);
}

function* handleTransfer(action) {
  const result = yield call(api.transfer, action.payload);  // Pause until API responds
  yield put({ type: 'TRANSFER_SUCCESS', payload: result }); // Then dispatch success
}
```

---

## Q46. Explain `WeakMap` and `WeakSet`. When would you use them?

**Answer:**

**WeakMap** and **WeakSet** hold "weak" references to objects — if no other reference exists, the object can be garbage collected.

```javascript
// Regular Map: Keeps objects alive (prevents garbage collection)
const cache = new Map();
let user = { name: 'Neelesh' };
cache.set(user, 'some data');
user = null;  // Object is NOT garbage collected because Map still holds reference!

// WeakMap: Allows garbage collection
const weakCache = new WeakMap();
let user2 = { name: 'Neelesh' };
weakCache.set(user2, 'some data');
user2 = null;  // Object CAN be garbage collected! WeakMap doesn't prevent it.
```

**Differences:**

| Feature | Map/Set | WeakMap/WeakSet |
|---|---|---|
| Keys | Any type | Only objects |
| Garbage collection | Prevents | Allows |
| Iterable (for...of) | Yes | No |
| `.size` property | Yes | No |

**Use case 1: Caching component data without memory leaks:**
```javascript
const componentCache = new WeakMap();

function processComponent(component) {
  if (componentCache.has(component)) {
    return componentCache.get(component);  // Return cached result
  }
  const result = expensiveCalculation(component);
  componentCache.set(component, result);
  return result;
}
// When component is unmounted and no references remain, cache entry is auto-cleaned
```

**Use case 2: Private data for objects:**
```javascript
const privateData = new WeakMap();

class BankAccount {
  constructor(pin) {
    privateData.set(this, { pin });  // Truly private!
  }
  verifyPin(input) {
    return privateData.get(this).pin === input;
  }
}

const acc = new BankAccount('1234');
acc.verifyPin('1234');  // true
// No way to access the PIN from outside!
```

---

## Q47. What is debouncing vs throttling? Implement both from scratch.

**Answer:**

- **Debounce:** Wait until user stops doing something, then execute once. ("Wait for silence")
- **Throttle:** Execute at most once every X milliseconds. ("Let through at regular intervals")

```
User typing: a-b-c-d-e (rapid keystrokes)

Debounce (300ms):  ......................execute(e)
                   (waits 300ms after last keystroke)

Throttle (300ms):  execute(a)......execute(c)......execute(e)
                   (executes every 300ms)
```

**Debounce implementation:**
```javascript
function debounce(func, delay) {
  let timer;
  return function(...args) {
    clearTimeout(timer);  // Cancel previous timer
    timer = setTimeout(() => {
      func.apply(this, args);  // Execute after delay
    }, delay);
  };
}

// Usage: Search input
const searchInput = document.getElementById('search');
searchInput.addEventListener('input', debounce((e) => {
  fetchSearchResults(e.target.value);  // API call only after user stops typing
}, 300));
```

**Throttle implementation:**
```javascript
function throttle(func, limit) {
  let inThrottle = false;
  return function(...args) {
    if (!inThrottle) {
      func.apply(this, args);
      inThrottle = true;
      setTimeout(() => {
        inThrottle = false;
      }, limit);
    }
  };
}

// Usage: Scroll event for lazy loading transactions
window.addEventListener('scroll', throttle(() => {
  checkIfNeedToLoadMore();  // Runs at most once every 200ms
}, 200));
```

**When to use which:**

| Debounce | Throttle |
|---|---|
| Search input | Scroll events |
| Window resize (final size) | Mouse move |
| Auto-save after typing | Rate-limiting API calls |
| Form validation | Live price ticker updates |

---

## Q48. Explain the `this` keyword in different contexts.

**Answer:**

```javascript
// 1. Global context
console.log(this);  // Window (browser) or {} (Node.js)

// 2. Object method
const account = {
  balance: 50000,
  getBalance() {
    return this.balance;  // `this` = account object
  }
};
account.getBalance();  // 50000

// 3. Regular function
function showThis() {
  console.log(this);  // Window (non-strict) or undefined (strict mode)
}

// 4. Arrow function — inherits `this` from surrounding scope
const account2 = {
  balance: 50000,
  getBalance: () => {
    return this.balance;  // `this` is NOT account2! It's the outer scope (Window)
  }
};
account2.getBalance();  // undefined! 😱

// 5. Arrow function INSIDE a method (useful pattern)
const account3 = {
  balance: 50000,
  getBalanceAfterDelay() {
    setTimeout(() => {
      console.log(this.balance);  // 50000 ✅ (arrow inherits `this` from getBalanceAfterDelay)
    }, 1000);
  }
};

// 6. Constructor / Class
class BankAccount {
  constructor(balance) {
    this.balance = balance;  // `this` = the new instance
  }
}

// 7. Event handler
button.addEventListener('click', function() {
  console.log(this);  // `this` = the button element
});
button.addEventListener('click', () => {
  console.log(this);  // `this` = outer scope (NOT the button!)
});

// 8. Explicit binding
function greet() { console.log(this.name); }
greet.call({ name: 'Neelesh' });   // 'Neelesh' — call with this context
greet.apply({ name: 'Neelesh' });  // Same, but args as array
const bound = greet.bind({ name: 'Neelesh' });  // Returns a new function with fixed `this`
bound();  // 'Neelesh'
```

**React class component `this` problem:**
```jsx
class OldComponent extends React.Component {
  constructor() {
    super();
    this.handleClick = this.handleClick.bind(this);  // Must bind!
  }
  handleClick() {
    this.setState({ clicked: true });  // Without bind, `this` is undefined
  }
}

// Modern solution: arrow functions (auto-bind)
class OldComponent extends React.Component {
  handleClick = () => {
    this.setState({ clicked: true });  // Arrow function, `this` is always the instance
  };
}
```

---

## Q49. What are Proxy and Reflect in ES6? How are they used in frameworks?

**Answer:**

**Proxy** creates a wrapper around an object that can intercept and customize operations (get, set, delete, etc.).

**Reflect** provides methods for interceptable operations (same methods as Proxy traps).

```javascript
// Basic Proxy example
const account = { balance: 50000, holder: 'Neelesh' };

const secureAccount = new Proxy(account, {
  get(target, prop) {
    console.log(`Reading: ${prop}`);
    return target[prop];
  },
  set(target, prop, value) {
    if (prop === 'balance' && value < 0) {
      throw new Error('Balance cannot be negative!');
    }
    console.log(`Setting ${prop} to ${value}`);
    target[prop] = value;
    return true;
  }
});

secureAccount.balance;        // Logs: "Reading: balance" → 50000
secureAccount.balance = -100; // Throws: "Balance cannot be negative!"
```

**Validation Proxy for financial forms:**
```javascript
function createValidatedForm(rules) {
  const data = {};
  return new Proxy(data, {
    set(target, field, value) {
      if (rules[field]) {
        const error = rules[field](value);
        if (error) throw new Error(`${field}: ${error}`);
      }
      target[field] = value;
      return true;
    }
  });
}

const transferForm = createValidatedForm({
  amount: (v) => v <= 0 ? 'Must be positive' : v > 1000000 ? 'Exceeds limit' : null,
  accountNo: (v) => v.length !== 12 ? 'Must be 12 digits' : null
});

transferForm.amount = 5000;      // ✅ OK
transferForm.amount = -100;      // ❌ Throws: "amount: Must be positive"
transferForm.accountNo = '123';  // ❌ Throws: "accountNo: Must be 12 digits"
```

**Used in frameworks:**
- **Vue.js 3** uses Proxy for its reactivity system (replacing `Object.defineProperty`)
- **MobX** uses Proxy for observable objects
- **Immer** (used by Redux Toolkit) uses Proxy to track mutations

---

## Q50. Explain deep copy vs shallow copy. Pitfalls of `structuredClone` vs `JSON.parse(JSON.stringify())`.

**Answer:**

**Shallow copy:** Copies top-level properties. Nested objects still share the same reference.
**Deep copy:** Copies everything at all levels. Completely independent copy.

```javascript
// Shallow Copy
const original = { name: 'Neelesh', address: { city: 'Pune' } };
const shallow = { ...original };

shallow.name = 'John';          // Only changes shallow
shallow.address.city = 'Mumbai'; // Changes BOTH! (shared reference) 😱

console.log(original.address.city);  // 'Mumbai'! ← Bug!
```

**Deep copy methods:**

```javascript
// Method 1: JSON trick (old way)
const deep1 = JSON.parse(JSON.stringify(original));
// ⚠️ Problems:
// - Loses: functions, undefined, Symbol, Date (becomes string), RegExp, Map, Set
// - Cannot handle circular references
// - Slow for large objects

const obj = {
  date: new Date(),          // Becomes string: "2026-04-22T..."
  fn: () => 'hello',         // LOST! undefined
  undef: undefined,          // LOST!
  regex: /test/g,            // Becomes empty object: {}
  map: new Map([['a', 1]])   // Becomes empty object: {}
};

// Method 2: structuredClone (modern way — built-in)
const deep2 = structuredClone(original);
// ✅ Handles: Date, Map, Set, ArrayBuffer, RegExp, circular references
// ❌ Cannot handle: functions, DOM nodes, Error objects

const obj2 = {
  date: new Date(),           // ✅ Stays Date
  map: new Map([['a', 1]]),   // ✅ Stays Map
  fn: () => 'hello'           // ❌ Error! Can't clone functions
};
const clone = structuredClone(obj2);  // Throws DOMException!
```

**Comparison:**

| Feature | `JSON.parse(JSON.stringify())` | `structuredClone` |
|---|---|---|
| Functions | Silently drops | Throws error |
| Date | Becomes string | Stays Date |
| Map/Set | Becomes empty `{}` | Cloned correctly |
| Circular references | Throws error | Handles correctly |
| `undefined` | Dropped | Preserved |
| Performance | Slower | Faster |
| Browser support | All | Modern browsers (2022+) |

---

## Q51. What are tagged template literals? Give a use case (e.g., styled-components).

**Answer:**

A **tagged template** is a function that processes a template literal. The function receives the string parts and the interpolated values separately.

```javascript
// Basic tagged template
function highlight(strings, ...values) {
  // strings = ['Hello, ', '! Your balance is ₹', '.']
  // values = ['Neelesh', 50000]
  return strings.reduce((result, str, i) => {
    return result + str + (values[i] ? `<b>${values[i]}</b>` : '');
  }, '');
}

const name = 'Neelesh';
const balance = 50000;
const html = highlight`Hello, ${name}! Your balance is ₹${balance}.`;
// "Hello, <b>Neelesh</b>! Your balance is ₹<b>50000</b>."
```

**Use case: SQL injection prevention:**
```javascript
function sql(strings, ...values) {
  return {
    text: strings.join('$'),  // Parameterized query
    values: values             // Safe parameters
  };
}

const userId = "1; DROP TABLE accounts;--";  // SQL injection attempt!
const query = sql`SELECT * FROM accounts WHERE user_id = ${userId}`;
// { text: "SELECT * FROM accounts WHERE user_id = $", values: ["1; DROP TABLE accounts;--"] }
// The value is parameterized, not concatenated — SAFE! ✅
```

**styled-components (React CSS-in-JS library):**
```jsx
import styled from 'styled-components';

// `styled.div` is a TAG FUNCTION
const AccountCard = styled.div`
  background: ${props => props.active ? '#e8f5e9' : '#fff'};
  border: 1px solid ${props => props.active ? '#4caf50' : '#ddd'};
  padding: 16px;
  border-radius: 8px;
`;

// Usage:
<AccountCard active={true}>
  Savings Account — ₹50,000
</AccountCard>
```

---

## Quick Revision Checklist

- [ ] Closures and how React hooks use them internally
- [ ] Event loop: microtasks (Promises) vs macrotasks (setTimeout)
- [ ] Promise.all / allSettled / race / any
- [ ] var (function scope) vs let/const (block scope) + Temporal Dead Zone
- [ ] Prototypal inheritance + class is syntactic sugar
- [ ] Generators (yield/next) for pagination and Redux Saga
- [ ] WeakMap/WeakSet for preventing memory leaks
- [ ] Debounce vs Throttle implementations
- [ ] `this` in 8 different contexts
- [ ] Proxy/Reflect for validation and framework reactivity
- [ ] Deep copy: structuredClone vs JSON trick
- [ ] Tagged template literals (styled-components, SQL safety)

