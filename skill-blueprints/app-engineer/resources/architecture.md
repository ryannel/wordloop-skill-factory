---
trigger: always_on
---

# Frontend Architecture & Codebase Conventions: Word Loop

This document governs all code generation, refactoring, and feature implementation for the Word Loop frontend. The application is a Next.js 16 App Router project consuming a Go backend REST API. Strict adherence to these structural boundaries is mandatory to maintain a predictable, type-safe, and highly performant codebase.

## 1. Core Tooling & Hard Rules

Before contributing to the codebase, engineers must align with the following non-negotiable environment constraints:

* **Package Manager:** Use `pnpm` exclusively. `npm` and `yarn` are strictly prohibited.
* **Data Fetching:** Never use `useEffect` for data fetching. All client-side fetching must route through SWR.
* **Logging:** `console.log` is forbidden in production environments. Utilise a dedicated logging utility.
* **Theme Testing:** Every component and feature must be tested in both Dark (Obsidian) and Light (Milk) themes before merging.
* **Icons:** Use `lucide-react` exclusively. Standardise standard actions to `size={20}` and inline text icons to `size={16}`. Always use `aria-label` for icon-only buttons.

## 2. File Naming & Structural Conventions

Predictability in the file system accelerates development and prevents structural decay.

* **Casing:** All files and directories must use `kebab-case` (e.g., `word-card.tsx`, `use-word-list.ts`). `PascalCase` and `camelCase` are strictly forbidden for filenames.
* **Component Exports:** Exported React components must use `PascalCase`, even though they reside in `kebab-case` files.
* **Barrel Exports:** Every domain-specific folder must include an `index.ts` file that re-exports its public components.
* **Decomposition Threshold:** Any component exceeding approximately 150 lines of code must be broken down into smaller, composable units.

## 3. Tailwind CSS v4 Configuration

The application utilises modern Tailwind v4, which shifts away from JavaScript-based configuration towards a CSS-first architecture.

* **No Config File:** Do not create or look for a `tailwind.config.ts` file. It does not exist in this architecture.
* **Global CSS:** All configuration is executed within `app/globals.css` using `@theme {}` blocks and CSS custom properties.
* **PostCSS:** The build pipeline relies on `@tailwindcss/postcss`.
* **Imports & Variants:** Ensure `@import "tailwindcss";` sits at the top of the global CSS file. Custom variants must use the `@custom-variant` directive (e.g., `@custom-variant dark (&:is(.dark *));`).
* **Legacy Syntax:** Do not use Tailwind v3 syntax or paradigms.
* **Animations:** Rely on `tw-animate-css` for utility-based animations.

## 4. Architectural Layers & Inward Dependencies

The codebase follows a strict inward-facing dependency graph. High-level routing and UI must depend on lower-level logic, but lower-level logic (like schemas or utilities) must never import from the UI or routing layers.

| Layer | Directory | Responsibility & Dependency Rules |
| :--- | :--- | :--- |
| **Schemas** | `lib/schemas/` | Zod schemas and inferred types. Must have **zero** internal application dependencies. |
| **API Client** | `lib/api.ts` | One function per Go REST endpoint. Depends strictly on Schemas. |
| **Utilities** | `lib/utils.ts` | Pure functions (e.g., `cn`, formatters). Must have **zero** internal application dependencies. |
| **Hooks** | `hooks/` | SWR data-fetching and utilities. Depends only on the API Client and Schemas. |
| **Contexts** | `lib/contexts.tsx` | Cross-cutting client state. Must **never** import UI components. |
| **UI Primitives**| `components/ui/` | `shadcn/ui` components. Never edit these manually. Update via `npx shadcn@latest add`. |
| **Features** | `components/<domain>/`| Domain-specific UI. Permitted to import Primitives, Hooks, Contexts, and Utilities. |
| **Pages** | `app/` | Route definitions. These must remain incredibly thin, immediately delegating rendering logic to Feature Components. |

## 5. Type System & The Data Boundary

TypeScript is enforced strictly to prevent runtime errors and ensure seamless communication with the Go backend.

* **Zod is the Contract:** Use `z.infer<typeof Schema>` for all types. Do not write standalone TypeScript interfaces for API data.
* **The RSC Boundary:** Server Components must only pass plain, serialisable objects to Client Components.
* **Type Safety:** The `any` type is forbidden. Use `unknown` and narrow the type using type guards.
* **State Management:** Favour discriminated unions (e.g., `{ type: "recording" } | { type: "upload" }`) over loose optional fields to prevent impossible states.
* **Switch Statements:** Always enforce exhaustive matching using `default: never` to catch unhandled cases at compile time.

## 6. Server Actions & The Result Pattern

To prevent unhandled client-side crashes from failed mutations, all Server Actions must return a strictly typed Result pattern.

```typescript
type ActionResult<T> = 
  | { data: T; error: null } 
  | { data: null; error: string };
```

Implement React Error Boundaries to catch catastrophic UI failures, and rely on SWR's built-in error states to handle graceful degradation for API fetch failures.

## 7. Component Strategy & Data Fetching

The application prioritises server-side execution for performance, dropping to the client only when interactivity demands it.

* **RSC Default:** All components are React Server Components by default. Only add the `"use client"` directive when browser APIs, React state, or hooks are strictly required.
* **Mutations & Forms:** Use Server Actions for all data mutations. Forms must utilise `react-hook-form` paired with `@hookform/resolvers/zod`. Manage loading states via `useFormStatus` or `useTransition`.
* **SWR Pattern:** All client-side data fetching hooks reside in `hooks/use-data.ts`.
* **Hydration:** Seed client-side SWR hooks by wrapping Server Components in `<SWRConfig value={{ fallback }}>`.
* **Optimistic UI:** When a user triggers an action, immediately update the local SWR cache using `mutate()` before receiving server confirmation. Revert the cache if the Server Action returns an error.

## 8. Testing Standards

Testing focuses on user behaviour, accessibility, and critical utility functions rather than brittle implementation details.

* **Framework:** Use Vitest alongside the React Testing Library (RTL).
* **Network Mocking:** Never mock the global `fetch` manually. Use Mock Service Worker (MSW) to intercept and resolve network requests.
* **Coverage:** `lib/utils.ts` requires 100% test coverage. 
* **Hooks:** Test custom hooks in isolation using `@testing-library/react-hooks`.
