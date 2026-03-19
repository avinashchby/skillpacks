---
name: builder-component-gen
description: >
  Generates UI components from a plain-language description. Triggered when the user says
  "create component", "build component", "generate component", "new component", "make a
  [ComponentName] component", or describes a UI element and asks for it to be built. Detects
  the frontend framework (React, Vue 3, Svelte 5) and language (TypeScript or JavaScript)
  from the project context, then produces the component file, TypeScript prop types or an
  interface, a Storybook story, and a unit test — all following the conventions already
  present in the codebase.
version: 1.0.0
---

# Component Generator

## When to Use

Activate this skill when the user:

- Says "create component [name]", "generate component for [description]", "build component",
  "new component called [Name]", or "make a [Name] component"
- Describes a UI element (button, modal, form field, data table, card, etc.) and asks for it
  to be built
- Asks to "add a [Name] to the design system" or "add [Name] to the component library"
- Wants a component with specific props, variants, or state

Do NOT activate for:
- Full page or view scaffolding (that combines many components — propose decomposition first)
- Generating a complete form with server submission (use builder-api-scaffolder for the API side)
- Styling-only changes to an existing component

---

## Instructions

### Step 1 — Detect Framework and Conventions

Scan the project before writing any code:

1. **Framework**: Look for `react` in `package.json` dependencies (React), `vue` (Vue 3),
   or `svelte` (Svelte 5). Check for `next.config.*`, `vite.config.*`, `nuxt.config.*`,
   or `svelte.config.*` for additional context.

2. **Language**: `.tsx`/`.ts` → TypeScript. `.jsx`/`.js` → JavaScript. If TypeScript is
   present, always generate TypeScript. Never downgrade to JS if TS is in use.

3. **Component directory**: Check for `src/components/`, `components/`, `src/ui/`, or
   `lib/components/`. Match the depth and nesting of existing components.

4. **File naming convention**: Check existing component files — PascalCase (`Button.tsx`),
   kebab-case (`button.tsx`), or directory-per-component (`Button/index.tsx`). Match exactly.

5. **Styling approach**: Detect `tailwindcss` in `package.json`, CSS Modules (`.module.css`
   files), `styled-components`, or plain CSS. Match the existing approach.

6. **Storybook**: Check for `.storybook/` directory. If present, generate a story.

7. **Test runner**: `jest` + `@testing-library/react`, `vitest`, `@testing-library/vue`,
   or `@testing-library/svelte`. If present, generate a test.

8. **Existing component patterns**: Read 1–2 existing components to understand:
   - How props are defined (interface, type alias, inline)
   - Whether `forwardRef` is used
   - How className/style overrides are handled
   - Whether a `cn()` or `clsx()` utility is used for conditional classes

If any of the above cannot be determined, ask before generating.

### Step 2 — Parse the Component Description

Extract from the user's description:

- **Component name**: PascalCase noun or noun phrase
- **Props**: name, type, required vs optional, default values, constraints (min, max, enum)
- **Variants**: sizes, color themes, states (loading, disabled, error)
- **Children / slots**: whether the component accepts children or named slots
- **Events / callbacks**: `onClick`, `onChange`, `onSubmit`, etc.
- **Accessibility requirements**: role, aria-label needs, keyboard navigation
- **Internal state**: controlled vs uncontrolled, any local state (open/closed, value, etc.)

If a prop type is ambiguous, use `string` and add a `// TODO: confirm type` comment.

### Step 3 — Generate Files

#### 3a. Component File

Structure the component following the conventions detected in Step 1.

**React (TypeScript) rules:**
- Define a `Props` interface (exported) before the component function.
- Use named exports, not default exports, unless the codebase uses defaults consistently.
- Destructure props in the function signature.
- Use `React.forwardRef` if the component wraps a native element (`button`, `input`, etc.).
- Add `displayName` when using `forwardRef`.
- Keep the component function under 50 lines; extract sub-components or hooks if needed.

**Vue 3 (TypeScript) rules:**
- Use `<script setup lang="ts">` composition API syntax.
- Define props with `defineProps<Props>()` and emits with `defineEmits<Emits>()`.
- Export the Props interface from a separate types file if it is reused elsewhere.

**Svelte 5 rules:**
- Use runes: `$props()`, `$state()`, `$derived()`, `$effect()`.
- Define a `Props` interface and use `let { ... }: Props = $props()`.
- Export the component as the default export of the `.svelte` file.

#### 3b. Types File (if Props are complex)

If the component has more than 4 props or shared types, extract them to a
`[ComponentName].types.ts` file. Otherwise, define inline.

#### 3c. Storybook Story

Generate a `[ComponentName].stories.tsx` (or `.svelte`, `.vue`) file with:
- A `meta` export with `title`, `component`, `argTypes` derived from the Props
- A `Default` story showing the base state
- Additional named stories for each major variant (e.g., `Loading`, `Disabled`, `Error`)
- `args` that mirror realistic prop values

#### 3d. Unit Test

Generate a `[ComponentName].test.tsx` (or framework equivalent) with:
- Render test: component mounts without error
- Props test: key props are reflected in the DOM
- Interaction test: any `onClick`, `onChange`, etc. fires the callback
- Accessibility test: check `role`, `aria-*` attributes where applicable
- One negative test: component handles missing optional props gracefully

### Step 4 — Output Format

1. Print the file tree for the new component.
2. Print each file in a labeled fenced code block.
3. Add a **"Usage example"** showing how to import and use the component in a parent.
4. Note any dependencies that need to be installed.

---

## Examples

### Example 1 — React + TypeScript + Tailwind: Badge Component

**User input:**
> create component Badge: shows a label string, supports variants: default/success/warning/error,
> optional size sm/md, accepts a className override

**File tree:**
```
src/components/Badge/
  Badge.tsx
  Badge.stories.tsx
  Badge.test.tsx
```

`Badge.tsx`:
```tsx
import { clsx } from 'clsx';

export interface BadgeProps {
  label: string;
  variant?: 'default' | 'success' | 'warning' | 'error';
  size?: 'sm' | 'md';
  className?: string;
}

const variantClasses: Record<NonNullable<BadgeProps['variant']>, string> = {
  default: 'bg-gray-100 text-gray-800',
  success: 'bg-green-100 text-green-800',
  warning: 'bg-yellow-100 text-yellow-800',
  error: 'bg-red-100 text-red-800',
};

const sizeClasses: Record<NonNullable<BadgeProps['size']>, string> = {
  sm: 'px-2 py-0.5 text-xs',
  md: 'px-3 py-1 text-sm',
};

export function Badge({
  label,
  variant = 'default',
  size = 'md',
  className,
}: BadgeProps) {
  return (
    <span
      className={clsx(
        'inline-flex items-center rounded-full font-medium',
        variantClasses[variant],
        sizeClasses[size],
        className,
      )}
    >
      {label}
    </span>
  );
}
```

`Badge.test.tsx`:
```tsx
import { render, screen } from '@testing-library/react';
import { Badge } from './Badge';

describe('Badge', () => {
  it('renders the label', () => {
    render(<Badge label="Active" />);
    expect(screen.getByText('Active')).toBeInTheDocument();
  });

  it('applies the success variant class', () => {
    render(<Badge label="OK" variant="success" />);
    expect(screen.getByText('OK')).toHaveClass('bg-green-100');
  });

  it('renders with sm size', () => {
    render(<Badge label="small" size="sm" />);
    expect(screen.getByText('small')).toHaveClass('text-xs');
  });

  it('accepts a className override', () => {
    render(<Badge label="custom" className="border border-gray-300" />);
    expect(screen.getByText('custom')).toHaveClass('border-gray-300');
  });
});
```

**Usage:**
```tsx
import { Badge } from '@/components/Badge/Badge';

<Badge label="Published" variant="success" size="sm" />
```

---

### Example 2 — Vue 3 + TypeScript: ConfirmDialog

**User input:**
> new component ConfirmDialog with props: title, message, confirmLabel, cancelLabel (both
> optional with defaults), emits: confirm and cancel events

**`ConfirmDialog.vue`** (excerpt):
```vue
<script setup lang="ts">
interface Props {
  title: string;
  message: string;
  confirmLabel?: string;
  cancelLabel?: string;
}

interface Emits {
  (e: 'confirm'): void;
  (e: 'cancel'): void;
}

const { confirmLabel = 'Confirm', cancelLabel = 'Cancel' } = defineProps<Props>();
const emit = defineEmits<Emits>();
</script>

<template>
  <div role="dialog" aria-modal="true" :aria-labelledby="'dialog-title'">
    <h2 id="dialog-title">{{ title }}</h2>
    <p>{{ message }}</p>
    <div>
      <button @click="emit('cancel')">{{ cancelLabel }}</button>
      <button @click="emit('confirm')">{{ confirmLabel }}</button>
    </div>
  </div>
</template>
```

---

### Example 3 — Svelte 5: CounterButton

**User input:**
> generate component CounterButton: displays a count, increments on click, accepts
> initialCount prop (number, default 0), emits change event with new count

**`CounterButton.svelte`:**
```svelte
<script lang="ts">
  interface Props {
    initialCount?: number;
    onchange?: (count: number) => void;
  }

  let { initialCount = 0, onchange }: Props = $props();
  let count = $state(initialCount);

  function increment() {
    count += 1;
    onchange?.(count);
  }
</script>

<button onclick={increment} aria-label="Increment counter">
  Count: {count}
</button>
```

---

## Anti-patterns

1. **Generating default exports when the codebase uses named exports** — This breaks
   tree-shaking and makes imports inconsistent. Always read an existing component first
   to match the export style.

2. **Hardcoding colors or spacing instead of using the project's token system** — If the
   project has a Tailwind theme or CSS custom properties, always use them. Never introduce
   new raw hex values or `px` numbers that don't exist in the design system.

3. **Creating components longer than 80 lines without decomposing** — A component that
   renders a complex list item, handles all its own state, and manages a popover should be
   split into at least two components. Propose the decomposition to the user before writing.

4. **Skipping `aria-*` attributes on interactive elements** — Buttons without accessible
   labels, modals without `role="dialog"`, and inputs without associated `<label>` elements
   are accessibility failures. Every interactive component must include the minimum required
   ARIA attributes.

5. **Writing tests that only test implementation details** — Tests that check internal state
   variable names or assert on component internals (vs. what the user sees or interacts with)
   are brittle. Use Testing Library queries (`getByRole`, `getByText`) and simulate real
   user interactions (`userEvent.click`).
