# Architecture & File Structure

## Table of Contents
- [Dependency Graph](#dependency-graph)
- [Directory Layout](#directory-layout)
- [File Naming Rules](#file-naming-rules)
- [Barrel Exports](#barrel-exports)
- [Component Decomposition](#component-decomposition)
- [Route Structure](#route-structure)
- [shadcn/ui Primitives](#shadcnui-primitives)
- [Import Validation](#import-validation)

---

## Dependency Graph

The wordloop-app enforces a strict inward-facing dependency graph. Every import must point **downward** in this hierarchy — never upward or sideways.

```
┌─────────────────────────────────────────────────┐
│  Pages (app/)                                    │
│  Thin route shells — delegate immediately        │
└──────────────────────┬──────────────────────────┘
                       │ imports
┌──────────────────────▼──────────────────────────┐
│  Features (components/<domain>/)                 │
│  Domain-specific UI — compose from everything    │
│  below                                           │
└──────────────────────┬──────────────────────────┘
                       │ imports
┌──────────────────────▼──────────────────────────┐
│  UI Primitives (components/ui/)                  │
│  shadcn/ui — never edit, update via CLI          │
├──────────────────────┬──────────────────────────┤
│  Contexts (lib/contexts.tsx)                     │
│  Cross-cutting client state — no UI imports      │
├──────────────────────┬──────────────────────────┤
│  Hooks (hooks/)                                  │
│  SWR fetching + utilities                        │
└──────────────────────┬──────────────────────────┘
                       │ imports
┌──────────────────────▼──────────────────────────┐
│  API Client (lib/api.ts)                         │
│  One function per Go REST endpoint               │
├──────────────────────┬──────────────────────────┤
│  Utilities (lib/utils.ts)                        │
│  Pure functions — zero internal dependencies     │
└──────────────────────┬──────────────────────────┘
                       │ imports
┌──────────────────────▼──────────────────────────┐
│  Schemas (lib/schemas/)                          │
│  Zod schemas + z.infer types — ZERO dependencies │
└─────────────────────────────────────────────────┘
```

### Allowed Import Relationships

| Layer | Can Import | Cannot Import |
|-------|-----------|---------------|
| **Schemas** | Standard library only | Everything else |
| **API Client** | Schemas | Hooks, Components, Contexts |
| **Utilities** | Standard library only | Everything else |
| **Hooks** | API Client, Schemas | Components, Contexts |
| **Contexts** | Schemas, Utilities | Components (any) |
| **UI Primitives** | Utilities | Features, Pages, Hooks, Contexts, API Client |
| **Features** | Primitives, Hooks, Contexts, Utilities, Schemas | Pages, other Features (unless explicitly composed) |
| **Pages** | Features, Contexts | Direct use of Hooks, API Client (delegate to features) |

When a page needs data, it fetches in a Server Component and passes serialised props to feature components. Pages are thin wrappers — they never contain business logic, complex layouts, or conditional rendering beyond route-level decisions.

---

## Directory Layout

```
wordloop-app/
├── app/                              # PAGES — Route definitions
│   ├── layout.tsx                    # Root layout (required)
│   ├── page.tsx                      # Home page (/)
│   ├── loading.tsx                   # Root loading UI
│   ├── error.tsx                     # Root error boundary
│   ├── not-found.tsx                 # Root 404
│   ├── global-error.tsx              # Catches root layout errors
│   ├── globals.css                   # Tailwind v4 config (@theme, tokens)
│   ├── (auth)/                       # Route group (no URL impact)
│   │   ├── login/
│   │   │   └── page.tsx
│   │   └── layout.tsx
│   ├── dashboard/
│   │   ├── page.tsx                  # /dashboard
│   │   ├── loading.tsx               # Dashboard-specific loading
│   │   └── [id]/
│   │       └── page.tsx              # /dashboard/:id
│   └── api/                          # Route Handlers (REST endpoints)
│       └── health/
│           └── route.ts              # /api/health
│
├── components/
│   ├── ui/                           # UI PRIMITIVES — shadcn/ui (do not edit)
│   │   ├── button.tsx
│   │   ├── dialog.tsx
│   │   └── ...
│   └── meetings/                     # FEATURES — domain-specific
│       ├── meeting-card.tsx
│       ├── meeting-list.tsx
│       ├── meeting-form.tsx
│       └── index.ts                  # Barrel export
│
├── hooks/                            # HOOKS — SWR + utilities
│   ├── use-meetings.ts
│   ├── use-auth.ts
│   └── use-theme.ts
│
├── lib/                              # SHARED LOGIC
│   ├── schemas/                      # SCHEMAS — Zod definitions
│   │   ├── meeting.ts
│   │   ├── user.ts
│   │   └── index.ts                  # Barrel export
│   ├── api.ts                        # API CLIENT — one fn per endpoint
│   ├── utils.ts                      # UTILITIES — pure functions
│   └── contexts.tsx                  # CONTEXTS — cross-cutting state
│
├── proxy.ts                          # Proxy (Next.js 16 — replaces middleware.ts)
├── postcss.config.js                 # @tailwindcss/postcss
└── next.config.ts
```

---

## File Naming Rules

Every file and directory uses `kebab-case`. This is enforced without exception.

```
# Correct
components/meetings/meeting-card.tsx
hooks/use-meeting-list.ts
lib/schemas/meeting-synthesis.ts

# Incorrect — will be rejected
components/meetings/MeetingCard.tsx        # PascalCase file
hooks/useMeetingList.ts                    # camelCase file
lib/schemas/MeetingSynthesis.ts            # PascalCase file
```

Exported React components use PascalCase names despite their kebab-case filenames:

```tsx
// File: components/meetings/meeting-card.tsx
export function MeetingCard({ meeting }: MeetingCardProps) {
  return <article>...</article>;
}
```

---

## Barrel Exports

Every domain-specific folder must include an `index.ts` that re-exports its public API. This keeps imports clean and establishes clear module boundaries.

```tsx
// components/meetings/index.ts
export { MeetingCard } from './meeting-card';
export { MeetingList } from './meeting-list';
export { MeetingForm } from './meeting-form';

// lib/schemas/index.ts
export { meetingSchema, type Meeting } from './meeting';
export { userSchema, type User } from './user';
```

Consuming code imports from the barrel, not from individual files:

```tsx
// Good — import from barrel
import { MeetingCard, MeetingList } from '@/components/meetings';
import { meetingSchema, type Meeting } from '@/lib/schemas';

// Bad — import from individual files
import { MeetingCard } from '@/components/meetings/meeting-card';
```

---

## Component Decomposition

Any component exceeding approximately **150 lines** must be broken down into smaller, composable units. This threshold is a guideline, not a hard line count — the goal is components that do one thing well.

### When to Split

- The component renders multiple distinct visual sections
- It manages multiple independent pieces of state
- It contains conditional rendering for different modes (view/edit/loading/error)
- It has complex event handling for unrelated interactions

### How to Split

Extract sub-components into the same domain folder. Co-located files share the same barrel export:

```
components/meetings/
├── meeting-detail.tsx          # Parent — orchestrates layout
├── meeting-header.tsx          # Sub-component — title, metadata
├── meeting-transcript.tsx      # Sub-component — transcript panel
├── meeting-actions.tsx         # Sub-component — action buttons
└── index.ts                   # Exports MeetingDetail (and sub-components if needed)
```

---

## Route Structure

### Special Files

| File | Purpose | Must Be Client Component? |
|------|---------|--------------------------|
| `page.tsx` | UI for a route segment | No (RSC by default) |
| `layout.tsx` | Shared UI for segment and children | No |
| `loading.tsx` | Loading UI (wraps page in Suspense) | No |
| `error.tsx` | Error boundary for segment | **Yes** |
| `not-found.tsx` | 404 UI | No |
| `route.ts` | API endpoint (no React) | N/A |
| `template.tsx` | Like layout but re-renders on navigation | No |
| `default.tsx` | Fallback for parallel route slots | No |
| `global-error.tsx` | Catches root layout errors | **Yes** (must include `<html>` and `<body>`) |

### Route Segments

```
app/
├── meetings/               # Static segment: /meetings
├── [id]/                   # Dynamic segment: /:id
├── [...slug]/              # Catch-all: /a/b/c
├── [[...slug]]/            # Optional catch-all: / or /a/b/c
├── (dashboard)/            # Route group (ignored in URL)
└── _components/            # Private folder (not a route)
```

Prefix with `_` to exclude a folder from routing. Use route groups `(name)` to organise routes without affecting the URL structure.

### Thin Pages

Pages in `app/` must remain thin. They fetch data on the server and immediately delegate to feature components:

```tsx
// app/meetings/[id]/page.tsx — CORRECT: thin page
import { getMeeting } from '@/lib/api';
import { MeetingDetail } from '@/components/meetings';

type Props = { params: Promise<{ id: string }> };

export default async function MeetingPage({ params }: Props) {
  const { id } = await params;
  const meeting = await getMeeting(id);
  return <MeetingDetail meeting={meeting} />;
}
```

```tsx
// app/meetings/[id]/page.tsx — WRONG: fat page with inline logic
export default async function MeetingPage({ params }: Props) {
  const { id } = await params;
  const meeting = await getMeeting(id);
  // ❌ 200 lines of rendering logic, state management, and event handlers
  return (
    <div>
      <h1>{meeting.title}</h1>
      {/* ... hundreds of lines ... */}
    </div>
  );
}
```

---

## shadcn/ui Primitives

The `components/ui/` directory contains shadcn/ui components. These are generated — never manually edited.

### Adding or Updating

```bash
pnpm dlx shadcn@latest add button
pnpm dlx shadcn@latest add dialog
```

### Customisation

If a shadcn/ui component needs customisation for the project, create a wrapper in the appropriate feature folder — do not modify the source in `components/ui/`:

```tsx
// components/meetings/meeting-dialog.tsx — wrapper with domain-specific behaviour
import { Dialog, DialogContent, DialogHeader } from '@/components/ui/dialog';

export function MeetingDialog({ children, title }: MeetingDialogProps) {
  return (
    <Dialog>
      <DialogContent className="glass-elevated">
        <DialogHeader>{title}</DialogHeader>
        {children}
      </DialogContent>
    </Dialog>
  );
}
```

---

## Import Validation

Before adding an import, ask:

> "Does this import point downward (toward Schemas) or upward (toward Pages)?"

If it points upward, the design needs to change. Common fixes:

| Problem | Fix |
|---------|-----|
| Hook needs a component | Extract the shared logic into a utility or context |
| Page contains business logic | Move it to a feature component |
| Feature needs data | Create a hook, don't call lib/api.ts directly from client code |
| Schema imports a hook | Move the derived logic out of the schema file |
| Context imports a component | Separate state logic from rendering |
