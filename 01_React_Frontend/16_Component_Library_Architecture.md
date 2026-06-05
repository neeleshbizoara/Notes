# Section 16: Component Library Architecture (10 Questions & Answers)

---

## Q146. How do you structure a shared component library for a large team?

**Answer:**

### Monorepo Structure

```
packages/
├── design-system/                    ← Core library
│   ├── package.json
│   ├── src/
│   │   ├── components/
│   │   │   ├── Button/
│   │   │   │   ├── Button.tsx        ← Implementation
│   │   │   │   ├── Button.types.ts   ← Public types
│   │   │   │   ├── Button.test.tsx   ← Unit tests
│   │   │   │   ├── Button.stories.tsx← Storybook
│   │   │   │   ├── Button.module.css ← Styles
│   │   │   │   └── index.ts         ← Barrel export
│   │   │   ├── Input/
│   │   │   ├── Dialog/
│   │   │   └── DataGrid/
│   │   ├── hooks/
│   │   │   ├── useClickOutside.ts
│   │   │   ├── useFocusTrap.ts
│   │   │   └── useMediaQuery.ts
│   │   ├── tokens/
│   │   │   ├── colors.ts
│   │   │   ├── spacing.ts
│   │   │   ├── typography.ts
│   │   │   └── breakpoints.ts
│   │   ├── utils/
│   │   │   ├── mergeRefs.ts
│   │   │   ├── cn.ts                ← className merger utility
│   │   │   └── polymorphic.ts       ← Polymorphic component helper
│   │   └── index.ts                 ← Public API barrel export
│   ├── tsconfig.json
│   └── rollup.config.ts             ← Build config (ESM + CJS)
│
├── design-system-docs/               ← Storybook + docs site
│   ├── .storybook/
│   └── stories/
│
├── eslint-config-ds/                 ← Shared ESLint rules
│   └── index.js
│
└── apps/
    ├── XXXX-portal/                  ← Main application
    └── admin-dashboard/              ← Another consumer
```

### Component Tiers

```
Tier 1: PRIMITIVES (atomic, no business logic)
────────────────────────────────────────────────
Button, Input, Checkbox, Radio, Select, Label,
Text, Icon, Avatar, Badge, Spinner, Divider

Tier 2: PATTERNS (composed primitives, generic UX patterns)
────────────────────────────────────────────────
FormField (Label + Input + Error), SearchInput,
Pagination, Tabs, Breadcrumb, Modal, Toast,
Dropdown, DatePicker, Popover

Tier 3: FEATURES (domain-aware, composed patterns)
────────────────────────────────────────────────
DataTable (sort + filter + paginate), FileUpload,
RichTextEditor, MultiSelect, TransferList

Tier 4: LAYOUTS (page-level composition)
────────────────────────────────────────────────
PageLayout, SidebarLayout, DashboardGrid,
ErrorPage, EmptyState
```

### Public API Surface Rules

```tsx
// ✅ EXPORT from package (public API)
export { Button } from './components/Button';
export type { ButtonProps } from './components/Button';
export { Input } from './components/Input';
export { dsTokens } from './tokens';
export { useClickOutside } from './hooks';

// ❌ NEVER export (internal implementation details)
// - Internal state management hooks
// - Implementation-specific utilities
// - Fluent/Motif direct imports
// - CSS class names
// - Internal component variants
```

---

## Q147. How do you design component APIs for maximum reusability?

**Answer:**

### Principle 1: Composition Over Configuration

```tsx
// ❌ BAD: Configuration-heavy (100 props, impossible to maintain)
<DataTable
  data={users}
  columns={columns}
  sortable
  filterable
  paginated
  pageSize={20}
  selectable
  onSelect={handleSelect}
  expandable
  renderExpanded={(row) => <Detail data={row} />}
  stickyHeader
  virtualScroll
  exportable
  exportFormats={['csv', 'pdf']}
  toolbar
  toolbarActions={[...]}
  // ... 50 more props
/>

// ✅ GOOD: Composition-based (each concern is a separate component)
<DataTable data={users}>
  <DataTable.Toolbar>
    <DataTable.Search />
    <DataTable.Export formats={['csv', 'pdf']} />
  </DataTable.Toolbar>
  <DataTable.Header sticky>
    <DataTable.Column key="name" sortable>Name</DataTable.Column>
    <DataTable.Column key="email" filterable>Email</DataTable.Column>
  </DataTable.Header>
  <DataTable.Body virtualScroll>
    {(row) => (
      <DataTable.Row key={row.id} expandable>
        <DataTable.Cell>{row.name}</DataTable.Cell>
        <DataTable.Cell>{row.email}</DataTable.Cell>
        <DataTable.ExpandedContent>
          <UserDetail user={row} />
        </DataTable.ExpandedContent>
      </DataTable.Row>
    )}
  </DataTable.Body>
  <DataTable.Pagination pageSize={20} />
</DataTable>
```

### Principle 2: Controlled + Uncontrolled Support

```tsx
interface InputProps {
  // Uncontrolled (internal state)
  defaultValue?: string;
  // Controlled (external state)
  value?: string;
  onChange?: (value: string) => void;
  // Common
  placeholder?: string;
  disabled?: boolean;
}

function Input({ defaultValue, value, onChange, ...rest }: InputProps) {
  // Support both controlled and uncontrolled usage
  const [internalValue, setInternalValue] = useState(defaultValue ?? '');
  const isControlled = value !== undefined;
  const currentValue = isControlled ? value : internalValue;

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const newValue = e.target.value;
    if (!isControlled) setInternalValue(newValue);
    onChange?.(newValue);
  };

  return <input value={currentValue} onChange={handleChange} {...rest} />;
}

// Uncontrolled (simple cases)
<Input defaultValue="hello" onChange={(v) => console.log(v)} />

// Controlled (form libraries, complex state)
<Input value={formState.name} onChange={(v) => setFormState({ ...s, name: v })} />
```

### Principle 3: Polymorphic "as" Prop

```tsx
// Component can render as any HTML element or custom component
interface BoxProps<E extends React.ElementType = 'div'> {
  as?: E;
  children?: React.ReactNode;
  padding?: 'sm' | 'md' | 'lg';
}

type PolymorphicBoxProps<E extends React.ElementType = 'div'> =
  BoxProps<E> & Omit<React.ComponentPropsWithoutRef<E>, keyof BoxProps>;

function Box<E extends React.ElementType = 'div'>({
  as, children, padding, ...rest
}: PolymorphicBoxProps<E>) {
  const Component = as || 'div';
  return (
    <Component className={`box box-pad-${padding}`} {...rest}>
      {children}
    </Component>
  );
}

// Usage
<Box padding="md">Regular div</Box>
<Box as="section" padding="lg">Section element</Box>
<Box as="a" href="/home" padding="sm">Link element</Box>
<Box as={Link} to="/home">React Router Link</Box>
```

### Principle 4: Forwarded Refs

```tsx
// Always forward refs — consumers may need to measure, focus, or scroll
interface InputProps extends React.InputHTMLAttributes<HTMLInputElement> {
  label: string;
  error?: string;
}

const Input = React.forwardRef<HTMLInputElement, InputProps>(
  ({ label, error, id, ...inputProps }, ref) => {
    const inputId = id || useId();
    const errorId = `${inputId}-error`;

    return (
      <div className="form-field">
        <label htmlFor={inputId}>{label}</label>
        <input
          ref={ref}  // ← Consumers can access the DOM element
          id={inputId}
          aria-invalid={!!error}
          aria-describedby={error ? errorId : undefined}
          {...inputProps}
        />
        {error && <span id={errorId} role="alert">{error}</span>}
      </div>
    );
  }
);

Input.displayName = 'Input';

// Consumer can now:
const inputRef = useRef<HTMLInputElement>(null);
<Input ref={inputRef} label="Name" />
// inputRef.current?.focus();
// inputRef.current?.scrollIntoView();
```

---

## Q148. How do you handle versioning and breaking changes in a component library?

**Answer:**

### Semantic Versioning Strategy

```
MAJOR (1.0.0 → 2.0.0): Breaking prop changes, removed components
MINOR (1.0.0 → 1.1.0): New components, new optional props, new variants
PATCH (1.0.0 → 1.0.1): Bug fixes, visual fixes, a11y fixes
```

### Changesets for Automated Versioning

```bash
# Developer workflow
npx changeset add

# Interactive prompt:
# ? Which packages would you like to include?
#   ◉ @XXXX/design-system
# ? Which packages should have a major bump?
#   (none)
# ? Which packages should have a minor bump?
#   ◉ @XXXX/design-system
# ? Summary: Added new 'ghost' variant to Button component
```

### Deprecation Strategy (NEVER remove without warning)

```tsx
/**
 * @deprecated Use `variant="outline"` instead. Will be removed in v3.0.
 */
interface ButtonProps {
  /** @deprecated Use `variant` prop instead */
  appearance?: 'primary' | 'secondary';
  variant?: 'primary' | 'secondary' | 'outline' | 'danger' | 'ghost';
}

function Button({ appearance, variant, ...rest }: ButtonProps) {
  // Support both during transition
  const resolvedVariant = variant ?? appearance ?? 'primary';

  if (appearance !== undefined) {
    console.warn(
      '[DesignSystem] Button: `appearance` prop is deprecated. Use `variant` instead. ' +
      'See migration guide: https://docs.internal/ds/migration/button-v3'
    );
  }

  return <InternalButton variant={resolvedVariant} {...rest} />;
}
```

### Codemods for Breaking Changes

```tsx
// When you MUST make breaking changes, provide an automated codemod
// codemods/button-appearance-to-variant.ts
import { API, FileInfo } from 'jscodeshift';

export default function transform(file: FileInfo, api: API) {
  const j = api.jscodeshift;
  const root = j(file.source);

  // Find all <Button appearance="..." /> and rename to variant
  root.findJSXElements('Button')
    .find(j.JSXAttribute, { name: { name: 'appearance' } })
    .forEach(path => {
      path.node.name.name = 'variant';
    });

  return root.toSource();
}

// Developers run:
// npx jscodeshift --transform codemods/button-appearance-to-variant.ts src/
```

---

## Q149. How do you ensure design system adoption across teams?

**Answer:**

### 1. ESLint Rules (Automated Enforcement)

```js
// eslint-plugin-ds/rules/no-direct-fluent-import.js
module.exports = {
  create(context) {
    return {
      ImportDeclaration(node) {
        if (node.source.value.startsWith('@fluentui/')) {
          context.report({
            node,
            message:
              'Import from @XXXX/design-system instead of @fluentui directly. ' +
              'See: https://docs.internal/ds/usage',
            fix(fixer) {
              // Auto-fix: Replace @fluentui/react-components → @XXXX/design-system
              return fixer.replaceText(node.source, "'@XXXX/design-system'");
            },
          });
        }
      },
    };
  },
};

// .eslintrc
{
  "rules": {
    "ds/no-direct-fluent-import": "error",
    "ds/no-inline-styles-for-tokens": "warn",
    "ds/require-design-token": "warn"
  }
}
```

### 2. Storybook as Living Documentation

```tsx
// Button.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import { Button } from './Button';

const meta: Meta<typeof Button> = {
  title: 'Components/Button',
  component: Button,
  tags: ['autodocs'],  // Auto-generate docs from types
  argTypes: {
    variant: {
      control: 'select',
      options: ['primary', 'secondary', 'outline', 'danger', 'ghost'],
      description: 'Visual style variant',
    },
  },
};
export default meta;

type Story = StoryObj<typeof Button>;

export const Primary: Story = {
  args: { variant: 'primary', children: 'Primary Button' },
};

export const AllVariants: Story = {
  render: () => (
    <div style={{ display: 'flex', gap: 8 }}>
      <Button variant="primary">Primary</Button>
      <Button variant="secondary">Secondary</Button>
      <Button variant="outline">Outline</Button>
      <Button variant="danger">Danger</Button>
      <Button variant="ghost">Ghost</Button>
    </div>
  ),
};

// Accessibility story
export const AccessibilityChecked: Story = {
  args: { variant: 'primary', children: 'Accessible Button' },
  play: async ({ canvasElement }) => {
    const results = await axe(canvasElement);
    expect(results.violations).toHaveLength(0);
  },
};
```

### 3. Migration Helper Scripts

```bash
# Scan codebase for non-DS component usage
npx ds-audit --report

# Output:
# ┌─────────────────────────────────────────────────────┐
# │ Design System Adoption Report                        │
# ├─────────────────────────────────────────────────────┤
# │ Total UI components found: 847                       │
# │ Using DS components: 712 (84%)                       │
# │ Direct Fluent imports: 89 (10.5%)                    │
# │ Custom/inline implementations: 46 (5.5%)            │
# │                                                      │
# │ Top offenders:                                       │
# │ - features/payments/TransferForm.tsx (8 direct)      │
# │ - features/reports/ReportGrid.tsx (5 direct)         │
# │ - features/admin/UserTable.tsx (4 direct)            │
# └─────────────────────────────────────────────────────┘
```

---

## Q150. How do you handle component state management inside a library?

**Answer:**

### Rule: Components should be state-agnostic (no Redux/Zustand inside)

```tsx
// ❌ BAD: Component tied to a specific state management library
import { useSelector } from 'react-redux';

function UserMenu() {
  const user = useSelector(state => state.auth.user);  // ❌ Library depends on Redux!
  return <Menu items={[{ label: user.name }]} />;
}

// ✅ GOOD: Component receives data via props (consumer controls state)
interface UserMenuProps {
  userName: string;
  avatarUrl?: string;
  onLogout: () => void;
  menuItems?: MenuItem[];
}

function UserMenu({ userName, avatarUrl, onLogout, menuItems }: UserMenuProps) {
  return (
    <Menu trigger={<Avatar name={userName} image={avatarUrl} />}>
      {menuItems?.map(item => <MenuItem key={item.id} {...item} />)}
      <MenuDivider />
      <MenuItem onClick={onLogout}>Logout</MenuItem>
    </Menu>
  );
}
```

### Internal State: Use Reducers for Complex Components

```tsx
// DataGrid internal state — complex but self-contained
interface DataGridState<T> {
  sortColumn: keyof T | null;
  sortDirection: 'asc' | 'desc';
  currentPage: number;
  pageSize: number;
  selectedRows: Set<string>;
  expandedRows: Set<string>;
  filters: Record<string, string>;
}

type DataGridAction<T> =
  | { type: 'SORT'; column: keyof T }
  | { type: 'PAGE'; page: number }
  | { type: 'SELECT_ROW'; id: string }
  | { type: 'EXPAND_ROW'; id: string }
  | { type: 'FILTER'; column: string; value: string }
  | { type: 'RESET' };

function dataGridReducer<T>(state: DataGridState<T>, action: DataGridAction<T>): DataGridState<T> {
  switch (action.type) {
    case 'SORT':
      return {
        ...state,
        sortColumn: action.column,
        sortDirection: state.sortColumn === action.column && state.sortDirection === 'asc' ? 'desc' : 'asc',
        currentPage: 1,
      };
    case 'PAGE':
      return { ...state, currentPage: action.page };
    case 'SELECT_ROW': {
      const next = new Set(state.selectedRows);
      next.has(action.id) ? next.delete(action.id) : next.add(action.id);
      return { ...state, selectedRows: next };
    }
    // ... other cases
    default:
      return state;
  }
}
```

### Context for Compound Components (Internal Only)

```tsx
// Tabs component — internal context for child coordination
interface TabsContextValue {
  activeTab: string;
  setActiveTab: (id: string) => void;
  orientation: 'horizontal' | 'vertical';
}

const TabsContext = createContext<TabsContextValue | undefined>(undefined);

function Tabs({ defaultTab, orientation = 'horizontal', children, onChange }: TabsProps) {
  const [activeTab, setActiveTab] = useState(defaultTab);

  const handleChange = (id: string) => {
    setActiveTab(id);
    onChange?.(id);
  };

  return (
    <TabsContext.Provider value={{ activeTab, setActiveTab: handleChange, orientation }}>
      <div role="tablist" aria-orientation={orientation}>
        {children}
      </div>
    </TabsContext.Provider>
  );
}

function Tab({ id, children }: { id: string; children: React.ReactNode }) {
  const ctx = useContext(TabsContext);
  if (!ctx) throw new Error('Tab must be used within Tabs');

  return (
    <button
      role="tab"
      aria-selected={ctx.activeTab === id}
      onClick={() => ctx.setActiveTab(id)}
    >
      {children}
    </button>
  );
}

function TabPanel({ id, children }: { id: string; children: React.ReactNode }) {
  const ctx = useContext(TabsContext);
  if (!ctx) throw new Error('TabPanel must be used within Tabs');
  if (ctx.activeTab !== id) return null;
  return <div role="tabpanel">{children}</div>;
}

// Public API
Tabs.Tab = Tab;
Tabs.Panel = TabPanel;

// Usage
<Tabs defaultTab="accounts" onChange={(tab) => trackAnalytics(tab)}>
  <Tabs.Tab id="accounts">Accounts</Tabs.Tab>
  <Tabs.Tab id="payments">Payments</Tabs.Tab>
  <Tabs.Panel id="accounts"><AccountList /></Tabs.Panel>
  <Tabs.Panel id="payments"><PaymentList /></Tabs.Panel>
</Tabs>
```

---

## Q151. How do you handle theming and dark mode in a component library?

**Answer:**

### Token-Based Theming (CSS Custom Properties)

```tsx
// tokens/themes.ts
export const lightTheme = {
  '--ds-color-background': '#ffffff',
  '--ds-color-foreground': '#1a1a1a',
  '--ds-color-brand': '#0078D4',
  '--ds-color-brand-hover': '#006CBE',
  '--ds-color-surface': '#f5f5f5',
  '--ds-color-border': '#e0e0e0',
  '--ds-color-danger': '#d32f2f',
  '--ds-color-success': '#2e7d32',
  '--ds-shadow-sm': '0 1px 2px rgba(0,0,0,0.08)',
  '--ds-shadow-md': '0 4px 8px rgba(0,0,0,0.12)',
};

export const darkTheme = {
  '--ds-color-background': '#1a1a1a',
  '--ds-color-foreground': '#f5f5f5',
  '--ds-color-brand': '#4dabf5',
  '--ds-color-brand-hover': '#6bbefc',
  '--ds-color-surface': '#2d2d2d',
  '--ds-color-border': '#404040',
  '--ds-color-danger': '#f44336',
  '--ds-color-success': '#66bb6a',
  '--ds-shadow-sm': '0 1px 2px rgba(0,0,0,0.3)',
  '--ds-shadow-md': '0 4px 8px rgba(0,0,0,0.4)',
};

// ThemeProvider
interface ThemeProviderProps {
  theme: 'light' | 'dark' | 'system';
  children: React.ReactNode;
}

export function ThemeProvider({ theme, children }: ThemeProviderProps) {
  const resolvedTheme = useResolvedTheme(theme); // Handles 'system' preference
  const tokens = resolvedTheme === 'dark' ? darkTheme : lightTheme;

  return (
    <div style={tokens as React.CSSProperties} data-theme={resolvedTheme}>
      {children}
    </div>
  );
}

function useResolvedTheme(theme: 'light' | 'dark' | 'system') {
  const [systemPreference, setSystemPreference] = useState<'light' | 'dark'>('light');

  useEffect(() => {
    const mq = window.matchMedia('(prefers-color-scheme: dark)');
    setSystemPreference(mq.matches ? 'dark' : 'light');
    const handler = (e: MediaQueryListEvent) => setSystemPreference(e.matches ? 'dark' : 'light');
    mq.addEventListener('change', handler);
    return () => mq.removeEventListener('change', handler);
  }, []);

  return theme === 'system' ? systemPreference : theme;
}
```

### Components Use Tokens (Never Hard-code Colors)

```css
/* Button.module.css */
.button {
  background: var(--ds-color-brand);
  color: white;
  border: none;
  border-radius: var(--ds-border-radius-md);
  padding: var(--ds-spacing-sm) var(--ds-spacing-md);
  font-size: var(--ds-font-size-base);
  box-shadow: var(--ds-shadow-sm);
  transition: background 0.2s;
}

.button:hover {
  background: var(--ds-color-brand-hover);
}

.button[data-variant="secondary"] {
  background: var(--ds-color-surface);
  color: var(--ds-color-foreground);
  border: 1px solid var(--ds-color-border);
}
```

---

## Q152. How do you optimize bundle size for a component library?

**Answer:**

### Tree-Shaking with Proper Exports

```json
// package.json
{
  "name": "@XXXX/design-system",
  "sideEffects": false,  // ← Critical for tree-shaking
  "main": "dist/cjs/index.js",
  "module": "dist/esm/index.js",
  "types": "dist/types/index.d.ts",
  "exports": {
    ".": {
      "import": "./dist/esm/index.js",
      "require": "./dist/cjs/index.js",
      "types": "./dist/types/index.d.ts"
    },
    "./Button": {
      "import": "./dist/esm/components/Button/index.js",
      "require": "./dist/cjs/components/Button/index.js"
    }
  }
}
```

### Per-Component Imports (Escape Hatch for Large Libraries)

```tsx
// If barrel export tree-shaking isn't sufficient:
// Instead of: import { Button, Input, Dialog } from '@XXXX/design-system';
// Allow: import { Button } from '@XXXX/design-system/Button';

// This guarantees ONLY Button code is bundled
```

### Bundle Size Monitoring in CI

```yaml
# .github/workflows/bundle-check.yml
- name: Check bundle size
  run: npx size-limit
  # size-limit config in package.json:
  # "size-limit": [
  #   { "path": "dist/esm/index.js", "limit": "50 KB" },
  #   { "path": "dist/esm/components/Button/index.js", "limit": "3 KB" },
  #   { "path": "dist/esm/components/DataGrid/index.js", "limit": "15 KB" }
  # ]
```

---

## Q153. How do you document components for developer adoption?

**Answer:**

### Storybook + MDX Documentation

```mdx
{/* Button.mdx */}
import { Meta, Story, Canvas, ArgsTable } from '@storybook/blocks';
import { Button } from './Button';

<Meta title="Components/Button" component={Button} />

# Button

Buttons trigger actions. Use the appropriate variant for the action's importance.

## When to Use

| Scenario | Variant |
|----------|---------|
| Primary action (Submit, Save) | `primary` |
| Secondary action (Cancel, Back) | `secondary` |
| Destructive action (Delete) | `danger` |
| Tertiary/subtle action | `ghost` |

## Do's and Don'ts

✅ **Do:** Use one primary button per section
✅ **Do:** Use verb labels ("Save", "Delete", not "Yes", "OK")
❌ **Don't:** Use more than 3 buttons in a row
❌ **Don't:** Disable buttons without explaining why

## Examples

<Canvas>
  <Story name="Primary">
    <Button variant="primary">Save Changes</Button>
  </Story>
</Canvas>

## Props

<ArgsTable of={Button} />

## Accessibility

- All buttons have `role="button"` (native `<button>` element)
- Icon-only buttons MUST have `aria-label`
- Loading state announces to screen readers via `aria-busy`
- Focus ring is visible in all themes (meets WCAG 2.4.7)
```

---

## Q154. What's the separation of concerns in a well-architected component?

**Answer:**

```
┌─────────────────────────────────────────────────────────────┐
│                    Component Architecture                     │
├──────────────┬──────────────┬──────────────┬────────────────┤
│   Types      │   Logic      │   UI         │   Styles       │
│  (.types.ts) │  (hooks)     │  (.tsx)      │  (.module.css) │
├──────────────┼──────────────┼──────────────┼────────────────┤
│ Props        │ State mgmt   │ JSX render   │ Visual tokens  │
│ Events       │ Side effects │ A11y attrs   │ Responsive     │
│ Variants     │ Derived data │ Composition  │ Animations     │
│ Constraints  │ Handlers     │ Refs         │ Dark mode      │
└──────────────┴──────────────┴──────────────┴────────────────┘
```

### Example: DatePicker Component

```
DatePicker/
├── DatePicker.types.ts      ← Props interface, types, constraints
├── useDatePicker.ts         ← All logic: date math, navigation, selection
├── DatePicker.tsx           ← Thin UI shell, composes Calendar + Input
├── Calendar.tsx             ← Sub-component: month grid
├── CalendarHeader.tsx       ← Sub-component: month/year navigation
├── DatePicker.module.css    ← Styles only
├── DatePicker.test.tsx      ← Tests against public API
├── DatePicker.stories.tsx   ← Visual documentation
└── index.ts                 ← Public exports
```

```tsx
// useDatePicker.ts — ALL logic lives here (testable without rendering)
interface UseDatePickerOptions {
  value?: Date;
  defaultValue?: Date;
  onChange?: (date: Date) => void;
  minDate?: Date;
  maxDate?: Date;
  disabledDates?: Date[];
}

interface UseDatePickerReturn {
  selectedDate: Date | null;
  viewDate: Date;
  isOpen: boolean;
  days: DayCell[];
  selectDate: (date: Date) => void;
  nextMonth: () => void;
  prevMonth: () => void;
  open: () => void;
  close: () => void;
  isDateDisabled: (date: Date) => boolean;
}

function useDatePicker(options: UseDatePickerOptions): UseDatePickerReturn {
  // All date logic, navigation, validation lives here
  // Zero rendering concerns
}

// DatePicker.tsx — THIN UI shell
function DatePicker(props: DatePickerProps) {
  const datePicker = useDatePicker(props);
  const inputRef = useRef<HTMLInputElement>(null);

  return (
    <div className={styles.wrapper}>
      <Input
        ref={inputRef}
        value={formatDate(datePicker.selectedDate)}
        onClick={datePicker.open}
        aria-expanded={datePicker.isOpen}
      />
      {datePicker.isOpen && (
        <Calendar
          days={datePicker.days}
          onSelect={datePicker.selectDate}
          onNextMonth={datePicker.nextMonth}
          onPrevMonth={datePicker.prevMonth}
        />
      )}
    </div>
  );
}
```

---

## Q155. How do you handle cross-cutting concerns in a component library?

**Answer:**

### Focus Management (shared utility)

```tsx
// hooks/useFocusTrap.ts
function useFocusTrap(active: boolean) {
  const containerRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (!active || !containerRef.current) return;

    const container = containerRef.current;
    const focusableElements = container.querySelectorAll(
      'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
    );
    const firstElement = focusableElements[0] as HTMLElement;
    const lastElement = focusableElements[focusableElements.length - 1] as HTMLElement;

    function handleKeyDown(e: KeyboardEvent) {
      if (e.key !== 'Tab') return;
      if (e.shiftKey && document.activeElement === firstElement) {
        e.preventDefault();
        lastElement.focus();
      } else if (!e.shiftKey && document.activeElement === lastElement) {
        e.preventDefault();
        firstElement.focus();
      }
    }

    firstElement?.focus();
    container.addEventListener('keydown', handleKeyDown);
    return () => container.removeEventListener('keydown', handleKeyDown);
  }, [active]);

  return containerRef;
}
```

### Error Boundary (library-level)

```tsx
// components/ErrorBoundary.tsx
interface ErrorBoundaryProps {
  fallback: React.ReactNode | ((error: Error) => React.ReactNode);
  onError?: (error: Error, errorInfo: React.ErrorInfo) => void;
  children: React.ReactNode;
}

class DSErrorBoundary extends React.Component<ErrorBoundaryProps, { error: Error | null }> {
  state = { error: null };

  static getDerivedStateFromError(error: Error) {
    return { error };
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    this.props.onError?.(error, errorInfo);
  }

  render() {
    if (this.state.error) {
      const { fallback } = this.props;
      return typeof fallback === 'function' ? fallback(this.state.error) : fallback;
    }
    return this.props.children;
  }
}
```

### Responsive Design Utilities

```tsx
// hooks/useBreakpoint.ts
const breakpoints = {
  sm: 640,
  md: 768,
  lg: 1024,
  xl: 1280,
} as const;

type Breakpoint = keyof typeof breakpoints;

function useBreakpoint(): Breakpoint {
  const [current, setCurrent] = useState<Breakpoint>('lg');

  useEffect(() => {
    function update() {
      const width = window.innerWidth;
      if (width < breakpoints.sm) setCurrent('sm');
      else if (width < breakpoints.md) setCurrent('md');
      else if (width < breakpoints.lg) setCurrent('lg');
      else setCurrent('xl');
    }
    update();
    window.addEventListener('resize', update);
    return () => window.removeEventListener('resize', update);
  }, []);

  return current;
}

// Responsive props pattern
interface ResponsiveProp<T> {
  base: T;
  sm?: T;
  md?: T;
  lg?: T;
  xl?: T;
}

// Usage in Grid component
<Grid columns={{ base: 1, md: 2, lg: 3 }} gap={{ base: 'sm', lg: 'md' }}>
  <Card />
  <Card />
  <Card />
</Grid>
```

---

## Key Interview Lines for Component Library Architecture

1. "I structure components in 4 tiers: Primitives, Patterns, Features, and Layouts — each with clear boundaries on what they can depend on."

2. "Every component supports both controlled and uncontrolled usage, forwards refs, and has a polymorphic `as` prop where appropriate."

3. "We use changesets for automated semver, deprecation warnings for a full major version before removal, and codemods for automated migration."

4. "Adoption is enforced via ESLint rules that prevent direct Fluent imports, plus a quarterly adoption report that tracks DS coverage."

5. "Components are state-agnostic — no Redux or Zustand inside the library. State management is the consumer's responsibility."

6. "All logic lives in a custom hook (testable without rendering), the component file is a thin UI shell, and styles use design tokens exclusively."

