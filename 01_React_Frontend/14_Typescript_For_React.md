# Section 14: TypeScript for Enterprise React (15 Questions & Answers)
---

## Q119. What are the essential TypeScript patterns for typing React components?

**Answer:**

### Function Components — 3 Approaches

```tsx
// Approach 1: Inline props type (PREFERRED — most explicit)
function Button({ label, onClick, disabled = false }: {
  label: string;
  onClick: () => void;
  disabled?: boolean;
}) {
  return <button onClick={onClick} disabled={disabled}>{label}</button>;
}

// Approach 2: Named interface (use when props are reused or complex)
interface ButtonProps {
  label: string;
  onClick: () => void;
  disabled?: boolean;
  variant?: 'primary' | 'secondary' | 'outline';
}

function Button({ label, onClick, disabled = false, variant = 'primary' }: ButtonProps) {
  return <button className={`btn-${variant}`} onClick={onClick} disabled={disabled}>{label}</button>;
}

// Approach 3: React.FC (AVOID — adds implicit children, less explicit)
const Button: React.FC<ButtonProps> = ({ label, onClick }) => { ... };
// ❌ Don't use — hides children prop, causes confusion
```

### Children Typing

```tsx
// Explicit children typing (PREFERRED)
interface CardProps {
  title: string;
  children: React.ReactNode;  // Most permissive — accepts anything renderable
}

// For specific children requirements
interface LayoutProps {
  children: React.ReactElement;        // Only a single React element
  header: React.ReactNode;             // Any renderable content
  renderFooter: () => React.ReactNode; // Render prop
}

// ReactNode vs ReactElement vs JSX.Element
// ReactNode = string | number | boolean | null | undefined | ReactElement | ReactFragment
// ReactElement = { type, props, key } — an instantiated component
// JSX.Element = ReactElement (identical in practice)
```

### Event Typing

```tsx
function SearchInput() {
  // Inline event typing
  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    console.log(e.target.value);
  };

  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
  };

  const handleKeyDown = (e: React.KeyboardEvent<HTMLInputElement>) => {
    if (e.key === 'Enter') { /* submit */ }
  };

  const handleClick = (e: React.MouseEvent<HTMLButtonElement>) => {
    console.log(e.clientX, e.clientY);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input onChange={handleChange} onKeyDown={handleKeyDown} />
      <button onClick={handleClick}>Search</button>
    </form>
  );
}
```

---

## Q120. How do you use Generics in React components?

**Answer:**

Generics let you create **reusable components that preserve type information** of the data they handle.

### Generic List Component

```tsx
// Without generics — loses type info
interface ListProps {
  items: any[];  // ❌ No type safety
  renderItem: (item: any) => React.ReactNode;
}

// With generics — preserves type info ✅
interface ListProps<T> {
  items: T[];
  renderItem: (item: T) => React.ReactNode;
  keyExtractor: (item: T) => string;
}

function List<T>({ items, renderItem, keyExtractor }: ListProps<T>) {
  return (
    <ul>
      {items.map(item => (
        <li key={keyExtractor(item)}>{renderItem(item)}</li>
      ))}
    </ul>
  );
}

// Usage — TypeScript infers T from items
interface User { id: string; name: string; email: string; }

<List<User>
  items={users}
  keyExtractor={(user) => user.id}      // ✅ user is typed as User
  renderItem={(user) => <span>{user.name}</span>}  // ✅ autocomplete works
/>
```

### Generic Table Component

```tsx
interface Column<T> {
  key: keyof T;
  header: string;
  render?: (value: T[keyof T], row: T) => React.ReactNode;
  sortable?: boolean;
}

interface DataTableProps<T> {
  data: T[];
  columns: Column<T>[];
  onRowClick?: (row: T) => void;
}

function DataTable<T extends { id: string | number }>({
  data, columns, onRowClick
}: DataTableProps<T>) {
  return (
    <table>
      <thead>
        <tr>{columns.map(col => <th key={String(col.key)}>{col.header}</th>)}</tr>
      </thead>
      <tbody>
        {data.map(row => (
          <tr key={row.id} onClick={() => onRowClick?.(row)}>
            {columns.map(col => (
              <td key={String(col.key)}>
                {col.render ? col.render(row[col.key], row) : String(row[col.key])}
              </td>
            ))}
          </tr>
        ))}
      </tbody>
    </table>
  );
}

// Usage
interface Transaction {
  id: string;
  amount: number;
  date: string;
  status: 'pending' | 'completed' | 'failed';
}

const columns: Column<Transaction>[] = [
  { key: 'date', header: 'Date', sortable: true },
  { key: 'amount', header: 'Amount', render: (val) => `$${val}` },
  { key: 'status', header: 'Status', render: (val) => <Badge status={val} /> },
];

<DataTable<Transaction> data={transactions} columns={columns} />
```

### Generic Custom Hook

```tsx
function useLocalStorage<T>(key: string, initialValue: T) {
  const [value, setValue] = useState<T>(() => {
    const stored = localStorage.getItem(key);
    return stored ? (JSON.parse(stored) as T) : initialValue;
  });

  const setStoredValue = useCallback((newValue: T | ((prev: T) => T)) => {
    setValue(prev => {
      const resolved = newValue instanceof Function ? newValue(prev) : newValue;
      localStorage.setItem(key, JSON.stringify(resolved));
      return resolved;
    });
  }, [key]);

  return [value, setStoredValue] as const;
}

// Usage — T is inferred from initialValue
const [theme, setTheme] = useLocalStorage('theme', 'dark');  // T = string
const [user, setUser] = useLocalStorage<User | null>('user', null);  // T = User | null
```

---

## Q121. Explain discriminated unions for component variants.

**Answer:**

Discriminated unions ensure **only valid prop combinations** are allowed — no impossible states.

```tsx
// ❌ BAD: Multiple booleans — allows impossible states
interface ButtonProps {
  isLoading?: boolean;
  isDisabled?: boolean;
  isIconOnly?: boolean;
  label?: string;  // Required unless isIconOnly... but TS can't enforce that
  icon?: React.ReactNode;
}
// Allows: { isLoading: true, isDisabled: true, isIconOnly: true, label: undefined }
// That makes no sense!

// ✅ GOOD: Discriminated union — only valid combinations exist
type ButtonProps =
  | { variant: 'standard'; label: string; icon?: React.ReactNode; onClick: () => void }
  | { variant: 'icon-only'; icon: React.ReactNode; ariaLabel: string; onClick: () => void }
  | { variant: 'loading'; label: string }
  | { variant: 'link'; label: string; href: string };

function Button(props: ButtonProps) {
  switch (props.variant) {
    case 'standard':
      return <button onClick={props.onClick}>{props.icon}{props.label}</button>;
    case 'icon-only':
      return <button aria-label={props.ariaLabel} onClick={props.onClick}>{props.icon}</button>;
    case 'loading':
      return <button disabled><Spinner />{props.label}</button>;
    case 'link':
      return <a href={props.href}>{props.label}</a>;
  }
}

// Usage
<Button variant="icon-only" icon={<TrashIcon />} ariaLabel="Delete" onClick={handleDelete} />
<Button variant="standard" label="Submit" onClick={handleSubmit} />
// ❌ TypeScript error: Property 'ariaLabel' is missing
<Button variant="icon-only" icon={<TrashIcon />} onClick={handleDelete} />
```

### Real-world: Modal/Dialog variants

```tsx
type DialogProps =
  | { type: 'alert'; title: string; message: string; onClose: () => void }
  | { type: 'confirm'; title: string; message: string; onConfirm: () => void; onCancel: () => void }
  | { type: 'form'; title: string; children: React.ReactNode; onSubmit: () => void; onCancel: () => void };

function Dialog(props: DialogProps) {
  switch (props.type) {
    case 'alert':
      return <div><h2>{props.title}</h2><p>{props.message}</p><button onClick={props.onClose}>OK</button></div>;
    case 'confirm':
      return (
        <div>
          <h2>{props.title}</h2><p>{props.message}</p>
          <button onClick={props.onCancel}>Cancel</button>
          <button onClick={props.onConfirm}>Confirm</button>
        </div>
      );
    case 'form':
      return (
        <div>
          <h2>{props.title}</h2>{props.children}
          <button onClick={props.onCancel}>Cancel</button>
          <button onClick={props.onSubmit}>Submit</button>
        </div>
      );
  }
}
```

---

## Q122. How do you type Redux Toolkit (createSlice, selectors, thunks) with TypeScript?

**Answer:**

### Typed Store Setup

```tsx
// store/index.ts
import { configureStore } from '@reduxjs/toolkit';
import { accountsSlice } from './accountsSlice';
import { authSlice } from './authSlice';

export const store = configureStore({
  reducer: {
    accounts: accountsSlice.reducer,
    auth: authSlice.reducer,
  },
});

// Infer types from the store itself — single source of truth
export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

### Typed Hooks (use everywhere instead of raw useSelector/useDispatch)

```tsx
// store/hooks.ts
import { useDispatch, useSelector, TypedUseSelectorHook } from 'react-redux';
import type { RootState, AppDispatch } from './index';

export const useAppDispatch = () => useDispatch<AppDispatch>();
export const useAppSelector: TypedUseSelectorHook<RootState> = useSelector;
```

### Typed Slice

```tsx
// store/accountsSlice.ts
import { createSlice, createAsyncThunk, PayloadAction } from '@reduxjs/toolkit';

interface Account {
  id: string;
  name: string;
  balance: number;
  currency: 'USD' | 'EUR' | 'GBP';
}

interface AccountsState {
  accounts: Account[];
  selectedAccountId: string | null;
  status: 'idle' | 'loading' | 'succeeded' | 'failed';
  error: string | null;
}

const initialState: AccountsState = {
  accounts: [],
  selectedAccountId: null,
  status: 'idle',
  error: null,
};

// Typed Async Thunk
export const fetchAccounts = createAsyncThunk<
  Account[],           // Return type
  void,                // Argument type
  { rejectValue: string }  // ThunkAPI config
>(
  'accounts/fetch',
  async (_, { rejectWithValue }) => {
    try {
      const response = await api.get<Account[]>('/accounts');
      return response.data;
    } catch (err) {
      return rejectWithValue('Failed to fetch accounts');
    }
  }
);

export const accountsSlice = createSlice({
  name: 'accounts',
  initialState,
  reducers: {
    selectAccount(state, action: PayloadAction<string>) {
      state.selectedAccountId = action.payload;
    },
    clearSelection(state) {
      state.selectedAccountId = null;
    },
  },
  extraReducers: (builder) => {
    builder
      .addCase(fetchAccounts.pending, (state) => {
        state.status = 'loading';
      })
      .addCase(fetchAccounts.fulfilled, (state, action) => {
        state.status = 'succeeded';
        state.accounts = action.payload;
      })
      .addCase(fetchAccounts.rejected, (state, action) => {
        state.status = 'failed';
        state.error = action.payload ?? 'Unknown error';
      });
  },
});

export const { selectAccount, clearSelection } = accountsSlice.actions;
```

### Typed Selectors

```tsx
// store/accountsSelectors.ts
import { createSelector } from '@reduxjs/toolkit';
import type { RootState } from './index';

// Simple selector
export const selectAllAccounts = (state: RootState) => state.accounts.accounts;
export const selectAccountsStatus = (state: RootState) => state.accounts.status;

// Memoized derived selector
export const selectTotalBalance = createSelector(
  selectAllAccounts,
  (accounts): number => accounts.reduce((sum, acc) => sum + acc.balance, 0)
);

export const selectSelectedAccount = createSelector(
  selectAllAccounts,
  (state: RootState) => state.accounts.selectedAccountId,
  (accounts, selectedId): Account | undefined =>
    accounts.find(acc => acc.id === selectedId)
);

// Parameterized selector factory
export const makeSelectAccountById = (accountId: string) =>
  createSelector(
    selectAllAccounts,
    (accounts): Account | undefined => accounts.find(acc => acc.id === accountId)
  );
```

### Usage in Components

```tsx
function AccountDashboard() {
  const dispatch = useAppDispatch();
  const accounts = useAppSelector(selectAllAccounts);
  const totalBalance = useAppSelector(selectTotalBalance);
  const status = useAppSelector(selectAccountsStatus);

  useEffect(() => {
    dispatch(fetchAccounts());  // ✅ Fully typed — no `any`
  }, [dispatch]);

  return (
    <div>
      <h2>Total: ${totalBalance}</h2>
      {accounts.map(acc => (
        <AccountCard
          key={acc.id}
          account={acc}  // ✅ TypeScript knows this is Account
          onSelect={() => dispatch(selectAccount(acc.id))}
        />
      ))}
    </div>
  );
}
```

---

## Q123. How do you create type-safe API layers in TypeScript React apps?

**Answer:**

### Typed Axios Instance with Interceptors

```tsx
// api/client.ts
import axios, { AxiosInstance, AxiosRequestConfig, AxiosResponse } from 'axios';

interface ApiError {
  message: string;
  code: string;
  statusCode: number;
}

class ApiClient {
  private client: AxiosInstance;

  constructor(baseURL: string) {
    this.client = axios.create({
      baseURL,
      timeout: 30000,
      headers: { 'Content-Type': 'application/json' },
    });

    this.client.interceptors.response.use(
      (response) => response,
      (error) => {
        const apiError: ApiError = {
          message: error.response?.data?.message ?? 'Network error',
          code: error.response?.data?.code ?? 'UNKNOWN',
          statusCode: error.response?.status ?? 500,
        };
        return Promise.reject(apiError);
      }
    );
  }

  async get<T>(url: string, config?: AxiosRequestConfig): Promise<T> {
    const response: AxiosResponse<T> = await this.client.get(url, config);
    return response.data;
  }

  async post<T, D = unknown>(url: string, data?: D, config?: AxiosRequestConfig): Promise<T> {
    const response: AxiosResponse<T> = await this.client.post(url, data, config);
    return response.data;
  }

  async put<T, D = unknown>(url: string, data?: D, config?: AxiosRequestConfig): Promise<T> {
    const response: AxiosResponse<T> = await this.client.put(url, data, config);
    return response.data;
  }

  async delete<T>(url: string, config?: AxiosRequestConfig): Promise<T> {
    const response: AxiosResponse<T> = await this.client.delete(url, config);
    return response.data;
  }
}

export const apiClient = new ApiClient(import.meta.env.VITE_API_BASE_URL);
```

### Zod Schema Validation (Runtime + Compile-time safety)

```tsx
// api/schemas/account.ts
import { z } from 'zod';

// Define schema — single source of truth for validation + types
export const AccountSchema = z.object({
  id: z.string().uuid(),
  name: z.string().min(1),
  balance: z.number(),
  currency: z.enum(['USD', 'EUR', 'GBP']),
  status: z.enum(['active', 'frozen', 'closed']),
  createdAt: z.string().datetime(),
});

export const AccountListSchema = z.array(AccountSchema);

// Derive TypeScript type FROM the schema
export type Account = z.infer<typeof AccountSchema>;

// API function with runtime validation
export async function fetchAccounts(): Promise<Account[]> {
  const data = await apiClient.get<unknown>('/accounts');
  // Runtime validation — catches API contract violations
  return AccountListSchema.parse(data);
}

// For mutations — separate input schema (no id, createdAt)
export const CreateAccountInput = AccountSchema.omit({ id: true, createdAt: true });
export type CreateAccountInput = z.infer<typeof CreateAccountInput>;

export async function createAccount(input: CreateAccountInput): Promise<Account> {
  const data = await apiClient.post<unknown>('/accounts', input);
  return AccountSchema.parse(data);
}
```

### Type-Safe API Service Layer

```tsx
// api/services/accountService.ts
import { Account, CreateAccountInput, AccountListSchema, AccountSchema } from '../schemas/account';

export const accountService = {
  getAll: async (): Promise<Account[]> => {
    const data = await apiClient.get<unknown>('/accounts');
    return AccountListSchema.parse(data);
  },

  getById: async (id: string): Promise<Account> => {
    const data = await apiClient.get<unknown>(`/accounts/${id}`);
    return AccountSchema.parse(data);
  },

  create: async (input: CreateAccountInput): Promise<Account> => {
    const data = await apiClient.post<unknown>('/accounts', input);
    return AccountSchema.parse(data);
  },

  updateBalance: async (id: string, amount: number): Promise<Account> => {
    const data = await apiClient.put<unknown>(`/accounts/${id}/balance`, { amount });
    return AccountSchema.parse(data);
  },
} as const;
```

---

## Q124. What are utility types and how do you use them in component libraries?

**Answer:**

```tsx
// ============ BUILT-IN UTILITY TYPES ============

// Partial<T> — All properties optional
interface FormValues { name: string; email: string; age: number; }
type FormPatch = Partial<FormValues>;
// { name?: string; email?: string; age?: number }

// Required<T> — All properties required
type StrictFormValues = Required<FormValues>;

// Pick<T, K> — Select specific properties
type LoginFields = Pick<FormValues, 'email'>;
// { email: string }

// Omit<T, K> — Exclude specific properties
type PublicProfile = Omit<User, 'password' | 'ssn'>;
// User without password and ssn

// Record<K, V> — Object with known keys
type ThemeColors = Record<'primary' | 'secondary' | 'danger', string>;
// { primary: string; secondary: string; danger: string }

// ReturnType<T> — Extract return type of a function
const createUser = (name: string, age: number) => ({ name, age, id: crypto.randomUUID() });
type NewUser = ReturnType<typeof createUser>;
// { name: string; age: number; id: string }

// Parameters<T> — Extract parameter types
type CreateUserParams = Parameters<typeof createUser>;
// [string, number]

// Extract / Exclude — Filter union types
type Status = 'active' | 'inactive' | 'pending' | 'deleted';
type ActiveStatuses = Extract<Status, 'active' | 'pending'>;  // 'active' | 'pending'
type ArchivedStatuses = Exclude<Status, 'active' | 'pending'>; // 'inactive' | 'deleted'

// ============ COMPONENT LIBRARY PATTERNS ============

// Extending HTML element props
interface CustomInputProps extends React.InputHTMLAttributes<HTMLInputElement> {
  label: string;
  error?: string;
  helperText?: string;
}
// Now CustomInputProps includes ALL native input props (onChange, value, placeholder, etc.)
// Plus our custom ones (label, error, helperText)

function CustomInput({ label, error, helperText, ...inputProps }: CustomInputProps) {
  return (
    <div>
      <label>{label}</label>
      <input {...inputProps} aria-invalid={!!error} />
      {error && <span role="alert">{error}</span>}
      {helperText && <span>{helperText}</span>}
    </div>
  );
}

// Polymorphic component with "as" prop
type PolymorphicProps<E extends React.ElementType> = {
  as?: E;
  children: React.ReactNode;
} & Omit<React.ComponentPropsWithoutRef<E>, 'as' | 'children'>;

function Text<E extends React.ElementType = 'span'>({
  as,
  children,
  ...props
}: PolymorphicProps<E>) {
  const Component = as || 'span';
  return <Component {...props}>{children}</Component>;
}

// Usage
<Text>Default span</Text>
<Text as="h1" className="title">Heading</Text>
<Text as="a" href="/home">Link text</Text>  // ✅ href is valid because as="a"
```

---

## Q125. How do you enforce strict TypeScript in a team codebase?

**Answer:**

### tsconfig.json — Strict Configuration

```json
{
  "compilerOptions": {
    "strict": true,                    // Enables ALL strict checks below
    "noImplicitAny": true,             // No implicit 'any' types
    "strictNullChecks": true,          // null/undefined must be handled
    "strictFunctionTypes": true,       // Strict function type checking
    "strictPropertyInitialization": true, // Class properties must be initialized
    "noImplicitReturns": true,         // All code paths must return
    "noFallthroughCasesInSwitch": true,// No fallthrough in switch
    "noUncheckedIndexedAccess": true,  // arr[0] is T | undefined (IMPORTANT!)
    "exactOptionalPropertyTypes": true,// undefined ≠ optional
    "noImplicitOverride": true,        // Override keyword required
    "forceConsistentCasingInFileNames": true,
    "isolatedModules": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "jsx": "react-jsx",
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@components/*": ["src/components/*"],
      "@hooks/*": ["src/hooks/*"]
    }
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

### ESLint TypeScript Rules

```js
// eslint.config.ts (flat config)
import tseslint from 'typescript-eslint';

export default tseslint.config({
  rules: {
    '@typescript-eslint/no-explicit-any': 'error',        // Ban 'any'
    '@typescript-eslint/no-unsafe-assignment': 'error',   // No assigning 'any'
    '@typescript-eslint/no-unsafe-call': 'error',         // No calling 'any'
    '@typescript-eslint/no-unsafe-member-access': 'error',// No accessing 'any'
    '@typescript-eslint/no-unsafe-return': 'error',       // No returning 'any'
    '@typescript-eslint/explicit-function-return-type': ['warn', {
      allowExpressions: true,  // Allow inline arrow functions
    }],
    '@typescript-eslint/consistent-type-imports': 'error', // import type { X }
    '@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_' }],
    '@typescript-eslint/prefer-nullish-coalescing': 'error', // ?? over ||
    '@typescript-eslint/prefer-optional-chain': 'error',     // ?. over &&
  },
});
```

### PR Standards for TypeScript

**Interview-ready talking points:**
- "Zero `any` policy — CI fails if any `any` is introduced without a linked justification comment"
- "We use `noUncheckedIndexedAccess` — forces null checks on array access, prevents runtime errors"
- "All API responses are validated with Zod — TypeScript types are derived from schemas, not manually duplicated"
- "We run `tsc --noEmit` in CI as a separate step — type checking is enforced even if build passes"
- "Consistent type imports (`import type`) keep runtime bundles clean"

---

## Q126. How do you type custom hooks properly?

**Answer:**

### Hook with complex return type

```tsx
// ✅ Typed return with named properties (better than tuple for complex hooks)
interface UsePaginationReturn {
  page: number;
  pageSize: number;
  totalPages: number;
  hasNext: boolean;
  hasPrev: boolean;
  goToPage: (page: number) => void;
  nextPage: () => void;
  prevPage: () => void;
  setPageSize: (size: number) => void;
}

function usePagination(totalItems: number, initialPageSize = 10): UsePaginationReturn {
  const [page, setPage] = useState(1);
  const [pageSize, setPageSize] = useState(initialPageSize);

  const totalPages = Math.ceil(totalItems / pageSize);

  return {
    page,
    pageSize,
    totalPages,
    hasNext: page < totalPages,
    hasPrev: page > 1,
    goToPage: (p) => setPage(Math.max(1, Math.min(p, totalPages))),
    nextPage: () => setPage(p => Math.min(p + 1, totalPages)),
    prevPage: () => setPage(p => Math.max(p - 1, 1)),
    setPageSize: (size) => { setPageSize(size); setPage(1); },
  };
}
```

### Hook with generics

```tsx
interface UseAsyncReturn<T> {
  data: T | null;
  error: Error | null;
  isLoading: boolean;
  execute: () => Promise<void>;
}

function useAsync<T>(asyncFn: () => Promise<T>, immediate = true): UseAsyncReturn<T> {
  const [data, setData] = useState<T | null>(null);
  const [error, setError] = useState<Error | null>(null);
  const [isLoading, setIsLoading] = useState(immediate);

  const execute = useCallback(async () => {
    setIsLoading(true);
    setError(null);
    try {
      const result = await asyncFn();
      setData(result);
    } catch (err) {
      setError(err instanceof Error ? err : new Error(String(err)));
    } finally {
      setIsLoading(false);
    }
  }, [asyncFn]);

  useEffect(() => {
    if (immediate) { execute(); }
  }, [execute, immediate]);

  return { data, error, isLoading, execute };
}

// Usage — T is inferred from asyncFn return type
const { data: accounts, isLoading } = useAsync(() => accountService.getAll());
// accounts is typed as Account[] | null  ✅
```

### Hook with overloaded signatures

```tsx
// useDebounce with different behaviors based on input
function useDebounce<T>(value: T, delay: number): T;
function useDebounce<T>(value: T, delay: number, immediate: true): { value: T; isPending: boolean };
function useDebounce<T>(value: T, delay: number, immediate?: boolean) {
  const [debouncedValue, setDebouncedValue] = useState(value);
  const [isPending, setIsPending] = useState(false);

  useEffect(() => {
    setIsPending(true);
    const timer = setTimeout(() => {
      setDebouncedValue(value);
      setIsPending(false);
    }, delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  if (immediate) {
    return { value: debouncedValue, isPending };
  }
  return debouncedValue;
}
```

---

## Q127. How do you handle type-safe context in React?

**Answer:**

```tsx
// ❌ BAD: Context with undefined default + non-null assertion
const UserContext = createContext<User>(undefined!); // Lies to TypeScript!

// ✅ GOOD: Proper null handling with custom hook
interface AuthContextValue {
  user: User | null;
  isAuthenticated: boolean;
  login: (credentials: LoginCredentials) => Promise<void>;
  logout: () => void;
  permissions: Permission[];
}

const AuthContext = createContext<AuthContextValue | undefined>(undefined);

// Custom hook with runtime check — guarantees non-null outside provider
export function useAuth(): AuthContextValue {
  const context = useContext(AuthContext);
  if (context === undefined) {
    throw new Error('useAuth must be used within an AuthProvider');
  }
  return context;
}

// Provider implementation
export function AuthProvider({ children }: { children: React.ReactNode }) {
  const [user, setUser] = useState<User | null>(null);

  const value: AuthContextValue = useMemo(() => ({
    user,
    isAuthenticated: user !== null,
    login: async (credentials) => {
      const user = await authService.login(credentials);
      setUser(user);
    },
    logout: () => {
      authService.logout();
      setUser(null);
    },
    permissions: user?.permissions ?? [],
  }), [user]);

  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}

// Usage — never need null checks!
function UserProfile() {
  const { user, logout } = useAuth();  // ✅ Guaranteed non-null by hook
  return <div>{user?.name} <button onClick={logout}>Logout</button></div>;
}
```

---

## Q128. How do you type higher-order components and render props in TypeScript?

**Answer:**

### HOC with TypeScript

```tsx
// withLoading HOC — adds loading state to any component
interface WithLoadingProps {
  isLoading: boolean;
}

function withLoading<P extends object>(
  WrappedComponent: React.ComponentType<P>
): React.ComponentType<P & WithLoadingProps> {
  return function WithLoadingComponent({ isLoading, ...props }: P & WithLoadingProps) {
    if (isLoading) return <Spinner />;
    return <WrappedComponent {...(props as P)} />;
  };
}

// Usage
const AccountListWithLoading = withLoading(AccountList);
<AccountListWithLoading isLoading={loading} accounts={accounts} />;

// withPermission HOC — restricts access based on role
function withPermission<P extends object>(
  WrappedComponent: React.ComponentType<P>,
  requiredPermission: Permission
) {
  return function PermissionGuard(props: P) {
    const { permissions } = useAuth();
    if (!permissions.includes(requiredPermission)) {
      return <AccessDenied />;
    }
    return <WrappedComponent {...props} />;
  };
}

const AdminPanel = withPermission(AdminDashboard, 'admin:read');
```

### Render Props with TypeScript

```tsx
interface RenderProps<T> {
  data: T;
  isLoading: boolean;
  error: Error | null;
  refetch: () => void;
}

interface DataFetcherProps<T> {
  url: string;
  schema: z.ZodSchema<T>;
  children: (props: RenderProps<T | null>) => React.ReactNode;
}

function DataFetcher<T>({ url, schema, children }: DataFetcherProps<T>) {
  const [data, setData] = useState<T | null>(null);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  const fetch = useCallback(async () => {
    setIsLoading(true);
    try {
      const raw = await apiClient.get<unknown>(url);
      setData(schema.parse(raw));
    } catch (err) {
      setError(err instanceof Error ? err : new Error(String(err)));
    } finally {
      setIsLoading(false);
    }
  }, [url, schema]);

  useEffect(() => { fetch(); }, [fetch]);

  return <>{children({ data, isLoading, error, refetch: fetch })}</>;
}

// Usage
<DataFetcher url="/accounts" schema={AccountListSchema}>
  {({ data, isLoading, error }) => {
    if (isLoading) return <Spinner />;
    if (error) return <ErrorMessage error={error} />;
    return <AccountList accounts={data!} />;
  }}
</DataFetcher>
```

---

## Q129. How do you handle `as const` and template literal types?

**Answer:**

### `as const` for immutable constants

```tsx
// Without as const — types are widened
const ROUTES = {
  home: '/',
  accounts: '/accounts',
  transfer: '/transfer',
};
// Type: { home: string; accounts: string; transfer: string }
// routes.home could be reassigned to anything!

// With as const — types are narrowed and readonly
const ROUTES = {
  home: '/',
  accounts: '/accounts',
  transfer: '/transfer',
} as const;
// Type: { readonly home: '/'; readonly accounts: '/accounts'; readonly transfer: '/transfer' }

// Extract union of values
type Route = typeof ROUTES[keyof typeof ROUTES];
// '/' | '/accounts' | '/transfer'

// Event names
const EVENTS = ['click', 'hover', 'focus', 'blur'] as const;
type EventName = typeof EVENTS[number]; // 'click' | 'hover' | 'focus' | 'blur'
```

### Template literal types

```tsx
// CSS spacing utilities
type Size = 'sm' | 'md' | 'lg' | 'xl';
type Direction = 'top' | 'right' | 'bottom' | 'left';
type SpacingProp = `margin-${Direction}` | `padding-${Direction}`;
// "margin-top" | "margin-right" | ... | "padding-left"

// Event handler naming
type EventType = 'click' | 'change' | 'submit' | 'focus';
type HandlerName = `on${Capitalize<EventType>}`;
// "onClick" | "onChange" | "onSubmit" | "onFocus"

// Design token keys
type ColorScale = 10 | 20 | 30 | 40 | 50 | 60 | 70 | 80 | 90;
type ColorName = 'brand' | 'neutral' | 'danger' | 'success';
type TokenKey = `color-${ColorName}-${ColorScale}`;
// "color-brand-10" | "color-brand-20" | ... | "color-success-90"

// API endpoint builder
type HttpMethod = 'GET' | 'POST' | 'PUT' | 'DELETE';
type ApiVersion = 'v1' | 'v2';
type Endpoint = `/api/${ApiVersion}/${string}`;

function apiCall(method: HttpMethod, endpoint: Endpoint): Promise<unknown> {
  return fetch(endpoint, { method });
}
apiCall('GET', '/api/v1/accounts');    // ✅
apiCall('GET', '/random/endpoint');     // ❌ TypeScript error
```

---

## Q130. What are mapped types and how are they useful in component libraries?

**Answer:**

```tsx
// Basic mapped type — transform all properties
type Readonly<T> = { readonly [K in keyof T]: T[K] };
type Optional<T> = { [K in keyof T]?: T[K] };
type Nullable<T> = { [K in keyof T]: T[K] | null };

// Real-world: Form field state from value type
interface FormValues {
  name: string;
  email: string;
  age: number;
}

// Generate error state for each field
type FormErrors<T> = { [K in keyof T]?: string };
// { name?: string; email?: string; age?: string }

// Generate touched state for each field
type FormTouched<T> = { [K in keyof T]?: boolean };
// { name?: boolean; email?: boolean; age?: boolean }

// Generate field config from values
type FieldConfig<T> = {
  [K in keyof T]: {
    label: string;
    placeholder?: string;
    validate?: (value: T[K]) => string | undefined;
    transform?: (value: string) => T[K];
  };
};

const formConfig: FieldConfig<FormValues> = {
  name: { label: 'Full Name', validate: (v) => v.length < 2 ? 'Too short' : undefined },
  email: { label: 'Email', validate: (v) => !v.includes('@') ? 'Invalid' : undefined },
  age: { label: 'Age', transform: (v) => parseInt(v, 10) },
};

// Component theme tokens
type ComponentTokens<T extends string> = {
  [K in T as `${K}Color`]: string;
} & {
  [K in T as `${K}Size`]: number;
};

type ButtonTokens = ComponentTokens<'background' | 'border' | 'text'>;
// { backgroundColor: string; borderColor: string; textColor: string;
//   backgroundSize: number; borderSize: number; textSize: number; }
```

---

## Q131. How do you handle type narrowing and type guards?

**Answer:**

```tsx
// Type guard function
interface ApiSuccess<T> { status: 'success'; data: T; }
interface ApiError { status: 'error'; message: string; code: number; }
type ApiResponse<T> = ApiSuccess<T> | ApiError;

// Custom type guard
function isApiError<T>(response: ApiResponse<T>): response is ApiError {
  return response.status === 'error';
}

// Usage
async function fetchData<T>(url: string): Promise<T> {
  const response = await apiClient.get<ApiResponse<T>>(url);
  if (isApiError(response)) {
    throw new Error(`API Error ${response.code}: ${response.message}`);
  }
  return response.data;  // TypeScript knows this is ApiSuccess<T>
}

// Exhaustive checking with never
type NotificationType = 'info' | 'warning' | 'error' | 'success';

function getNotificationIcon(type: NotificationType): React.ReactNode {
  switch (type) {
    case 'info': return <InfoIcon />;
    case 'warning': return <WarningIcon />;
    case 'error': return <ErrorIcon />;
    case 'success': return <SuccessIcon />;
    default:
      // If someone adds a new type but forgets to handle it here,
      // TypeScript will error because `type` is not assignable to `never`
      const _exhaustive: never = type;
      return _exhaustive;
  }
}

// Narrowing with `in` operator
interface TextInput { type: 'text'; value: string; maxLength?: number; }
interface NumberInput { type: 'number'; value: number; min?: number; max?: number; }
interface SelectInput { type: 'select'; value: string; options: string[]; }
type FormField = TextInput | NumberInput | SelectInput;

function renderField(field: FormField) {
  if ('options' in field) {
    // TypeScript knows: field is SelectInput
    return <select>{field.options.map(o => <option key={o}>{o}</option>)}</select>;
  }
  if (field.type === 'number') {
    // TypeScript knows: field is NumberInput
    return <input type="number" value={field.value} min={field.min} max={field.max} />;
  }
  // TypeScript knows: field is TextInput
  return <input type="text" value={field.value} maxLength={field.maxLength} />;
}
```

---

## Q132. How do you handle module augmentation and declaration merging?

**Answer:**

This is critical for extending third-party library types (like Fluent UI themes).

```tsx
// Extending Fluent UI theme with custom tokens
// types/fluent-ui.d.ts
import '@fluentui/react-components';

declare module '@fluentui/react-components' {
  interface Theme {
    customTokens: {
      headerHeight: string;
      sidebarWidth: string;
      brandGradient: string;
    };
  }
}

// Now you can use custom tokens with full type safety
const theme = createLightTheme(brandRamp);
theme.customTokens.headerHeight = '64px';  // ✅ TypeScript knows this exists

// Extending Window for global variables
declare global {
  interface Window {
    __APP_CONFIG__: {
      apiUrl: string;
      featureFlags: Record<string, boolean>;
    };
  }
}

// Extending route params
// types/routes.d.ts
declare module 'react-router-dom' {
  interface RouteParams {
    accountId: string;
    transactionId: string;
  }
}

// Extending environment variables
// env.d.ts
interface ImportMetaEnv {
  readonly VITE_API_BASE_URL: string;
  readonly VITE_AUTH_DOMAIN: string;
  readonly VITE_FEATURE_NEW_DASHBOARD: string;
}
```

---

## Q133. What's the difference between `interface` and `type` — when do you use each?

**Answer:**

| Feature | `interface` | `type` |
|---------|-------------|--------|
| Extend/inherit | `extends` keyword | Intersection `&` |
| Declaration merging | ✅ Yes | ❌ No |
| Computed properties | ❌ No | ✅ Yes (mapped types) |
| Union types | ❌ No | ✅ Yes |
| Tuple types | ❌ No | ✅ Yes |
| Primitive aliases | ❌ No | ✅ Yes |

### When to use `interface`:
- **Component props** (can be extended by consumers)
- **Public API contracts** (supports declaration merging for library augmentation)
- **Object shapes** that might be extended

### When to use `type`:
- **Union types** (discriminated unions for component variants)
- **Utility type operations** (Pick, Omit, mapped types)
- **Tuples and primitives**
- **Complex derived types**

```tsx
// interface — component props (extensible)
interface ButtonProps {
  label: string;
  onClick: () => void;
}

interface IconButtonProps extends ButtonProps {
  icon: React.ReactNode;
}

// type — union (cannot be done with interface)
type ButtonVariant = 'primary' | 'secondary' | 'danger';
type ButtonSize = 'sm' | 'md' | 'lg';

// type — complex derivation
type ButtonStyleProps = Pick<ButtonProps, 'label'> & {
  variant: ButtonVariant;
  size: ButtonSize;
};

// Team convention (my recommendation):
// - Use `interface` for all props and public API shapes
// - Use `type` for unions, tuples, and derived/computed types
// - Be consistent — pick one convention and enforce via ESLint
```

---

## Quick Reference: TypeScript Patterns You MUST Know for This Role

| Pattern | Use Case |
|---------|----------|
| Generics `<T>` | Reusable components (List, Table, Form) |
| Discriminated Unions | Component variants (Button, Dialog, Notification) |
| `as const` | Route maps, event constants, config objects |
| Template Literal Types | Design token keys, CSS utilities |
| Mapped Types | Form state derivation, theme tokens |
| Type Guards | API response handling, narrowing unions |
| Module Augmentation | Extending Fluent UI theme, env vars |
| Zod + infer | API contracts — single source of truth |
| Utility Types | Partial, Pick, Omit for component API design |
| `extends` constraint | Generic bounds (`<T extends { id: string }>`) |

---

**Interview line**: "In our codebase, TypeScript isn't just for autocompletion — it's a design tool. Discriminated unions prevent impossible states, Zod schemas are the single source of truth for API contracts, and strict mode catches bugs that tests would miss."


