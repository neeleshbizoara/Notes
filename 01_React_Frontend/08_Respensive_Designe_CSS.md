# Section 8: Responsive Design & CSS (7 Questions & Answers)



---

## Q76. Explain CSS-in-JS approaches (styled-components, Emotion) vs CSS Modules vs utility-first (Tailwind).

**Answer:**

| Approach | How It Works | Pros | Cons |
|---|---|---|---|
| **CSS Modules** | Write regular CSS, scoped by file | No runtime cost, familiar syntax | Limited dynamic styling |
| **Styled-components** | Write CSS in JS using template literals | Dynamic styling, auto-scoping | Runtime cost, larger bundle |
| **Emotion** | Similar to styled-components | Smaller bundle, flexible API | Runtime cost |
| **Tailwind CSS** | Utility classes directly in JSX | Fast development, consistent design | Verbose HTML, learning curve |

**CSS Modules:**
```css
/* AccountCard.module.css */
.card { background: white; border-radius: 8px; padding: 16px; }
.balance { font-size: 24px; font-weight: bold; color: #2e7d32; }
```
```jsx
import styles from './AccountCard.module.css';

function AccountCard({ balance }) {
  return (
    <div className={styles.card}>
      <span className={styles.balance}>₹{balance}</span>
    </div>
  );
}
// Compiles to: class="AccountCard_card__x3j2k" (unique, no conflicts)
```

**Styled-components:**
```jsx
import styled from 'styled-components';

const Card = styled.div`
  background: white;
  border-radius: 8px;
  padding: 16px;
  border-left: 4px solid ${props => props.active ? '#4caf50' : '#ddd'};
`;

const Balance = styled.span`
  font-size: 24px;
  color: ${props => props.amount >= 0 ? '#2e7d32' : '#c62828'};
`;

function AccountCard({ balance, active }) {
  return (
    <Card active={active}>
      <Balance amount={balance}>₹{balance}</Balance>
    </Card>
  );
}
```

**Tailwind CSS:**
```jsx
function AccountCard({ balance, active }) {
  return (
    <div className={`bg-white rounded-lg p-4 border-l-4 ${active ? 'border-green-500' : 'border-gray-300'}`}>
      <span className={`text-2xl font-bold ${balance >= 0 ? 'text-green-700' : 'text-red-700'}`}>
        ₹{balance}
      </span>
    </div>
  );
}
```

**My recommendation:** "For a financial services project, I prefer CSS Modules because they have zero runtime cost, work well with existing CSS knowledge, and are easy for new team members. If we need heavy dynamic styling, I'd add styled-components for specific components."

---

## Q77. How do you implement a mobile-first responsive design for a banking dashboard?

**Answer:**

**Mobile-first** = Design for mobile first, then add rules for larger screens using `min-width` media queries.

```css
/* Mobile first (default styles are for mobile) */
.dashboard {
  display: flex;
  flex-direction: column;  /* Stack vertically on mobile */
  padding: 12px;
  gap: 12px;
}

.account-card {
  width: 100%;  /* Full width on mobile */
}

/* Tablet (768px and up) */
@media (min-width: 768px) {
  .dashboard {
    flex-direction: row;
    flex-wrap: wrap;
    padding: 20px;
    gap: 16px;
  }
  .account-card {
    width: calc(50% - 8px);  /* 2 cards per row */
  }
}

/* Desktop (1024px and up) */
@media (min-width: 1024px) {
  .dashboard {
    padding: 32px;
    gap: 24px;
  }
  .account-card {
    width: calc(33.33% - 16px);  /* 3 cards per row */
  }
}
```

**React responsive hook:**
```jsx
function useMediaQuery(query) {
  const [matches, setMatches] = useState(
    window.matchMedia(query).matches
  );

  useEffect(() => {
    const media = window.matchMedia(query);
    const handler = (e) => setMatches(e.matches);
    media.addEventListener('change', handler);
    return () => media.removeEventListener('change', handler);
  }, [query]);

  return matches;
}

// Usage
function Dashboard() {
  const isMobile = useMediaQuery('(max-width: 767px)');
  const isTablet = useMediaQuery('(min-width: 768px) and (max-width: 1023px)');

  return (
    <div>
      {isMobile ? <MobileNavbar /> : <DesktopNavbar />}
      <AccountGrid columns={isMobile ? 1 : isTablet ? 2 : 3} />
    </div>
  );
}
```

**Key tips for banking responsive design:**
- Hide non-essential info on mobile (show full details on desktop)
- Make touch targets at least 44x44px on mobile
- Ensure forms are usable with mobile keyboards
- Test with actual devices (not just browser resize)

---

## Q78. What is the CSS `contain` property? How does it improve performance?

**Answer:**

`contain` tells the browser that an element's contents are independent from the rest of the page, so the browser can optimize rendering.

```css
/* Without contain: changing one card forces browser to recalculate layout for ENTIRE page */

/* With contain: browser only recalculates layout within this element */
.transaction-item {
  contain: layout style paint;
}

/* contain: layout  → Layout changes inside don't affect outside */
/* contain: style   → Styles inside don't leak out */
/* contain: paint   → Content won't be drawn outside this box */
/* contain: size    → Element's size doesn't depend on children */
/* contain: content → Shorthand for layout + style + paint */
/* contain: strict  → Shorthand for all of the above */
```

**Banking use case — Long transaction list:**
```css
.transaction-list {
  contain: content;  /* Each item is independent */
  overflow-y: auto;
  height: 500px;
}

.transaction-item {
  contain: layout style;  /* Changing one item doesn't affect others */
}
```

**Performance benefit:**
- When one transaction item expands/changes, the browser doesn't need to recalculate the layout of 1000 other items
- Can improve scrolling performance significantly for long lists

---

## Q79. Explain Flexbox vs CSS Grid. When do you use each?

**Answer:**

| Feature | Flexbox | CSS Grid |
|---|---|---|
| Direction | One-dimensional (row OR column) | Two-dimensional (rows AND columns) |
| Best for | Alignment, distribution in one axis | Complex layouts with rows and columns |
| Content-based | Items size based on content | Items size based on grid definition |

**Flexbox — Best for navigation bars, card rows, centering:**
```css
/* Navbar */
.navbar {
  display: flex;
  justify-content: space-between;  /* Logo left, menu right */
  align-items: center;             /* Vertically centered */
}

/* Card row with equal spacing */
.account-cards {
  display: flex;
  gap: 16px;
  flex-wrap: wrap;
}

/* Perfect centering */
.loading-container {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;
}
```

**CSS Grid — Best for page layouts, dashboards, data tables:**
```css
/* Banking Dashboard Layout */
.dashboard {
  display: grid;
  grid-template-columns: 250px 1fr 300px;  /* Sidebar | Main | Notifications */
  grid-template-rows: 60px 1fr 40px;       /* Header | Content | Footer */
  grid-template-areas:
    "header  header  header"
    "sidebar main   notifications"
    "footer  footer  footer";
  height: 100vh;
}

.header        { grid-area: header; }
.sidebar       { grid-area: sidebar; }
.main-content  { grid-area: main; }
.notifications { grid-area: notifications; }
.footer        { grid-area: footer; }

/* Responsive: stack on mobile */
@media (max-width: 768px) {
  .dashboard {
    grid-template-columns: 1fr;
    grid-template-areas:
      "header"
      "main"
      "footer";
  }
  .sidebar, .notifications { display: none; }
}
```

**My rule of thumb:**
- **Flexbox**: Aligning items in a single direction (navbar, button groups, form layouts)
- **Grid**: Complex 2D layouts (page structure, dashboards, card grids)
- Often I use **Grid for the page layout** and **Flexbox inside each section**

---

## Q80. How do you handle cross-browser compatibility issues?

**Answer:**

**1. Use Autoprefixer (add vendor prefixes automatically):**
```css
/* You write: */
.card { display: flex; }

/* Autoprefixer adds: */
.card {
  display: -webkit-flex;  /* Safari */
  display: -ms-flexbox;   /* IE10 */
  display: flex;
}
```

**2. Set browserslist target in package.json:**
```json
{
  "browserslist": [
    ">0.2%",
    "not dead",
    "not ie 11"
  ]
}
```

**3. Use CSS feature detection with `@supports`:**
```css
/* Fallback for browsers that don't support Grid */
.dashboard {
  display: flex;
  flex-wrap: wrap;
}

@supports (display: grid) {
  .dashboard {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
  }
}
```

**4. Polyfills for JavaScript features:**
```javascript
// babel.config.js
module.exports = {
  presets: [
    ['@babel/preset-env', {
      useBuiltIns: 'usage',  // Only import polyfills for features you use
      corejs: 3
    }]
  ]
};
```

**5. CSS Reset/Normalize:**
```css
/* Remove default browser inconsistencies */
*, *::before, *::after {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}
```

**Common issues I've dealt with:**
- Safari date input handling (different format)
- Safari flexbox gap not supported (older versions)
- IE11 CSS Grid not supported (use feature detection)
- Mobile Safari 100vh includes address bar (use `dvh` or JS)

---

## Q81. How do you implement dark mode in a React application?

**Answer:**

```jsx
// 1. Create a ThemeContext
const ThemeContext = createContext();

function ThemeProvider({ children }) {
  const [theme, setTheme] = useState(() => {
    // Check saved preference or system preference
    return localStorage.getItem('theme') ||
      (window.matchMedia('(prefers-color-scheme: dark)').matches ? 'dark' : 'light');
  });

  function toggleTheme() {
    const newTheme = theme === 'light' ? 'dark' : 'light';
    setTheme(newTheme);
    localStorage.setItem('theme', newTheme);
  }

  useEffect(() => {
    document.documentElement.setAttribute('data-theme', theme);
  }, [theme]);

  return (
    <ThemeContext.Provider value={{ theme, toggleTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}
```

```css
/* 2. Define CSS variables for each theme */
:root[data-theme="light"] {
  --bg-primary: #ffffff;
  --bg-secondary: #f5f5f5;
  --text-primary: #1a1a1a;
  --text-secondary: #666666;
  --accent: #1976d2;
  --success: #2e7d32;
  --danger: #c62828;
  --border: #e0e0e0;
}

:root[data-theme="dark"] {
  --bg-primary: #121212;
  --bg-secondary: #1e1e1e;
  --text-primary: #e0e0e0;
  --text-secondary: #999999;
  --accent: #64b5f6;
  --success: #66bb6a;
  --danger: #ef5350;
  --border: #333333;
}

/* 3. Use variables in components */
.account-card {
  background: var(--bg-primary);
  color: var(--text-primary);
  border: 1px solid var(--border);
}

.balance-positive { color: var(--success); }
.balance-negative { color: var(--danger); }
```

```jsx
// 4. Toggle button
function ThemeToggle() {
  const { theme, toggleTheme } = useContext(ThemeContext);
  return (
    <button onClick={toggleTheme}>
      {theme === 'light' ? '🌙 Dark Mode' : '☀️ Light Mode'}
    </button>
  );
}
```

---

## Q82. How do you implement keyboard navigation and accessibility (a11y) in a React application?

**Answer:**

Accessibility is a **legal requirement** in financial services (WCAG 2.1 AA compliance). Key areas: keyboard navigation, ARIA attributes, focus management, and screen reader support.

**1. Keyboard navigation — focus management:**
```jsx
// Focus trap for modals (user can't tab outside)
function Modal({ isOpen, onClose, children }) {
  const modalRef = useRef();
  const previousFocus = useRef();

  useEffect(() => {
    if (isOpen) {
      previousFocus.current = document.activeElement; // Remember where focus was
      modalRef.current?.focus();                      // Move focus into modal
    }
    return () => {
      previousFocus.current?.focus();                 // Restore focus on close
    };
  }, [isOpen]);

  // Trap Tab key inside modal
  const handleKeyDown = (e) => {
    if (e.key === 'Escape') return onClose();
    if (e.key !== 'Tab') return;

    const focusable = modalRef.current.querySelectorAll(
      'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
    );
    const first = focusable[0];
    const last = focusable[focusable.length - 1];

    if (e.shiftKey && document.activeElement === first) {
      e.preventDefault();
      last.focus();      // Shift+Tab from first → wrap to last
    } else if (!e.shiftKey && document.activeElement === last) {
      e.preventDefault();
      first.focus();     // Tab from last → wrap to first
    }
  };

  if (!isOpen) return null;

  return (
    <div className="modal-overlay" onClick={onClose}>
      <div
        ref={modalRef}
        role="dialog"
        aria-modal="true"
        aria-labelledby="modal-title"
        tabIndex={-1}
        onKeyDown={handleKeyDown}
        onClick={e => e.stopPropagation()}
      >
        {children}
      </div>
    </div>
  );
}
```

**2. ARIA attributes — making custom components accessible:**
```jsx
// Custom dropdown (not a native <select>)
function CustomDropdown({ label, options, value, onChange }) {
  const [isOpen, setIsOpen] = useState(false);
  const [activeIndex, setActiveIndex] = useState(-1);

  const handleKeyDown = (e) => {
    switch (e.key) {
      case 'ArrowDown':
        e.preventDefault();
        setActiveIndex(i => Math.min(i + 1, options.length - 1));
        if (!isOpen) setIsOpen(true);
        break;
      case 'ArrowUp':
        e.preventDefault();
        setActiveIndex(i => Math.max(i - 1, 0));
        break;
      case 'Enter':
      case ' ':
        e.preventDefault();
        if (isOpen && activeIndex >= 0) {
          onChange(options[activeIndex]);
          setIsOpen(false);
        } else {
          setIsOpen(true);
        }
        break;
      case 'Escape':
        setIsOpen(false);
        break;
    }
  };

  return (
    <div>
      <label id="dropdown-label">{label}</label>
      <button
        role="combobox"
        aria-expanded={isOpen}
        aria-haspopup="listbox"
        aria-labelledby="dropdown-label"
        aria-activedescendant={activeIndex >= 0 ? `option-${activeIndex}` : undefined}
        onKeyDown={handleKeyDown}
        onClick={() => setIsOpen(!isOpen)}
      >
        {value || 'Select...'}
      </button>
      {isOpen && (
        <ul role="listbox" aria-labelledby="dropdown-label">
          {options.map((opt, i) => (
            <li
              key={opt.value}
              id={`option-${i}`}
              role="option"
              aria-selected={opt.value === value}
              className={i === activeIndex ? 'active' : ''}
              onClick={() => { onChange(opt); setIsOpen(false); }}
            >
              {opt.label}
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

**3. Live regions — announcing dynamic changes to screen readers:**
```jsx
// Announce transaction success/failure
function PaymentStatus({ status, message }) {
  return (
    <div
      role="status"           {/* polite: waits for screen reader to finish */}
      aria-live="polite"
      aria-atomic="true"      {/* reads entire region, not just changed part */}
    >
      {status === 'success' && <p>✅ {message}</p>}
      {status === 'error' && (
        <p role="alert">       {/* alert: interrupts screen reader immediately */}
          ❌ {message}
        </p>
      )}
    </div>
  );
}
```

**4. Skip navigation link (for keyboard users):**
```jsx
function App() {
  return (
    <>
      <a href="#main-content" className="skip-link">
        Skip to main content
      </a>
      <Header />
      <nav>{/* navigation links */}</nav>
      <main id="main-content" tabIndex={-1}>
        {/* page content */}
      </main>
    </>
  );
}

// CSS — hidden until focused
.skip-link {
  position: absolute;
  top: -40px;
  left: 0;
  z-index: 100;
}
.skip-link:focus {
  top: 0;    /* Becomes visible when Tab key reaches it */
}
```

**Essential ARIA cheat sheet:**

| ARIA Attribute | Purpose | Example |
|---|---|---|
| `role="dialog"` | Identifies a modal | Modal overlays |
| `aria-label` | Names an element (no visible label) | Icon-only buttons |
| `aria-labelledby` | Names element using another element's text | Form sections |
| `aria-describedby` | Links error message to input | Validation errors |
| `aria-live="polite"` | Announces changes when idle | Status updates |
| `aria-live="assertive"` | Announces changes immediately | Error alerts |
| `aria-expanded` | Is this section open/closed? | Accordions, dropdowns |
| `aria-hidden="true"` | Hide from screen readers | Decorative icons |
| `tabIndex={0}` | Make focusable (in tab order) | Custom interactive elements |
| `tabIndex={-1}` | Focusable via JS only (not Tab) | Programmatic focus targets |


**Quick accessibility checklist:**
- ✅ All interactive elements reachable by keyboard (Tab/Shift+Tab)
- ✅ Visible focus indicator (`outline`) — never `outline: none` without replacement
- ✅ Modals trap focus and restore it on close
- ✅ Images have `alt` text (or `alt=""` for decorative)
- ✅ Form inputs have `<label>` or `aria-label`
- ✅ Error messages linked with `aria-describedby`
- ✅ Color contrast ratio ≥ 4.5:1 (text), ≥ 3:1 (large text)
- ✅ Dynamic content changes announced via `aria-live`
- ✅ Skip navigation link present

----

## Q83. How do you approach accessibility in React applications? What tools, techniques, and best practices do you use to ensure your applications are accessible to all users?

**Answer:**

My approach to accessibility (a11y) in React applications is systematic and proactive, focusing on both prevention and validation:

### 1. Semantic HTML First
- I always use semantic HTML elements (`<button>`, `<nav>`, `<header>`, `<main>`, etc.) to provide a strong foundation for accessibility.
- I avoid using `<div>` and `<span>` for interactive elements, ensuring screen readers and assistive tech can interpret the UI correctly.

### 2. ARIA Attributes (When Needed)
- I use ARIA roles, states, and properties (like `aria-label`, `aria-live`, `aria-disabled`) only when native HTML cannot provide the required semantics.
- I ensure ARIA is not overused, as improper use can harm accessibility.

### 3. Keyboard Navigation
- I ensure all interactive elements are reachable and usable via keyboard (Tab, Enter, Space, Esc, etc.).
- I use the `tabIndex` attribute and manage focus programmatically with React refs when building custom components (e.g., modals, dropdowns).

### 4. Accessible Forms and Components
- I associate labels with inputs using `htmlFor` and `id`.
- I provide clear error messages and use appropriate roles for alerts and validation.

### 5. Color Contrast & Visual Design
- I check color contrast ratios to meet WCAG AA/AAA standards.
- I avoid using color as the only means of conveying information.

### 6. Testing & Tooling
- **Linting:** I use `eslint-plugin-jsx-a11y` to catch common accessibility issues during development.
- **Automated Testing:** I use tools like Axe (via browser extension or `jest-axe` for unit tests) to scan for accessibility violations.
- **Manual Testing:** I test with screen readers (NVDA, VoiceOver) and keyboard-only navigation.

### 7. React-Specific Best Practices
- I use accessible React component libraries (like Reach UI, Radix UI, or Material-UI) that follow WAI-ARIA guidelines.
- I ensure custom components expose proper roles and keyboard handlers.
- I leverage React’s `useEffect` and refs to manage focus for dynamic content (e.g., focus trapping in modals).

### 8. Continuous Education & Audits
- I stay updated with WCAG guidelines and regularly audit apps for accessibility regressions.

**Example:**
When building a custom modal, I:
- Trap focus within the modal using refs.
- Restore focus to the triggering element on close.
- Use `role="dialog"` and `aria-modal="true"`.
- Provide a descriptive label via `aria-labelledby`.

**Summary:**
Accessibility is a core part of my development workflow. I combine semantic HTML, ARIA, keyboard support, color contrast, automated and manual testing, and React-specific patterns to ensure my applications are usable by everyone.

---
Unit |  Relative To Main | Use Case
-----|-------------------|----------
rem Root | <html> element | Standardized font sizes, margins, a