# Reusable React Form Architecture with `useReducer` (SOLID + Practical Snippets)

## Table of Contents

- [Reusable React Form Architecture with `useReducer` (SOLID + Practical Snippets)](#reusable-react-form-architecture-with-usereducer-solid--practical-snippets)
  - [Table of Contents](#table-of-contents)
  - [Why this architecture instead of one big form component?](#why-this-architecture-instead-of-one-big-form-component)
  - [1) `useFormReducer.js` — Form Brain](#1-useformreducerjs--form-brain)
    - [Responsibilities](#responsibilities)
    - [Snippet: Actions and initial state](#snippet-actions-and-initial-state)
    - [Snippet: `handleChange` (update + validate)](#snippet-handlechange-update--validate)
    - [Snippet: `handleSubmit` (validate + async submit)](#snippet-handlesubmit-validate--async-submit)
  - [2) `ReusableForm.js` — Generic Form UI Shell](#2-reusableformjs--generic-form-ui-shell)
    - [Responsibilities](#responsibilities-1)
    - [Snippet: Dynamic field rendering](#snippet-dynamic-field-rendering)
    - [Snippet: Submit and reset buttons](#snippet-submit-and-reset-buttons)
    - [Snippet: Accessible input wiring](#snippet-accessible-input-wiring)
  - [3) `ContactUs.js` — Feature-Level Configuration](#3-contactusjs--feature-level-configuration)
    - [Responsibilities](#responsibilities-2)
    - [Snippet: Field schema](#snippet-field-schema)
    - [Snippet: Validators](#snippet-validators)
    - [Snippet: Page composition](#snippet-page-composition)
  - [SOLID Mapping in this Implementation](#solid-mapping-in-this-implementation)
  - [End-to-end flow summary](#end-to-end-flow-summary)
  - [Future enhancement ideas](#future-enhancement-ideas)

This document explains how the Contact form system is structured using three layers:

1. `utils/useFormReducer.js` — form logic/state engine  
2. `component/ReusableForm.js` — generic UI renderer  
3. `component/ContactUs.js` — feature-specific configuration

This separation keeps the implementation reusable, testable, and aligned with SOLID principles.

---

## Why this architecture instead of one big form component?

In a single form component, state updates, validation, rendering, and submit logic are mixed together.
That works for small forms, but becomes hard to maintain and reuse.

Here, responsibilities are split:

- `useFormReducer` = behavior and state transitions
- `ReusableForm` = rendering and accessibility wiring
- `ContactUs` = business rules and form configuration

---

## 1) `useFormReducer.js` — Form Brain

### Responsibilities

- Manage `values`, `errors`, `touched`
- Track `isSubmitting`, `isSubmitted`, `submitError`
- Handle field updates and blur events
- Validate fields and full form
- Support submit and reset workflows

### Snippet: Actions and initial state

```javascript
const ACTIONS = {
  CHANGE_FIELD: 'CHANGE_FIELD',
  BLUR_FIELD: 'BLUR_FIELD',
  SET_ERRORS: 'SET_ERRORS',
  SUBMIT_START: 'SUBMIT_START',
  SUBMIT_SUCCESS: 'SUBMIT_SUCCESS',
  SUBMIT_ERROR: 'SUBMIT_ERROR',
  RESET_FORM: 'RESET_FORM',
};

const createInitialState = (initialValues) => ({
  values: initialValues,
  touched: {},
  errors: {},
  isSubmitting: false,
  isSubmitted: false,
  submitError: '',
});
```

### Snippet: `handleChange` (update + validate)

```javascript
const handleChange = (event) => {
  const { name, value } = event.target;

  dispatch({
    type: ACTIONS.CHANGE_FIELD,
    payload: { name, value },
  });

  const nextValues = {
    ...state.values,
    [name]: value,
  };

  const nextErrors = {
    ...state.errors,
    [name]: runFieldValidator(name, value, nextValues, validators),
  };

  if (!nextErrors[name]) delete nextErrors[name];

  dispatch({
    type: ACTIONS.SET_ERRORS,
    payload: nextErrors,
  });
};
```

### Snippet: `handleSubmit` (validate + async submit)

```javascript
const handleSubmit = async (event) => {
  event.preventDefault();

  const nextErrors = createErrors(state.values, validators);
  dispatch({ type: ACTIONS.SET_ERRORS, payload: nextErrors });

  if (Object.keys(nextErrors).length > 0) return;

  dispatch({ type: ACTIONS.SUBMIT_START });

  try {
    await onSubmit?.(state.values, { resetForm });
    dispatch({ type: ACTIONS.SUBMIT_SUCCESS });
  } catch (error) {
    dispatch({
      type: ACTIONS.SUBMIT_ERROR,
      payload: error?.message || 'Something went wrong while submitting.',
    });
  }
};
```

---

## 2) `ReusableForm.js` — Generic Form UI Shell

### Responsibilities

- Render form fields dynamically from config
- Wire events from inputs to reducer handlers
- Show validation and submit feedback
- Provide reusable submit/reset UI
- Keep accessibility defaults (`aria-*`)

### Snippet: Dynamic field rendering

```javascript
{fields.map((field) => (
  <FormField
    key={field.name}
    field={field}
    value={values[field.name] || ''}
    error={errors[field.name]}
    touched={touched[field.name]}
    onChange={handleChange}
    onBlur={handleBlur}
  />
))}
```

### Snippet: Submit and reset buttons

```javascript
<button type='submit' disabled={isSubmitDisabled} aria-disabled={isSubmitDisabled}>
  {isSubmitting ? 'Submitting...' : submitLabel}
</button>
<button type='button' onClick={() => resetForm()} disabled={isSubmitting}>
  {resetLabel}
</button>
```

### Snippet: Accessible input wiring

```javascript
<input
  id={field.name}
  name={field.name}
  type={field.type || 'text'}
  required={field.required}
  aria-required={field.required ? 'true' : 'false'}
  aria-invalid={error ? 'true' : 'false'}
  aria-describedby={error ? `${field.name}-error` : undefined}
  value={value}
  onChange={onChange}
  onBlur={onBlur}
/>
```

---

## 3) `ContactUs.js` — Feature-Level Configuration

### Responsibilities

- Define Contact-specific field schema
- Define initial values
- Define validation rules
- Provide submit behavior
- Compose page with `ReusableForm`

### Snippet: Field schema

```javascript
const fields = [
  {
    name: 'fullName',
    label: 'Full Name',
    required: true,
    placeholder: 'Enter your full name',
    autoComplete: 'name',
  },
  {
    name: 'email',
    label: 'Email',
    type: 'email',
    required: true,
    placeholder: 'Enter your email',
    autoComplete: 'email',
  },
  {
    name: 'message',
    label: 'Message',
    required: true,
    placeholder: 'How can we help you?',
  },
];
```

### Snippet: Validators

```javascript
const validators = {
  fullName: (value) => {
    if (!value.trim()) return 'Name is required.';
    if (value.trim().length < 2) return 'Name should be at least 2 characters.';
    return '';
  },
  email: (value) => {
    if (!value.trim()) return 'Email is required.';
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    return emailRegex.test(value) ? '' : 'Please enter a valid email address.';
  },
  message: (value) => {
    if (!value.trim()) return 'Message is required.';
    if (value.trim().length < 10) return 'Message should be at least 10 characters.';
    return '';
  },
};
```

### Snippet: Page composition

```javascript
<ReusableForm
  title='Contact Information'
  fields={fields}
  initialValues={initialValues}
  validators={validators}
  onSubmit={handleContactSubmit}
  submitLabel='Send Message'
  resetLabel='Reset Form'
  successMessage='Thanks! We received your message.'
/>
```

---

## SOLID Mapping in this Implementation

- **Single Responsibility (S):**  
  `useFormReducer` handles logic, `ReusableForm` handles rendering, `ContactUs` handles business config.

- **Open/Closed (O):**  
  Add a new form by changing configuration (`fields`, `validators`, `onSubmit`) without changing core engine.

- **Liskov Substitution (L):**  
  Any page can substitute its own field schema while keeping the same `ReusableForm` contract.

- **Interface Segregation (I):**  
  Consumers pass only focused props they need.

- **Dependency Inversion (D):**  
  The generic form system depends on abstractions (`validators`, callback `onSubmit`) rather than page-specific details.

---

## End-to-end flow summary

1. User updates a field (`handleChange`)  
2. Reducer updates state and error map  
3. On blur, touched status + validation updates  
4. On submit, full-form validation runs  
5. If valid, async submit callback executes  
6. UI shows success/error state  
7. Reset restores initial values and clears metadata

---

## Future enhancement ideas

- Support textarea/select/checkbox in `FormField`
- Add optional `onReset` callback
- Add unit tests for reducer actions and validators
- Integrate schema validators (Zod/Yup) for larger forms
- Add field-level async validation hooks

---

This architecture gives you a reusable base for Login, Signup, Feedback, Profile, and Contact forms with minimal duplication.

