---
name: compound-pattern
description: >
  Expert guide for designing and implementing the Compound Component pattern in React.
  Use this skill whenever the user wants to build reusable UI components that share state
  internally (menus, dropdowns, tabs, accordions, modals, data tables, lists), when they
  ask how to avoid prop drilling or "God component" anti-patterns, when they want to build
  a component library with a clean declarative API, or when they mention compound components,
  Context API for component composition, React.Children, or cloneElement. Trigger even if
  the user just describes a component with sub-parts that need to communicate (e.g. "tabs
  with active state", "dropdown with a trigger and list", "table with rows and cells").
---

# Compound Component Pattern

Compound components expose a **parent + a set of child components** that work together
through shared internal state. The consumer controls the structure declaratively — exactly
like HTML's `<table>` / `<thead>` / `<tbody>` / `<tr>` / `<td>`, where the browser uses
the structure you declare to construct a well-behaved table — while the components
themselves handle the behavior.

> Sources: [jjenzz.com/compound-components](https://jjenzz.com/compound-components/) ·
> [patterns.dev/react/compound-pattern](https://www.patterns.dev/react/compound-pattern/)

***

## Why choose compound components?

The alternative — a single "God component" driven by config props — has real costs:

| Problem with God components | Compound solution |
|---|---|
| Consumer must transform data into the format the component expects | Consumer renders their data however they want — no transformation needed |
| Every new feature needs a new prop and a new release | Consumer binds directly to the sub-component they need |
| Props proliferate with prefixes: `rowClassName`, `cellClassName`, `onRowClick`, `onCellClick`… | Each sub-component accepts its own standard HTML/React props |
| Hard to visualize what renders by reading the JSX | The tree IS the output — what you write is what you get |

> "If you find yourself repeating a prefix amongst your props, try converting that prefix
> into a child component. It will give more control to the consumer and mean less
> maintenance for you in the long run." — jjenzz.com

Example contrast:

```tsx
// ❌ God component — must transform data, prop explosion
<Table
  caption="Cats"
  columns={columns}
  rowData={cats}
  rowClassName="table-row"
  cellClassName="table-cell"
  onRowClick={handleRowClick}
/>

// ✅ Compound — consumer places and styles each part directly
<Table>
  <TableCaption>Cats</TableCaption>
  <TableHead>
    <TableRow>
      <TableCell>Name</TableCell>
      <TableCell>Breed</TableCell>
    </TableRow>
  </TableHead>
  <TableBody>
    {cats.map(cat => (
      <TableRow key={cat.id} className="table-row" onClick={handleRowClick}>
        <TableCell className="table-cell">{cat.name}</TableCell>
        <TableCell className="table-cell">{cat.breed}</TableCell>
      </TableRow>
    ))}
  </TableBody>
</Table>
```

Need an `onCellClick` for just one cell? With compound components the consumer binds to
that cell directly — no need to release a new version with a new prop.

***

## Two implementation approaches

### 1. Context API (preferred)

Shares state at any depth — child components don't need to be direct children of the
parent. Internal state stays private; consumers can't accidentally break consistency.

```tsx
import { createContext, useContext, useState, useMemo } from 'react';

// --- Internal context (NOT exported) ---
const FlyOutContext = createContext<{ open: boolean; toggle: () => void } | null>(null);

function useFlyOut() {
  const ctx = useContext(FlyOutContext);
  if (!ctx) throw new Error('FlyOut sub-components must be used inside <FlyOut>');
  return ctx;
}

// --- Parent: owns the state ---
function FlyOut({ children }: { children: React.ReactNode }) {
  const [open, setOpen] = useState(false);
  const value = useMemo(
    () => ({ open, toggle: () => setOpen(o => !o) }),
    [open]
  );
  return (
    <FlyOutContext.Provider value={value}>
      {children}
    </FlyOutContext.Provider>
  );
}

// --- Sub-components: consume shared state ---
function Toggle() {
  const { toggle } = useFlyOut();
  return <button onClick={toggle}>☰</button>;
}

function List({ children }: { children: React.ReactNode }) {
  const { open } = useFlyOut();
  return open ? <ul>{children}</ul> : null;
}

function Item({ children }: { children: React.ReactNode }) {
  return <li>{children}</li>;
}

// --- Attach sub-components as static properties ---
FlyOut.Toggle = Toggle;
FlyOut.List   = List;
FlyOut.Item   = Item;

export { FlyOut };
```

Usage — the consumer imports a single thing and gets all sub-parts via dot notation:

```tsx
import { FlyOut } from './FlyOut';

export function FlyoutMenu() {
  return (
    <FlyOut>
      <FlyOut.Toggle />
      <FlyOut.List>
        <FlyOut.Item>Edit</FlyOut.Item>
        <FlyOut.Item>Delete</FlyOut.Item>
      </FlyOut.List>
    </FlyOut>
  );
}
```

> Source: [patterns.dev — Context API example](https://www.patterns.dev/react/compound-pattern/)

***

### 2. React.Children + cloneElement (limited use)

Passes state as props by cloning each direct child. Simpler for tiny cases, but has
real constraints documented by patterns.dev:

- Only **direct children** receive the injected props — wrap them in a `<div>` and they
  lose access immediately.
- Props are shallowly merged, so naming collisions **silently overwrite** values.
- The injected props leak into the public API — consumers can read them.

```tsx
export function FlyOut(props: { children: React.ReactNode }) {
  const [open, setOpen] = React.useState(false);
  const toggle = () => setOpen(o => !o);

  return (
    <div>
      {React.Children.map(props.children, child =>
        React.cloneElement(child as React.ReactElement, { open, toggle })
      )}
    </div>
  );
}
```

```tsx
// ❌ Breaks — Toggle and List are no longer direct children
<FlyOut>
  <div>
    <FlyOut.Toggle />
    <FlyOut.List>...</FlyOut.List>
  </div>
</FlyOut>
```

**Prefer Context API in almost every real case.** Use `cloneElement` only for simple
presentational parent/child relationships with no deep nesting.

> Source: [patterns.dev — React.Children.map section](https://www.patterns.dev/react/compound-pattern/)

***

## Key design decisions

### Keep context private

Don't export the context object — only export the custom hook (`useFlyOut`) if children
need to be consumed outside the file. This prevents consumers from reading or injecting
state they shouldn't control.

### Guard hook for early, clear errors

```tsx
function useFlyOut() {
  const ctx = useContext(FlyOutContext);
  if (!ctx) throw new Error('FlyOut sub-components must be used inside <FlyOut>');
  return ctx;
}
```

### Memoize context value to avoid extra renders

When the provider re-renders for unrelated reasons, an inline `value={{ open, toggle }}`
creates a new object each render, forcing all consumers to re-render too:

```tsx
// ❌ New object every render
<FlyOutContext.Provider value={{ open, toggle }}>

// ✅ Stable reference — only re-renders consumers when open or toggle actually changes
const value = useMemo(() => ({ open, toggle }), [open, toggle]);
<FlyOutContext.Provider value={value}>
```

For fine-grained control, split into two contexts: one for the state (`open`), one for
the setter (`toggle`). Components that only dispatch will never re-render on state changes.

> Source: [patterns.dev — avoiding unnecessary re-renders note](https://www.patterns.dev/react/compound-pattern/)

### Server Components (React 18+/19+)

Context providers and `useState` are client-only. Mark the file with `'use client'`.
Static sub-components (like `Item` if it holds no state) can remain server components
if extracted to separate files — only the stateful shell needs `'use client'`.

***

## Common use-cases for this pattern

- **Dropdown / FlyOut menu** — Toggle + List + Item (the canonical example)
- **Tabs** — Tabs + TabList + Tab + TabPanel
- **Accordion** — Accordion + AccordionItem + AccordionTrigger + AccordionContent
- **Modal / Dialog** — Dialog + DialogTrigger + DialogContent + DialogTitle + DialogClose
- **Data table** — Table + TableHead + TableBody + TableRow + TableCell
- **Select / Combobox** — Select + SelectTrigger + SelectContent + SelectItem
- **Form** — Form + FormField + FormLabel + FormMessage

> "You'll often see this pattern when using UI libraries like Semantic UI." — patterns.dev

***

## When NOT to use it

- The component has only one "part" with no meaningful sub-structure → plain component.
- Children are purely presentational and share no state → just use the `children` prop.
- You need a data-driven API for many repetitive items (e.g. 200-row virtual list) →
  consider a hybrid: compound wrapper + virtualized inner renderer.

***

## Building checklist

1. **Define the state** that needs to be shared and put it in the parent.
2. **Create a context** (not exported) and a `useXxx` guard hook.
3. **Build each sub-component** so it reads only what it needs from context.
4. **Attach sub-components** as static properties (`Parent.Child = Child`).
5. **Export only the parent** (and optionally named exports for individual sub-components).
6. **Memoize** the context value if the parent can re-render for unrelated reasons.
7. **Write the usage example first** — does the JSX feel natural and readable?
   If not, reconsider the API before implementing.

***

## Quick diagnosis

| Signal | Recommendation |
|---|---|
| You're passing config arrays/objects as props | Convert into child components |
| Multiple props share the same prefix | That prefix is a child component |
| The `return` requires mental abstraction to visualize | Switch to compound components |
| Children must be in a specific order or format | Consider a data-driven API instead |

> "If you find yourself passing config objects/arrays as props, consumers will often have
> to transform their data first. A compound component can prevent that overhead."
> — jjenzz.com
