# Section 15: Design System Migration — Fluent UI to Motif (12 Questions & Answers)

> **Why this matters for XXXX:** The JD explicitly states "Lead the transition from the current frontend component model toward a future design-system target." The current state is Fluent UI; the target is Motif-aligned. This is THE core responsibility of this role.

---

## Q134. What is Fluent UI and how is it architected (v9)?

**Answer:**

Fluent UI React v9 (`@fluentui/react-components`) is Microsoft's design system for enterprise apps.

### Core Architecture Concepts:

```tsx
// 1. FluentProvider — Theme + direction + SSR config at the root
import { FluentProvider, webLightTheme } from '@fluentui/react-components';

function App() {
  return (
    <FluentProvider theme={webLightTheme}>
      <MainLayout />
    </FluentProvider>
  );
}

// 2. Design Tokens — CSS custom properties under the hood
// Fluent v9 uses CSS variables for theming (NOT runtime JS objects)
// This means: zero-cost theme switching, SSR-friendly, performant
const customTheme = {
  ...webLightTheme,
  colorBrandBackground: '#0078D4',   // Maps to --colorBrandBackground CSS var
  colorNeutralForeground1: '#242424',
};

// 3. Slot-based composition (Fluent v9's key pattern)
import { Button } from '@fluentui/react-components';

// Slots let you customize any internal part of a component
<Button
  icon={<CalendarIcon />}         // icon slot
  appearance="primary"
  size="medium"
>
  Schedule Meeting
</Button>

// 4. Griffel (CSS-in-JS engine) — atomic CSS at build time
import { makeStyles, tokens } from '@fluentui/react-components';

const useStyles = makeStyles({
  root: {
    display: 'flex',
    padding: tokens.spacingHorizontalM,
    backgroundColor: tokens.colorNeutralBackground1,
    borderRadius: tokens.borderRadiusMedium,
  },
});
```

### Fluent UI v9 Component Categories:

| Category | Components |
|----------|-----------|
| **Layout** | Flex (deprecated, use CSS), Divider |
| **Input** | Input, Textarea, Select, Checkbox, Radio, Switch, Slider, SpinButton |
| **Buttons** | Button, ToggleButton, MenuButton, SplitButton, CompoundButton |
| **Navigation** | Tab, TabList, Breadcrumb, Link |
| **Data Display** | Table, DataGrid, Tree, Accordion, Card, Badge, Avatar |
| **Feedback** | Dialog, Toast, MessageBar, ProgressBar, Spinner |
| **Overlay** | Popover, Tooltip, Menu, Drawer |

---

## Q135. What is a Motif-aligned design system? How does it differ from Fluent?

**Answer:**

**Note:** "Motif" in this JD context likely refers to an internal/proprietary design system target. The principles below apply to any modern design system migration.

### Design System Comparison Framework:

| Aspect | Fluent UI v9 | Motif-aligned (Target) |
|--------|-------------|----------------------|
| **Philosophy** | Microsoft's one-size-fits-all enterprise system | Domain-specific, tailored to XXXX's needs |
| **Tokens** | Fluent token taxonomy (600+ tokens) | Custom semantic tokens mapped to brand |
| **Composition** | Slot-based | Likely composition-first (children pattern) |
| **Styling** | Griffel (atomic CSS-in-JS) | Could be CSS Modules, Tailwind, or tokens-only |
| **Bundling** | Tree-shakeable per-component imports | Same expectation |
| **Customization** | Theme overrides via FluentProvider | Likely more granular control |

### Key Migration Challenges:

1. **Token mapping** — Fluent's `tokens.colorBrandBackground` → Motif's equivalent
2. **API differences** — Fluent's slot pattern vs Motif's composition pattern
3. **Style system** — Griffel (CSS-in-JS) → potentially different approach
4. **Behavioral differences** — Focus management, keyboard handling built into Fluent
5. **Testing coverage** — Must ensure no visual or behavioral regressions

**Interview talking point:** "I'd first build a complete token mapping document — this is the foundation. If the semantic intent is preserved (e.g., 'brand primary' maps to 'brand primary'), the visual migration is straightforward. The harder part is behavioral parity — focus traps, keyboard patterns, ARIA attributes."

---

## Q136. How would you architect a design system migration strategy?

**Answer:**

### The Strangler Fig Pattern for UI

```
Phase 1: Abstraction Layer
────────────────────────────
Consumers → Abstraction Layer → Fluent UI (current)

Phase 2: Incremental Replacement
────────────────────────────
Consumers → Abstraction Layer → Fluent UI (most components)
                               → Motif (migrated components)

Phase 3: Complete Migration
────────────────────────────
Consumers → Abstraction Layer → Motif (all components)

Phase 4: Remove Abstraction (optional)
────────────────────────────
Consumers → Motif (direct)
```

### Implementation Architecture

```
src/
├── design-system/              ← ABSTRACTION LAYER (consumers import from here)
│   ├── index.ts                ← Barrel exports
│   ├── tokens/
│   │   ├── colors.ts           ← Semantic color tokens
│   │   ├── spacing.ts          ← Spacing scale
│   │   ├── typography.ts       ← Font sizes, weights
│   │   └── index.ts
│   ├── components/
│   │   ├── Button/
│   │   │   ├── Button.tsx      ← Implementation (currently wraps Fluent, will wrap Motif)
│   │   │   ├── Button.types.ts ← Stable public API types (NEVER changes during migration)
│   │   │   ├── Button.test.tsx ← Tests against public API
│   │   │   └── index.ts
│   │   ├── Input/
│   │   ├── Dialog/
│   │   ├── DataGrid/
│   │   └── ...
│   └── providers/
│       └── ThemeProvider.tsx    ← Wraps FluentProvider today, Motif tomorrow
├── features/                   ← Business features ONLY import from design-system/
│   ├── accounts/
│   ├── payments/
│   └── ...
```

### The Key Principle: Stable Public API

```tsx
// design-system/components/Button/Button.types.ts
// This interface NEVER changes during migration — it's the contract with consumers

export interface DSButtonProps {
  /** Visual variant */
  variant: 'primary' | 'secondary' | 'outline' | 'danger' | 'ghost';
  /** Size */
  size?: 'sm' | 'md' | 'lg';
  /** Button content */
  children: React.ReactNode;
  /** Click handler */
  onClick?: () => void;
  /** Disabled state */
  disabled?: boolean;
  /** Loading state — shows spinner */
  loading?: boolean;
  /** Icon before label */
  startIcon?: React.ReactNode;
  /** Icon after label */
  endIcon?: React.ReactNode;
  /** Full width */
  fullWidth?: boolean;
  /** HTML type */
  type?: 'button' | 'submit' | 'reset';
  /** Accessible label (for icon-only) */
  'aria-label'?: string;
}
```

```tsx
// design-system/components/Button/Button.tsx
// CURRENT: Fluent UI implementation

import { Button as FluentButton } from '@fluentui/react-components';
import type { DSButtonProps } from './Button.types';

const variantMap = {
  primary: 'primary',
  secondary: 'secondary',
  outline: 'outline',
  danger: 'primary',   // Fluent doesn't have danger — we add custom styling
  ghost: 'transparent',
} as const;

export function Button({
  variant, size = 'md', children, onClick, disabled,
  loading, startIcon, endIcon, fullWidth, type = 'button',
  'aria-label': ariaLabel,
}: DSButtonProps) {
  return (
    <FluentButton
      appearance={variantMap[variant]}
      size={size === 'sm' ? 'small' : size === 'lg' ? 'large' : 'medium'}
      onClick={onClick}
      disabled={disabled || loading}
      icon={loading ? <Spinner size="tiny" /> : startIcon}
      iconPosition="before"
      type={type}
      aria-label={ariaLabel}
      style={fullWidth ? { width: '100%' } : undefined}
    >
      {children}
      {endIcon && <span className="ds-button-end-icon">{endIcon}</span>}
    </FluentButton>
  );
}
```

```tsx
// FUTURE: Motif implementation (swap without changing consumers)

import { MotifButton } from '@motif/react';
import type { DSButtonProps } from './Button.types';

export function Button({
  variant, size = 'md', children, onClick, disabled,
  loading, startIcon, endIcon, fullWidth, type = 'button',
  'aria-label': ariaLabel,
}: DSButtonProps) {
  return (
    <MotifButton
      variant={variant}
      size={size}
      onClick={onClick}
      disabled={disabled}
      loading={loading}
      leftIcon={startIcon}
      rightIcon={endIcon}
      block={fullWidth}
      type={type}
      aria-label={ariaLabel}
    >
      {children}
    </MotifButton>
  );
}
```

**The consumers never change:**
```tsx
// features/payments/TransferButton.tsx — UNCHANGED during entire migration
import { Button } from '@/design-system';

function TransferButton({ onTransfer }: { onTransfer: () => void }) {
  return (
    <Button variant="primary" startIcon={<SendIcon />} onClick={onTransfer}>
      Transfer Funds
    </Button>
  );
}
```

---

## Q137. How do you manage design tokens during a migration?

**Answer:**

### Token Architecture (3 Tiers)

```
Primitive Tokens       → Raw values (colors, sizes)
     ↓
Semantic Tokens        → Intent-based (brand-primary, danger-foreground)
     ↓
Component Tokens       → Specific usage (button-primary-background)
```

### Token Mapping Strategy

```tsx
// design-system/tokens/mapping.ts
// Maps our semantic tokens to the CURRENT implementation (Fluent)

import { tokens as fluentTokens } from '@fluentui/react-components';

export const dsTokens = {
  // Colors — Semantic
  colorBrandPrimary: fluentTokens.colorBrandBackground,
  colorBrandPrimaryHover: fluentTokens.colorBrandBackgroundHover,
  colorDanger: fluentTokens.colorPaletteRedBackground3,
  colorSuccess: fluentTokens.colorPaletteGreenBackground3,
  colorNeutralBackground: fluentTokens.colorNeutralBackground1,
  colorNeutralForeground: fluentTokens.colorNeutralForeground1,
  colorNeutralBorder: fluentTokens.colorNeutralStroke1,

  // Spacing
  spacingXs: fluentTokens.spacingHorizontalXS,   // 4px
  spacingSm: fluentTokens.spacingHorizontalS,    // 8px
  spacingMd: fluentTokens.spacingHorizontalM,    // 12px
  spacingLg: fluentTokens.spacingHorizontalL,    // 16px
  spacingXl: fluentTokens.spacingHorizontalXL,   // 20px

  // Typography
  fontSizeBase: fluentTokens.fontSizeBase300,    // 14px
  fontSizeLarge: fluentTokens.fontSizeBase400,   // 16px
  fontSizeHeading: fluentTokens.fontSizeBase600, // 24px
  fontWeightRegular: fluentTokens.fontWeightRegular,
  fontWeightSemibold: fluentTokens.fontWeightSemibold,
  fontWeightBold: fluentTokens.fontWeightBold,

  // Borders
  borderRadiusSm: fluentTokens.borderRadiusSmall,
  borderRadiusMd: fluentTokens.borderRadiusMedium,
  borderRadiusLg: fluentTokens.borderRadiusLarge,

  // Shadows
  shadowSmall: fluentTokens.shadow2,
  shadowMedium: fluentTokens.shadow8,
  shadowLarge: fluentTokens.shadow16,
} as const;

// Type for consumers
export type DSTokens = typeof dsTokens;
export type DSTokenKey = keyof DSTokens;
```

### Token Migration — Swap Implementation

```tsx
// FUTURE: Same export, different source
import { motifTokens } from '@motif/tokens';

export const dsTokens = {
  colorBrandPrimary: motifTokens.brand.primary,
  colorBrandPrimaryHover: motifTokens.brand.primaryHover,
  colorDanger: motifTokens.feedback.error,
  colorSuccess: motifTokens.feedback.success,
  // ... same keys, different source values
} as const;
```

### CSS Custom Properties Approach (Recommended)

```tsx
// design-system/providers/ThemeProvider.tsx
import { dsTokens } from '../tokens';

export function ThemeProvider({ children }: { children: React.ReactNode }) {
  // Inject tokens as CSS custom properties
  const style = {
    '--ds-color-brand-primary': dsTokens.colorBrandPrimary,
    '--ds-color-danger': dsTokens.colorDanger,
    '--ds-spacing-md': dsTokens.spacingMd,
    // ...
  } as React.CSSProperties;

  return (
    <FluentProvider theme={customTheme}>
      <div style={style}>
        {children}
      </div>
    </FluentProvider>
  );
}

// Components use CSS variables — implementation-agnostic
// components/Card.module.css
.card {
  padding: var(--ds-spacing-md);
  background: var(--ds-color-neutral-background);
  border-radius: var(--ds-border-radius-md);
  box-shadow: var(--ds-shadow-small);
}
```

---

## Q138. How do you handle coexistence of two design systems during migration?

**Answer:**

### Challenge: Fluent + Motif running simultaneously

```tsx
// ThemeProvider handles both systems during transition
export function DualThemeProvider({ children }: { children: React.ReactNode }) {
  return (
    // Fluent provides its CSS vars for un-migrated components
    <FluentProvider theme={webLightTheme}>
      {/* Motif provides its CSS vars for migrated components */}
      <MotifProvider theme={motifTheme}>
        {/* Our abstraction layer resolves conflicts */}
        <DSTokenProvider>
          {children}
        </DSTokenProvider>
      </MotifProvider>
    </FluentProvider>
  );
}
```

### CSS Isolation Strategy

```css
/* Scope Fluent styles to un-migrated areas */
[data-ds-version="fluent"] .fui-Button { /* Fluent styles */ }

/* Scope Motif styles to migrated areas */
[data-ds-version="motif"] .motif-button { /* Motif styles */ }

/* Prevent cascade conflicts */
.ds-component-boundary {
  /* Reset inherited styles at component boundaries */
  all: revert-layer;
}
```

### Feature Flag Integration

```tsx
// Feature flag determines which implementation to use
import { useFeatureFlag } from '@/utils/featureFlags';

// design-system/components/Button/index.ts
export function Button(props: DSButtonProps) {
  const useMotif = useFeatureFlag('ds-motif-button');

  if (useMotif) {
    return <MotifButtonImpl {...props} />;
  }
  return <FluentButtonImpl {...props} />;
}

// This allows:
// 1. A/B testing new components with subset of users
// 2. Instant rollback if Motif Button has issues
// 3. Gradual rollout: 10% → 50% → 100%
```

---

## Q139. What is your component migration order strategy?

**Answer:**

### Migration Order: Leaf Components First, Composites Last

```
PHASE 1: Primitives (lowest risk, highest reuse)
──────────────────────────────────────────────────
Button, IconButton, Link, Badge, Avatar, Spinner,
Divider, Text, Label, Tag

PHASE 2: Form Controls (medium complexity)
──────────────────────────────────────────────────
Input, Textarea, Select, Checkbox, Radio, Switch,
DatePicker, TimePicker, Slider, FileUpload

PHASE 3: Feedback & Overlay (behavioral complexity)
──────────────────────────────────────────────────
Dialog, Toast, Tooltip, Popover, Menu, Drawer,
MessageBar, ProgressBar, Alert

PHASE 4: Navigation (app-wide impact)
──────────────────────────────────────────────────
Tabs, Breadcrumb, Sidebar, Nav, Pagination

PHASE 5: Data Display (highest complexity)
──────────────────────────────────────────────────
Table, DataGrid, Tree, Accordion, Card, List

PHASE 6: Layout & Composition
──────────────────────────────────────────────────
Page layouts, Grid systems, responsive containers
```

### Why This Order?

| Principle | Explanation |
|-----------|-------------|
| **Leaf first** | Buttons/Inputs have no children — safest to swap |
| **High-usage first** | Button appears 200+ times — migration has biggest visual impact |
| **Low-risk first** | Primitives rarely have complex state or side effects |
| **Behavioral complexity last** | DataGrid has sorting, pagination, virtualization — needs most testing |
| **Dependencies flow up** | Migrate `Button` before `Dialog` (Dialog uses Button internally) |

### Tracking Progress

```tsx
// design-system/migration-status.ts — Living documentation
export const MIGRATION_STATUS = {
  // Phase 1 — COMPLETE
  Button: 'motif',
  IconButton: 'motif',
  Badge: 'motif',
  Spinner: 'motif',

  // Phase 2 — IN PROGRESS
  Input: 'motif',
  Textarea: 'motif',
  Select: 'fluent',    // ← Next to migrate
  Checkbox: 'fluent',
  DatePicker: 'fluent',

  // Phase 3 — NOT STARTED
  Dialog: 'fluent',
  Toast: 'fluent',
  Menu: 'fluent',
} as const;

type MigrationStatus = typeof MIGRATION_STATUS;
type ComponentName = keyof MigrationStatus;
type MigratedComponents = {
  [K in ComponentName]: MigrationStatus[K] extends 'motif' ? K : never
}[ComponentName];
```

---

## Q140. How do you ensure visual regression safety during migration?

**Answer:**

### Visual Regression Testing Pipeline

```
Developer makes change → PR created → CI runs:
  1. Unit tests (jest)
  2. Component tests (Testing Library)
  3. Visual regression (Chromatic/Playwright screenshots)
  4. Accessibility audit (axe-core)
  5. Bundle size check
```

### Playwright Visual Comparison

```tsx
// tests/visual/button.visual.spec.ts
import { test, expect } from '@playwright/test';

const variants = ['primary', 'secondary', 'outline', 'danger', 'ghost'] as const;
const sizes = ['sm', 'md', 'lg'] as const;
const states = ['default', 'hover', 'focus', 'disabled', 'loading'] as const;

// Generate screenshots for ALL combinations
for (const variant of variants) {
  for (const size of sizes) {
    test(`Button ${variant} ${size} renders correctly`, async ({ page }) => {
      await page.goto(`/storybook/button--${variant}-${size}`);
      await expect(page.locator('.ds-button')).toHaveScreenshot(
        `button-${variant}-${size}.png`,
        { maxDiffPixelRatio: 0.01 }  // Allow 1% pixel difference
      );
    });
  }
}

// State testing
test('Button states match baseline', async ({ page }) => {
  await page.goto('/storybook/button--all-states');

  // Hover state
  await page.locator('[data-testid="button-primary"]').hover();
  await expect(page.locator('[data-testid="button-primary"]')).toHaveScreenshot('button-hover.png');

  // Focus state
  await page.locator('[data-testid="button-primary"]').focus();
  await expect(page.locator('[data-testid="button-primary"]')).toHaveScreenshot('button-focus.png');
});
```

### Before/After Migration Comparison

```tsx
// tests/migration/button-parity.test.tsx
import { render } from '@testing-library/react';
import { Button as FluentImpl } from '../components/Button/FluentButton';
import { Button as MotifImpl } from '../components/Button/MotifButton';

describe('Button migration parity', () => {
  const testCases: DSButtonProps[] = [
    { variant: 'primary', children: 'Click me' },
    { variant: 'secondary', size: 'lg', children: 'Large' },
    { variant: 'danger', disabled: true, children: 'Disabled' },
    { variant: 'primary', loading: true, children: 'Loading' },
    { variant: 'primary', startIcon: <PlusIcon />, children: 'Add' },
  ];

  testCases.forEach((props, i) => {
    it(`case ${i}: same accessible role and name`, () => {
      const { getByRole: getOld } = render(<FluentImpl {...props} />);
      const { getByRole: getNew } = render(<MotifImpl {...props} />);

      const oldButton = getOld('button');
      const newButton = getNew('button');

      // Behavioral parity checks
      expect(newButton).toHaveAccessibleName(oldButton.getAttribute('aria-label') || props.children as string);
      expect(newButton.getAttribute('disabled')).toBe(oldButton.getAttribute('disabled'));
      expect(newButton.getAttribute('type')).toBe(oldButton.getAttribute('type'));
    });
  });
});
```

---

## Q141. How do you handle migration in a team with ongoing feature development?

**Answer:**

### The Golden Rule: Migration Never Blocks Feature Work

```
Feature developers → Import from design-system/ (abstraction)
                     They DON'T know or care if it's Fluent or Motif underneath

Migration team    → Works inside design-system/ internals
                     Swaps implementations behind the abstraction
```

### Branch Strategy

```
main
├── feature/add-payment-flow     ← Uses Button from design-system/ (doesn't care about impl)
├── feature/account-details      ← Same
├── migration/button-to-motif    ← Changes design-system/Button internals
└── migration/input-to-motif     ← Changes design-system/Input internals
```

### Communication Plan

```markdown
## Weekly Migration Update (shared with team)

### Migrated This Week:
- ✅ Button (all variants) — now using Motif
- ✅ Badge — now using Motif

### In Progress:
- 🔄 Input — 80% done, blocked on date picker integration

### Coming Next Week:
- Select
- Checkbox
- Radio

### Action Required:
- None — migration is transparent via abstraction layer

### Known Issues:
- Button focus ring is 1px thicker in Motif — design approved ✅
- Custom `className` prop on Badge needs manual review in 3 places
```

### PR Standards for Migration PRs

```markdown
## Migration PR Template

### Component: [Button]
### Direction: Fluent → Motif

### Checklist:
- [ ] Public API (DSButtonProps) is UNCHANGED
- [ ] All existing unit tests pass without modification
- [ ] Visual regression screenshots reviewed and approved
- [ ] Accessibility audit passes (jest-axe)
- [ ] Storybook stories render correctly
- [ ] No bundle size regression (±5KB tolerance)
- [ ] Feature flag added for gradual rollout
- [ ] Rollback plan documented
```

---

## Q142. How do you handle Fluent UI-specific patterns that don't exist in the target system?

**Answer:**

### Common Fluent-Specific Patterns to Address:

**1. Slot Pattern (Fluent v9 unique concept)**
```tsx
// Fluent UI uses "slots" for sub-component customization
<Button icon={{ children: <CalendarIcon />, className: 'custom' }}>
  Schedule
</Button>

// Your abstraction normalizes this to a simpler API:
<Button startIcon={<CalendarIcon />}>Schedule</Button>
```

**2. Griffel Styles (Fluent's CSS-in-JS)**
```tsx
// Fluent-specific: makeStyles with Griffel
import { makeStyles, tokens } from '@fluentui/react-components';
const useStyles = makeStyles({
  root: { color: tokens.colorBrandForeground1 }
});

// Your abstraction: Use CSS custom properties instead
// This is implementation-agnostic
.component {
  color: var(--ds-color-brand-primary);
}
```

**3. Fluent's Compound Components Pattern**
```tsx
// Fluent DataGrid uses compound components
<DataGrid>
  <DataGridHeader>
    <DataGridRow>
      <DataGridHeaderCell>Name</DataGridHeaderCell>
    </DataGridRow>
  </DataGridHeader>
  <DataGridBody>
    <DataGridRow>
      <DataGridCell>John</DataGridCell>
    </DataGridRow>
  </DataGridBody>
</DataGrid>

// Your abstraction: Config-driven API (easier to swap implementations)
<DSTable
  columns={[{ key: 'name', header: 'Name', render: (row) => row.name }]}
  data={users}
  sortable
  paginated
/>
```

**4. Fluent's useArrowNavigationGroup (keyboard nav helper)**
```tsx
// Fluent provides keyboard navigation utilities
import { useArrowNavigationGroup } from '@fluentui/react-components';

// Your abstraction: Implement framework-agnostic keyboard hook
import { useKeyboardNavigation } from '@/design-system/hooks';
```

---

## Q143. What metrics do you track during a design system migration?

**Answer:**

| Metric | Target | Tool |
|--------|--------|------|
| **Migration progress** | % components migrated | Migration status map |
| **Bundle size** | No increase >10KB per component | Bundlewatch / size-limit |
| **Visual regression** | 0 unintended changes | Chromatic / Playwright |
| **Accessibility score** | 0 new violations | jest-axe / Lighthouse |
| **Performance** | No LCP/CLS regression | Web Vitals monitoring |
| **Developer satisfaction** | Survey score ≥4/5 | Quarterly survey |
| **Adoption rate** | New features use DS components | ESLint rule + PR review |
| **Bug rate** | Migration bugs < 2/sprint | Jira tracking |

### Dashboard Example:

```
┌─────────────────────────────────────────────────────┐
│          Design System Migration Dashboard           │
├─────────────────────────────────────────────────────┤
│                                                      │
│  Overall Progress: ████████░░░░░░░░ 52% (26/50)     │
│                                                      │
│  Phase 1 (Primitives):   ████████████████ 100%       │
│  Phase 2 (Form Controls): ████████░░░░░░ 60%         │
│  Phase 3 (Feedback):      ████░░░░░░░░░░ 25%         │
│  Phase 4 (Navigation):    ░░░░░░░░░░░░░░ 0%          │
│  Phase 5 (Data Display):  ░░░░░░░░░░░░░░ 0%          │
│                                                      │
│  Bundle Size Delta: +3.2KB (within budget)           │
│  Visual Regressions: 0 this sprint                   │
│  A11y Violations: 0 new                              │
│  Rollbacks: 1 (Toast — fixed in next sprint)         │
│                                                      │
└─────────────────────────────────────────────────────┘
```

---

## Q144. How would you pitch this migration strategy to stakeholders?

**Answer:**

### The Business Case

```markdown
## Why Migrate (Business Value)

1. **Reduced development time** — Motif components align better with our design specs,
   reducing designer-developer handoff friction by ~30%

2. **Smaller bundle** — Motif is tree-shakeable and lighter than full Fluent UI,
   improving load times (impacts user engagement metrics)

3. **Design consistency** — Single source of truth eliminates
   "which component do I use?" confusion across 10+ feature teams

4. **Talent retention** — Modern design system attracts better frontend talent

5. **Reduced maintenance** — No more fighting Fluent's opinionated patterns
   when our designs diverge from Microsoft's vision
```

### Risk Mitigation Answers

| Stakeholder Concern | Your Answer |
|----|-----|
| "Will this break things?" | Abstraction layer means zero breaking changes for feature teams |
| "How long will it take?" | Phased approach — usable value from Phase 1 (4 weeks) |
| "Will it slow feature work?" | No — migration is transparent; feature PRs don't touch migration |
| "What if Motif doesn't work?" | Feature flags enable instant per-component rollback |
| "What if the team can't keep up?" | Migration is incremental — we can pause at any phase boundary |

---


## Q145. What's your rollback strategy if a migrated component causes issues?

**Answer:**

```tsx
// Three levels of rollback, from fastest to most complete:

// Level 1: Feature Flag (instant, no deploy needed)
// Toggle in LaunchDarkly / Azure App Config → immediately reverts to Fluent
// Recovery time: < 1 minute

// Level 2: Code Rollback (per-component)
// design-system/components/Button/index.ts
export { Button } from './FluentButton';  // ← Just change this import
// export { Button } from './MotifButton';  // Commented out
// Recovery time: 5-minute PR → merge → deploy

// Level 3: Full Branch Revert (nuclear option)
// git revert <migration-commit-hash>
// Recovery time: ~30 minutes

// Our DEFAULT strategy: Feature Flags (Level 1)
// - Every migration starts at 0% rollout
// - 10% → monitor for 24h
// - 50% → monitor for 48h
// - 100% → stable for 1 week → remove flag
```

---

## Key Interview Lines for Design System Migration

1. "I use the Strangler Fig pattern — an abstraction layer that lets me swap implementations without changing any consuming code."

2. "Migration never blocks feature development. Feature teams import from our design system package; they don't know or care what's underneath."

3. "We migrate bottom-up: primitives first (Button, Input), composites last (DataGrid, Dialog). Dependencies always flow upward."

4. "Every migration PR requires: unchanged public API, passing visual regression, passing a11y audit, and a feature flag for gradual rollout."

5. "Token mapping is the foundation. If semantic tokens are properly mapped, the visual transition is smooth. The harder part is behavioral parity — focus management, keyboard navigation, ARIA patterns."

6. "I track progress with a migration dashboard — percentage complete, bundle size delta, visual regressions, and a11y violations. Zero tolerance for new a11y issues."

