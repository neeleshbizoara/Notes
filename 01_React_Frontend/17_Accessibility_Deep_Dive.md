# Section 17: Accessibility (A11y) Deep Dive (12 Questions & Answers)


---

## Q156. What is WCAG 2.1 and what does "Level AA" mean?

**Answer:**

**WCAG** = Web Content Accessibility Guidelines (by W3C)

### Three Conformance Levels:

| Level | Meaning | Examples |
|-------|---------|----------|
| **A** | Minimum accessibility | Alt text on images, keyboard accessible |
| **AA** | Standard for enterprise/government (YOUR TARGET) | Color contrast 4.5:1, focus visible, error identification |
| **AAA** | Enhanced — usually not required in full | Sign language for videos, contrast 7:1 |

### WCAG 2.1 — Four Principles (POUR):

```
P — Perceivable:   Can users perceive the content? (alt text, captions, contrast)
O — Operable:      Can users operate the UI? (keyboard, timing, navigation)
U — Understandable: Can users understand the content? (readable, predictable, errors)
R — Robust:        Does it work with assistive tech? (valid HTML, ARIA, compatibility)
```

### Key AA Requirements for React Applications:

| Requirement | WCAG Criterion | React Implementation |
|-------------|---------------|---------------------|
| Color contrast 4.5:1 (text) | 1.4.3 | Design tokens with verified contrast |
| Color contrast 3:1 (UI components) | 1.4.11 | Borders, icons, focus rings |
| Keyboard accessible | 2.1.1 | All interactive elements focusable |
| Focus visible | 2.4.7 | Visible focus indicator on all elements |
| No keyboard traps | 2.1.2 | Escape closes modals, menus |
| Error identification | 3.3.1 | Errors announced, linked to fields |
| Labels or instructions | 3.3.2 | All inputs have visible labels |
| Name, Role, Value | 4.1.2 | Correct ARIA roles and states |
| Reflow (responsive) | 1.4.10 | Content usable at 400% zoom |
| Non-text contrast | 1.4.11 | Icons/borders distinguishable |

**Interview line:** "We target WCAG 2.1 AA as our baseline. Every PR must pass automated axe-core checks, and we do manual screen reader testing before major releases."

---

## Q157. How do ARIA roles, states, and properties work?

**Answer:**

ARIA (**Accessible Rich Internet Applications**) provides semantics for dynamic content that HTML alone can't express.

### Three Categories:

```tsx
// 1. ROLES — What is this element? (noun)
<div role="alert">Payment failed</div>          // Announces immediately
<div role="dialog" aria-modal="true">...</div>   // Modal dialog
<div role="tablist">...</div>                     // Tab container
<nav role="navigation" aria-label="Main">...</nav> // Navigation landmark

// 2. STATES — What's the current condition? (adjective, changes dynamically)
<button aria-expanded="true">Menu</button>        // Expanded/collapsed
<input aria-invalid="true" />                     // Field has error
<div aria-busy="true">Loading...</div>            // Content loading
<button aria-pressed="true">Bold</button>         // Toggle button ON
<option aria-selected="true">Option 1</option>    // Selected item
<section aria-hidden="true">...</section>         // Hidden from AT

// 3. PROPERTIES — Relationships and descriptions (adjective, usually static)
<input aria-label="Search accounts" />            // Label (no visible label)
<input aria-labelledby="name-label" />            // Points to label element
<input aria-describedby="name-help name-error" /> // Additional description
<ul aria-owns="dropdown-list" />                  // Owns another element
<div aria-live="polite">3 results found</div>     // Live region
<input aria-required="true" />                    // Required field
```

### Rules of ARIA:

```
Rule 1: Don't use ARIA if native HTML works
         <button> is better than <div role="button">

Rule 2: Don't change native semantics
         ❌ <h2 role="button">  — confusing!
         ✅ <h2><button>Click</button></h2>

Rule 3: All interactive ARIA roles must be keyboard accessible
         role="button" → must handle Enter and Space key

Rule 4: Don't use role="presentation" or aria-hidden="true" on focusable elements

Rule 5: All interactive elements must have an accessible name
         Every button, link, input needs a label (visible or aria-label)
```

### Common ARIA Patterns in React:

```tsx
// Disclosure (Accordion)
function Accordion({ title, children }: AccordionProps) {
  const [isOpen, setIsOpen] = useState(false);
  const panelId = useId();

  return (
    <div>
      <button
        aria-expanded={isOpen}
        aria-controls={panelId}
        onClick={() => setIsOpen(!isOpen)}
      >
        {title}
      </button>
      <div
        id={panelId}
        role="region"
        aria-labelledby={/* button id */}
        hidden={!isOpen}
      >
        {children}
      </div>
    </div>
  );
}

// Combobox (Autocomplete)
function Autocomplete({ options, onSelect }: AutocompleteProps) {
  const [isOpen, setIsOpen] = useState(false);
  const [activeIndex, setActiveIndex] = useState(-1);
  const listId = useId();

  return (
    <div>
      <input
        role="combobox"
        aria-expanded={isOpen}
        aria-controls={listId}
        aria-activedescendant={activeIndex >= 0 ? `option-${activeIndex}` : undefined}
        aria-autocomplete="list"
      />
      <ul id={listId} role="listbox" aria-label="Suggestions">
        {options.map((opt, i) => (
          <li
            key={opt.id}
            id={`option-${i}`}
            role="option"
            aria-selected={i === activeIndex}
            onClick={() => onSelect(opt)}
          >
            {opt.label}
          </li>
        ))}
      </ul>
    </div>
  );
}
```

---

## Q158. How do you handle focus management in a React SPA?

**Answer:**

### Problem: SPAs Don't Announce Route Changes

In a traditional website, page navigation announces the new page title. In SPAs, route changes are silent — screen reader users don't know the page changed.

### Solution 1: Focus on Route Change

```tsx
// Layout.tsx — Move focus to main content on navigation
import { useLocation } from 'react-router-dom';

function Layout({ children }: { children: React.ReactNode }) {
  const location = useLocation();
  const mainRef = useRef<HTMLElement>(null);

  useEffect(() => {
    // Move focus to main content area on route change
    mainRef.current?.focus();
  }, [location.pathname]);

  return (
    <>
      <SkipLink />
      <Header />
      <main ref={mainRef} tabIndex={-1} aria-label="Main content">
        {children}
      </main>
      <Footer />
    </>
  );
}
```

### Solution 2: Live Region Announcements

```tsx
// RouteAnnouncer.tsx — Announce page title to screen readers
function RouteAnnouncer() {
  const location = useLocation();
  const [announcement, setAnnouncement] = useState('');

  useEffect(() => {
    // Get page title from route metadata or document.title
    const title = document.title || 'Page loaded';
    setAnnouncement(title);
  }, [location.pathname]);

  return (
    <div
      role="status"
      aria-live="polite"
      aria-atomic="true"
      className="sr-only"  // Visually hidden
    >
      {announcement}
    </div>
  );
}

// CSS for sr-only (visually hidden but accessible)
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}
```

### Solution 3: Skip Links

```tsx
function SkipLink() {
  return (
    <a
      href="#main-content"
      className="skip-link"  // Hidden until focused
    >
      Skip to main content
    </a>
  );
}

// CSS
.skip-link {
  position: absolute;
  top: -100%;
  left: 0;
  z-index: 1000;
  padding: 8px 16px;
  background: var(--ds-color-brand);
  color: white;
}

.skip-link:focus {
  top: 0;  /* Visible when focused via Tab */
}
```

### Focus Trapping in Modals

```tsx
function Dialog({ isOpen, onClose, title, children }: DialogProps) {
  const dialogRef = useRef<HTMLDivElement>(null);
  const previousFocusRef = useRef<HTMLElement | null>(null);

  useEffect(() => {
    if (isOpen) {
      // Save current focus
      previousFocusRef.current = document.activeElement as HTMLElement;
      // Move focus into dialog
      dialogRef.current?.focus();
    } else {
      // Restore focus when dialog closes
      previousFocusRef.current?.focus();
    }
  }, [isOpen]);

  // Trap focus inside dialog
  const handleKeyDown = (e: React.KeyboardEvent) => {
    if (e.key === 'Escape') {
      onClose();
      return;
    }
    if (e.key === 'Tab') {
      const focusable = dialogRef.current?.querySelectorAll(
        'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
      );
      if (!focusable?.length) return;

      const first = focusable[0] as HTMLElement;
      const last = focusable[focusable.length - 1] as HTMLElement;

      if (e.shiftKey && document.activeElement === first) {
        e.preventDefault();
        last.focus();
      } else if (!e.shiftKey && document.activeElement === last) {
        e.preventDefault();
        first.focus();
      }
    }
  };

  if (!isOpen) return null;

  return (
    <>
      {/* Backdrop */}
      <div className="dialog-backdrop" aria-hidden="true" onClick={onClose} />
      {/* Dialog */}
      <div
        ref={dialogRef}
        role="dialog"
        aria-modal="true"
        aria-labelledby="dialog-title"
        tabIndex={-1}
        onKeyDown={handleKeyDown}
      >
        <h2 id="dialog-title">{title}</h2>
        {children}
        <button onClick={onClose}>Close</button>
      </div>
    </>
  );
}
```

---

## Q159. How do you build accessible forms?

**Answer:**

### Complete Accessible Form Pattern

```tsx
function TransferForm() {
  const [errors, setErrors] = useState<Record<string, string>>({});
  const errorSummaryRef = useRef<HTMLDivElement>(null);

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    const newErrors = validate(formData);

    if (Object.keys(newErrors).length > 0) {
      setErrors(newErrors);
      // Focus error summary for screen reader announcement
      errorSummaryRef.current?.focus();
      return;
    }
    // Submit...
  };

  return (
    <form onSubmit={handleSubmit} noValidate aria-label="Transfer funds">
      {/* Error Summary — Announced when form has errors */}
      {Object.keys(errors).length > 0 && (
        <div
          ref={errorSummaryRef}
          role="alert"
          tabIndex={-1}
          aria-label={`${Object.keys(errors).length} errors in form`}
        >
          <h3>Please fix the following errors:</h3>
          <ul>
            {Object.entries(errors).map(([field, msg]) => (
              <li key={field}>
                <a href={`#${field}`}>{msg}</a>  {/* Link to field */}
              </li>
            ))}
          </ul>
        </div>
      )}

      {/* Accessible Input Field */}
      <div className="form-field">
        <label htmlFor="amount">
          Transfer Amount
          <span aria-hidden="true"> *</span>  {/* Visual asterisk */}
        </label>
        <input
          id="amount"
          name="amount"
          type="text"
          inputMode="decimal"
          aria-required="true"
          aria-invalid={!!errors.amount}
          aria-describedby="amount-help amount-error"
        />
        <span id="amount-help" className="help-text">
          Enter amount in USD (e.g., 1000.00)
        </span>
        {errors.amount && (
          <span id="amount-error" role="alert" className="error-text">
            {errors.amount}
          </span>
        )}
      </div>

      {/* Accessible Select */}
      <div className="form-field">
        <label htmlFor="from-account">From Account</label>
        <select
          id="from-account"
          aria-required="true"
          aria-invalid={!!errors.fromAccount}
          aria-describedby={errors.fromAccount ? 'from-error' : undefined}
        >
          <option value="">Select an account</option>
          <option value="savings">Savings (****1234) - $10,000</option>
          <option value="current">Current (****5678) - $25,000</option>
        </select>
        {errors.fromAccount && (
          <span id="from-error" role="alert">{errors.fromAccount}</span>
        )}
      </div>

      {/* Submit with loading state */}
      <button
        type="submit"
        aria-busy={isSubmitting}
        disabled={isSubmitting}
      >
        {isSubmitting ? 'Processing...' : 'Transfer'}
      </button>
    </form>
  );
}
```

### Key Form Accessibility Rules:

| Rule | Implementation |
|------|---------------|
| Every input has a label | `<label htmlFor="id">` or `aria-label` |
| Errors linked to fields | `aria-describedby="error-id"` |
| Required fields marked | `aria-required="true"` + visual indicator |
| Invalid state communicated | `aria-invalid="true"` when error exists |
| Error announced immediately | `role="alert"` on error message |
| Error summary focusable | `tabIndex={-1}` + focus on submit failure |
| Help text associated | `aria-describedby` includes help span |
| Group related fields | `<fieldset>` + `<legend>` |

---

## Q160. How do you handle keyboard navigation patterns?

**Answer:**

### Standard Keyboard Patterns (WAI-ARIA Authoring Practices)

| Component | Keys | Behavior |
|-----------|------|----------|
| **Button** | Enter, Space | Activate |
| **Link** | Enter | Navigate |
| **Menu** | Arrow Down/Up | Move between items |
| **Menu** | Enter | Select item |
| **Menu** | Escape | Close menu |
| **Tabs** | Arrow Left/Right | Switch tab |
| **Dialog** | Escape | Close dialog |
| **Dialog** | Tab | Cycle within dialog (trapped) |
| **Combobox** | Arrow Down | Open list / move to next |
| **Combobox** | Enter | Select option |
| **Tree** | Arrow Right | Expand / move to child |
| **Tree** | Arrow Left | Collapse / move to parent |
| **Grid/Table** | Arrow keys | Move between cells |

### Custom Keyboard Handler for Menu

```tsx
function Menu({ items, onSelect }: MenuProps) {
  const [activeIndex, setActiveIndex] = useState(0);
  const itemRefs = useRef<HTMLLIElement[]>([]);

  const handleKeyDown = (e: React.KeyboardEvent) => {
    switch (e.key) {
      case 'ArrowDown':
        e.preventDefault();
        setActiveIndex(i => (i + 1) % items.length);
        break;
      case 'ArrowUp':
        e.preventDefault();
        setActiveIndex(i => (i - 1 + items.length) % items.length);
        break;
      case 'Home':
        e.preventDefault();
        setActiveIndex(0);
        break;
      case 'End':
        e.preventDefault();
        setActiveIndex(items.length - 1);
        break;
      case 'Enter':
      case ' ':
        e.preventDefault();
        onSelect(items[activeIndex]);
        break;
      case 'Escape':
        // Close menu — handled by parent
        break;
    }
  };

  useEffect(() => {
    itemRefs.current[activeIndex]?.focus();
  }, [activeIndex]);

  return (
    <ul role="menu" onKeyDown={handleKeyDown}>
      {items.map((item, index) => (
        <li
          key={item.id}
          ref={el => { if (el) itemRefs.current[index] = el; }}
          role="menuitem"
          tabIndex={index === activeIndex ? 0 : -1}  // Roving tabindex
          onClick={() => onSelect(item)}
        >
          {item.label}
        </li>
      ))}
    </ul>
  );
}
```

### Roving Tabindex Pattern

```tsx
// Only ONE item in a group is tabbable at a time
// Arrow keys move focus WITHIN the group
// Tab moves focus OUT of the group

function TabList({ tabs, activeTab, onChange }: TabListProps) {
  const handleKeyDown = (e: React.KeyboardEvent, index: number) => {
    let newIndex = index;

    switch (e.key) {
      case 'ArrowRight':
        newIndex = (index + 1) % tabs.length;
        break;
      case 'ArrowLeft':
        newIndex = (index - 1 + tabs.length) % tabs.length;
        break;
      case 'Home':
        newIndex = 0;
        break;
      case 'End':
        newIndex = tabs.length - 1;
        break;
      default:
        return;
    }

    e.preventDefault();
    onChange(tabs[newIndex].id);
    // Focus the new tab
    document.getElementById(`tab-${tabs[newIndex].id}`)?.focus();
  };

  return (
    <div role="tablist">
      {tabs.map((tab, index) => (
        <button
          key={tab.id}
          id={`tab-${tab.id}`}
          role="tab"
          aria-selected={activeTab === tab.id}
          aria-controls={`panel-${tab.id}`}
          tabIndex={activeTab === tab.id ? 0 : -1}  // Only active tab is tabbable
          onClick={() => onChange(tab.id)}
          onKeyDown={(e) => handleKeyDown(e, index)}
        >
          {tab.label}
        </button>
      ))}
    </div>
  );
}
```

---

## Q161. How do you handle live regions and dynamic content announcements?

**Answer:**

### Live Region Types

```tsx
// aria-live="polite" — Announces when user is idle (non-urgent)
// Use for: search results count, form save confirmation, content updates
<div aria-live="polite" aria-atomic="true">
  {searchResults.length} results found
</div>

// aria-live="assertive" — Interrupts immediately (urgent)
// Use for: errors, session timeouts, critical alerts
<div aria-live="assertive" role="alert">
  Session expires in 2 minutes
</div>

// role="status" — Equivalent to aria-live="polite"
<div role="status">Form saved successfully</div>

// role="alert" — Equivalent to aria-live="assertive"
<div role="alert">Payment failed: insufficient funds</div>
```

### Toast Notification System (Accessible)

```tsx
// Toasts must be announced to screen readers without stealing focus

function ToastContainer() {
  const { toasts } = useToastContext();

  return (
    <div
      aria-live="polite"
      aria-relevant="additions"
      className="toast-container"
    >
      {toasts.map(toast => (
        <div
          key={toast.id}
          role="status"
          className={`toast toast-${toast.type}`}
        >
          <span className="toast-message">{toast.message}</span>
          <button
            onClick={() => dismissToast(toast.id)}
            aria-label={`Dismiss: ${toast.message}`}
          >
            ×
          </button>
        </div>
      ))}
    </div>
  );
}

// For error toasts, use role="alert" instead of role="status"
```

### Loading State Announcements

```tsx
function DataGrid({ isLoading, data }: DataGridProps) {
  return (
    <div>
      {/* Screen reader announcement for loading state */}
      <div aria-live="polite" className="sr-only">
        {isLoading ? 'Loading data...' : `${data.length} rows loaded`}
      </div>

      {/* Visual loading indicator */}
      {isLoading && <Spinner aria-hidden="true" />}

      {/* Table */}
      <table aria-busy={isLoading}>
        {/* ... */}
      </table>
    </div>
  );
}
```

---

## Q162. How do you test accessibility in React applications?

**Answer:**

### Layer 1: Automated Testing (CI — catches ~30-40% of issues)

```tsx
// jest-axe — Accessibility assertions in unit tests
import { render } from '@testing-library/react';
import { axe, toHaveNoViolations } from 'jest-axe';

expect.extend(toHaveNoViolations);

describe('Button accessibility', () => {
  it('has no accessibility violations', async () => {
    const { container } = render(
      <Button variant="primary" onClick={() => {}}>Submit</Button>
    );
    const results = await axe(container);
    expect(results).toHaveNoViolations();
  });

  it('icon-only button has accessible name', async () => {
    const { container } = render(
      <Button variant="primary" aria-label="Delete item" onClick={() => {}}>
        <TrashIcon />
      </Button>
    );
    const results = await axe(container);
    expect(results).toHaveNoViolations();
  });

  it('disabled button is not in tab order', () => {
    render(<Button disabled>Disabled</Button>);
    const button = screen.getByRole('button');
    expect(button).toBeDisabled();
    // Native disabled buttons are already removed from tab order
  });
});
```

### Layer 2: Integration Testing (keyboard + screen reader behavior)

```tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

describe('Dialog accessibility', () => {
  it('traps focus inside dialog', async () => {
    const user = userEvent.setup();
    render(
      <Dialog isOpen={true} onClose={() => {}}>
        <input aria-label="Name" />
        <button>Save</button>
        <button>Cancel</button>
      </Dialog>
    );

    const nameInput = screen.getByLabelText('Name');
    const saveBtn = screen.getByText('Save');
    const cancelBtn = screen.getByText('Cancel');

    // Tab cycles within dialog
    await user.tab();
    expect(nameInput).toHaveFocus();
    await user.tab();
    expect(saveBtn).toHaveFocus();
    await user.tab();
    expect(cancelBtn).toHaveFocus();
    await user.tab();
    expect(nameInput).toHaveFocus(); // Wraps back!
  });

  it('closes on Escape', async () => {
    const onClose = jest.fn();
    render(<Dialog isOpen={true} onClose={onClose}><p>Content</p></Dialog>);

    await userEvent.keyboard('{Escape}');
    expect(onClose).toHaveBeenCalled();
  });

  it('announces dialog title to screen readers', () => {
    render(<Dialog isOpen={true} title="Confirm Transfer" onClose={() => {}}><p>Are you sure?</p></Dialog>);

    const dialog = screen.getByRole('dialog');
    expect(dialog).toHaveAttribute('aria-labelledby');
    expect(screen.getByText('Confirm Transfer')).toBeInTheDocument();
  });
});
```

### Layer 3: Storybook Accessibility Addon

```tsx
// .storybook/main.ts
export default {
  addons: ['@storybook/addon-a11y'],  // Adds a11y panel to every story
};

// Every story automatically shows:
// - Violations (must fix)
// - Passes (verified)
// - Incomplete (needs manual review)
```

### Layer 4: Manual Testing Checklist

```markdown
## Pre-Release A11y Checklist

### Keyboard Testing:
- [ ] Tab through entire page — logical order?
- [ ] All interactive elements reachable via Tab?
- [ ] Focus visible on every element?
- [ ] Escape closes modals/menus?
- [ ] Arrow keys work in menus/tabs/grids?
- [ ] No keyboard traps?

### Screen Reader Testing (NVDA on Windows):
- [ ] Page title announced on navigation?
- [ ] Headings hierarchy makes sense? (h1 → h2 → h3)
- [ ] Form labels read correctly?
- [ ] Error messages announced?
- [ ] Loading states announced?
- [ ] Dynamic content changes announced?
- [ ] Images have meaningful alt text?

### Visual Testing:
- [ ] Works at 200% zoom?
- [ ] Works at 400% zoom? (horizontal scrollbar OK, content not cut off)
- [ ] Sufficient color contrast? (4.5:1 text, 3:1 UI)
- [ ] Information not conveyed by color alone?
- [ ] Focus ring visible in all themes?
- [ ] Reduced motion preference respected?
```

---

## Q163. How do you handle color contrast and visual accessibility?

**Answer:**

### WCAG Contrast Requirements

| Element | Minimum Ratio | Example |
|---------|--------------|---------|
| Normal text (<18px) | **4.5:1** | Body text, labels, buttons |
| Large text (≥18px or 14px bold) | **3:1** | Headings, large buttons |
| UI components (borders, icons) | **3:1** | Input borders, icons, focus ring |
| Decorative elements | None | Background patterns, dividers |

### Implementing Contrast-Safe Tokens

```tsx
// tokens/colors.ts
// Every color pairing is pre-validated for WCAG AA

export const colorPairings = {
  // Background → Foreground (verified 4.5:1+)
  brand: {
    background: '#0078D4',  // Blue
    foreground: '#FFFFFF',  // White on blue = 4.7:1 ✅
  },
  danger: {
    background: '#D32F2F',  // Red
    foreground: '#FFFFFF',  // White on red = 4.6:1 ✅
  },
  surface: {
    background: '#FFFFFF',
    foreground: '#1A1A1A',  // Near-black on white = 17.4:1 ✅
  },
  // ❌ NEVER: light gray text on white — fails contrast
  // '#999999' on '#FFFFFF' = 2.8:1 — FAILS
};
```

### Information Not Conveyed by Color Alone

```tsx
// ❌ BAD: Only color indicates status
<span style={{ color: 'red' }}>{amount}</span>     // Color-blind users can't see this
<span style={{ color: 'green' }}>{amount}</span>

// ✅ GOOD: Color + icon + text
<span className="status-error">
  <ErrorIcon aria-hidden="true" />  {/* Visual icon */}
  <span className="amount">{amount}</span>
  <span className="sr-only">Overdue</span>  {/* Screen reader text */}
</span>

<span className="status-success">
  <CheckIcon aria-hidden="true" />
  <span className="amount">{amount}</span>
  <span className="sr-only">Paid</span>
</span>
```

### Respecting Motion Preferences

```css
/* Reduce animations for users who prefer reduced motion */
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}

/* Only animate when user hasn't opted out */
@media (prefers-reduced-motion: no-preference) {
  .toast-enter {
    animation: slideIn 300ms ease-out;
  }
}
```

```tsx
// React hook for reduced motion
function usePrefersReducedMotion(): boolean {
  const [prefersReduced, setPrefersReduced] = useState(false);

  useEffect(() => {
    const mq = window.matchMedia('(prefers-reduced-motion: reduce)');
    setPrefersReduced(mq.matches);
    const handler = (e: MediaQueryListEvent) => setPrefersReduced(e.matches);
    mq.addEventListener('change', handler);
    return () => mq.removeEventListener('change', handler);
  }, []);

  return prefersReduced;
}

// Usage
function AnimatedNotification({ message }: Props) {
  const reduceMotion = usePrefersReducedMotion();
  return (
    <div className={reduceMotion ? 'notification-static' : 'notification-animated'}>
      {message}
    </div>
  );
}
```

---

## Q164. How do you make data tables/grids accessible?

**Answer:**

```tsx
function AccessibleDataTable({ data, columns, caption }: DataTableProps) {
  const [sortColumn, setSortColumn] = useState<string | null>(null);
  const [sortDir, setSortDir] = useState<'asc' | 'desc'>('asc');
  const [announcement, setAnnouncement] = useState('');

  const handleSort = (column: string) => {
    const newDir = sortColumn === column && sortDir === 'asc' ? 'desc' : 'asc';
    setSortColumn(column);
    setSortDir(newDir);
    // Announce sort change to screen readers
    setAnnouncement(`Table sorted by ${column}, ${newDir}ending`);
  };

  return (
    <>
      {/* Live region for sort announcements */}
      <div aria-live="polite" className="sr-only">{announcement}</div>

      <table aria-label={caption}>
        <caption className="sr-only">{caption}</caption>
        <thead>
          <tr>
            {columns.map(col => (
              <th
                key={col.key}
                scope="col"
                aria-sort={
                  sortColumn === col.key
                    ? sortDir === 'asc' ? 'ascending' : 'descending'
                    : 'none'
                }
              >
                {col.sortable ? (
                  <button
                    onClick={() => handleSort(col.key)}
                    aria-label={`Sort by ${col.header}, currently ${
                      sortColumn === col.key ? sortDir : 'unsorted'
                    }`}
                  >
                    {col.header}
                    <SortIcon direction={sortColumn === col.key ? sortDir : undefined} />
                  </button>
                ) : (
                  col.header
                )}
              </th>
            ))}
          </tr>
        </thead>
        <tbody>
          {data.map(row => (
            <tr key={row.id}>
              {columns.map(col => (
                <td key={col.key}>{col.render ? col.render(row) : row[col.key]}</td>
              ))}
            </tr>
          ))}
        </tbody>
      </table>

      {/* Pagination with accessible labels */}
      <nav aria-label="Table pagination">
        <button aria-label="Previous page" disabled={page === 1}>←</button>
        <span aria-current="page">Page {page} of {totalPages}</span>
        <button aria-label="Next page" disabled={page === totalPages}>→</button>
      </nav>
    </>
  );
}
```

### Key Table A11y Rules:

| Rule | Implementation |
|------|---------------|
| Table must have a caption/label | `<caption>` or `aria-label` |
| Header cells use `<th scope="col">` | Tells screen reader "this is a column header" |
| Row headers use `<th scope="row">` | First column identifies the row |
| Sort state communicated | `aria-sort="ascending/descending/none"` |
| Sort change announced | `aria-live="polite"` region |
| Pagination labeled | `<nav aria-label="Table pagination">` |

---

## Q165. How do you implement accessible error handling and validation?

**Answer:**

### Inline Validation with Announcements

```tsx
function useFormValidation<T extends Record<string, unknown>>(
  values: T,
  validators: Record<keyof T, (value: unknown) => string | undefined>
) {
  const [errors, setErrors] = useState<Partial<Record<keyof T, string>>>({});
  const [touched, setTouched] = useState<Partial<Record<keyof T, boolean>>>({});
  const announcerRef = useRef<HTMLDivElement>(null);

  const validateField = (field: keyof T) => {
    const error = validators[field]?.(values[field]);
    setErrors(prev => ({ ...prev, [field]: error }));

    // Announce error to screen reader when field is blurred
    if (error && touched[field]) {
      // Use setTimeout so screen reader finishes reading the field first
      setTimeout(() => {
        if (announcerRef.current) {
          announcerRef.current.textContent = `Error: ${error}`;
        }
      }, 100);
    }
  };

  const ErrorAnnouncer = () => (
    <div
      ref={announcerRef}
      role="status"
      aria-live="assertive"
      aria-atomic="true"
      className="sr-only"
    />
  );

  return { errors, touched, setTouched, validateField, ErrorAnnouncer };
}
```

### Error Summary Pattern (WCAG Best Practice)

```tsx
// After submit, focus moves to error summary
// Each error links to its field — screen reader user can jump directly

function ErrorSummary({ errors }: { errors: Record<string, string> }) {
  const summaryRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (Object.keys(errors).length > 0) {
      summaryRef.current?.focus();
    }
  }, [errors]);

  if (Object.keys(errors).length === 0) return null;

  return (
    <div
      ref={summaryRef}
      role="alert"
      tabIndex={-1}
      aria-label={`Form has ${Object.keys(errors).length} errors`}
      className="error-summary"
    >
      <h3>There are {Object.keys(errors).length} errors in this form:</h3>
      <ol>
        {Object.entries(errors).map(([field, message]) => (
          <li key={field}>
            <a href={`#${field}`} onClick={() => document.getElementById(field)?.focus()}>
              {message}
            </a>
          </li>
        ))}
      </ol>
    </div>
  );
}
```

---

## Q166. What are common accessibility anti-patterns in React?

**Answer:**

```tsx
// ❌ ANTI-PATTERN 1: div buttons (no keyboard support, no semantics)
<div onClick={handleClick} className="button">Submit</div>
// ✅ FIX: Use native button
<button onClick={handleClick}>Submit</button>

// ❌ ANTI-PATTERN 2: Placeholder as label (disappears on focus)
<input placeholder="Email address" />
// ✅ FIX: Always use a visible label
<label htmlFor="email">Email address</label>
<input id="email" placeholder="name@example.com" />

// ❌ ANTI-PATTERN 3: Click handler on non-interactive element
<span onClick={toggleMenu}>Menu</span>
// ✅ FIX: Use button (or add role, tabIndex, keyDown)
<button onClick={toggleMenu} aria-expanded={isOpen}>Menu</button>

// ❌ ANTI-PATTERN 4: Auto-focus stealing (jarring for screen readers)
useEffect(() => { inputRef.current?.focus(); }, []); // On every mount!
// ✅ FIX: Only auto-focus when user initiated the interaction
// E.g., focus search input when user clicks "Search" button

// ❌ ANTI-PATTERN 5: Images without alt text
<img src={avatar} />
// ✅ FIX: Meaningful alt or empty alt for decorative
<img src={avatar} alt="John Smith's profile photo" />  // Meaningful
<img src={decorativeLine} alt="" />  // Decorative — empty alt = ignored

// ❌ ANTI-PATTERN 6: Using aria-hidden on focusable content
<div aria-hidden="true">
  <button>Still focusable!</button>  // Trap! Focusable but invisible to AT
</div>
// ✅ FIX: Also add tabIndex="-1" or inert to all children

// ❌ ANTI-PATTERN 7: Color-only error indication
<input style={{ borderColor: hasError ? 'red' : 'gray' }} />
// ✅ FIX: Error text + icon + aria-invalid
<input aria-invalid={hasError} aria-describedby="error-msg" />
{hasError && <span id="error-msg" role="alert">⚠ Required field</span>}

// ❌ ANTI-PATTERN 8: Infinite scroll without keyboard access
// Screen reader users can never reach the footer!
// ✅ FIX: Provide "Load more" button as alternative
<button onClick={loadMore}>Load more results ({remaining} remaining)</button>

// ❌ ANTI-PATTERN 9: Custom select without ARIA
<div className="dropdown" onClick={toggle}>{selected}</div>
// ✅ FIX: Full ARIA combobox pattern or use native <select>
```

---

## Q167. How do you enforce accessibility standards across a team?

**Answer:**

### Multi-Layer Enforcement

```
Layer 1: DESIGN (prevent issues before code)
─────────────────────────────────────────────
- Design tokens with verified contrast ratios
- Component specs include keyboard behavior
- Designers use a11y Figma plugins (Stark, A11y Annotation Kit)

Layer 2: DEVELOPMENT (catch during coding)
─────────────────────────────────────────────
- ESLint: eslint-plugin-jsx-a11y (catches 20+ patterns)
- TypeScript: Required aria-label on icon-only buttons
- Storybook: @storybook/addon-a11y panel

Layer 3: TESTING (catch in PR)
─────────────────────────────────────────────
- jest-axe in every component test
- CI fails on any a11y violation
- Playwright a11y audit on key pages

Layer 4: REVIEW (human verification)
─────────────────────────────────────────────
- PR checklist includes a11y items
- Quarterly screen reader testing sessions
- Annual third-party a11y audit
```

### ESLint Rules (Automated Prevention)

```json
{
  "plugins": ["jsx-a11y"],
  "rules": {
    "jsx-a11y/alt-text": "error",
    "jsx-a11y/anchor-has-content": "error",
    "jsx-a11y/aria-props": "error",
    "jsx-a11y/aria-role": "error",
    "jsx-a11y/aria-unsupported-elements": "error",
    "jsx-a11y/click-events-have-key-events": "error",
    "jsx-a11y/heading-has-content": "error",
    "jsx-a11y/label-has-associated-control": "error",
    "jsx-a11y/no-autofocus": "warn",
    "jsx-a11y/no-noninteractive-element-interactions": "error",
    "jsx-a11y/no-static-element-interactions": "error",
    "jsx-a11y/role-has-required-aria-props": "error",
    "jsx-a11y/tabindex-no-positive": "error"
  }
}
```

### TypeScript Enforcement

```tsx
// Force aria-label on icon-only buttons via types
type IconButtonProps = {
  icon: React.ReactNode;
  'aria-label': string;  // REQUIRED — TypeScript won't compile without it
  onClick: () => void;
};

// ❌ TypeScript error: Property 'aria-label' is missing
<IconButton icon={<TrashIcon />} onClick={del} />

// ✅ Compiles
<IconButton icon={<TrashIcon />} aria-label="Delete item" onClick={del} />
```

---

## Key Interview Lines for Accessibility

1. "We target WCAG 2.1 AA as our minimum standard. Automated tools catch 30-40% of issues; the rest requires manual testing with keyboard and screen readers."

2. "Every component in our design system has built-in accessibility — proper ARIA roles, keyboard support, and focus management. Consumers get accessibility for free."

3. "We enforce accessibility at 4 layers: design (contrast-verified tokens), development (eslint-plugin-jsx-a11y), CI (jest-axe with zero-violation policy), and review (quarterly screen reader audits)."

4. "Focus management in SPAs is the most overlooked issue — we announce route changes via live regions and move focus to the main content on navigation."

5. "In financial applications, accessibility isn't just ethical — it's legal. Many of our enterprise clients require WCAG AA compliance as a contractual obligation."

6. "I enforce accessible patterns through TypeScript types — icon-only buttons require `aria-label` at compile time, not just at code review time."

