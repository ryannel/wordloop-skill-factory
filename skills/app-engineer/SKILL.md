---
name: app-engineer
description: Implements features for the wordloop-app Next.js 16 App Router frontend with strict adherence to architectural layers, Tailwind v4 CSS-first styling, OKLCH colour system, Liquid Glass visual language, Zod-based type safety, SWR data fetching, and Server Action mutations. Use when writing, reviewing, or refactoring any TypeScript, React, or CSS code in wordloop-app — including page routes, feature components, data hooks, server actions, schemas, styling, theming, forms, error boundaries, modals, or tests. Invoke for any work touching app/, components/, hooks/, lib/, or configuration files, even if the user doesn't explicitly mention architecture, styling, or best practices. Also invoke when the user asks about React Server Components, client components, Tailwind v4, SWR patterns, Zod schemas, Next.js routing, parallel routes, intercepting routes, image optimisation, font loading, deployment, or Docker for this codebase.
---

# App Engineer — wordloop-app

Frontend engineer specialising in the wordloop-app service. Every component, route, and style respects the codebase's architectural layer boundaries, Liquid Glass visual language, and modern Next.js App Router patterns.

## Codebase Identity

The wordloop-app is a **Next.js 16 App Router** project that consumes a Go backend REST API. It uses React Server Components by default, drops to client components only when interactivity demands it, and enforces strict type safety through Zod schemas as the single source of truth for all API data.

### Hard Rules

These are non-negotiable constraints that apply to every change:

- **Package manager:** Use `pnpm` exclusively. `npm` and `yarn` are forbidden.
- **Data fetching:** Never use `useEffect` for data fetching. All client-side fetching routes through SWR.
- **Logging:** `console.log` is forbidden in production. Use the dedicated logging utility.
- **Theme testing:** Every component must be verified in both **Obsidian** (dark) and **Milk** (light) themes before merging.
- **Icons:** Use `lucide-react` exclusively. Standard actions: `size={20}`. Inline text icons: `size={16}`. Always use `aria-label` for icon-only buttons.
- **File naming:** All files and directories use `kebab-case`. `PascalCase` and `camelCase` filenames are forbidden.
- **Component exports:** Exported React components use `PascalCase` names despite residing in `kebab-case` files.
- **Barrel exports:** Every domain-specific folder includes an `index.ts` that re-exports its public components.
- **Decomposition:** Any component exceeding ~150 lines must be broken into smaller, composable units.
- **Types:** The `any` type is forbidden. Use `unknown` and narrow with type guards.
- **Colours:** All colours are defined in **OKLCH**. Hexadecimal, RGB, and HSL are strictly forbidden.
- **Tailwind:** Use Tailwind v4 CSS-first syntax only. No `tailwind.config.ts`, no v3 syntax.

## Architectural Layers

The codebase follows a strict inward-facing dependency graph. Higher layers depend on lower layers — never the reverse.

| Layer | Directory | Responsibility | Depends On |
|-------|-----------|----------------|------------|
| **Schemas** | `lib/schemas/` | Zod schemas and `z.infer` types | Zero internal dependencies |
| **API Client** | `lib/api.ts` | One function per Go REST endpoint | Schemas only |
| **Utilities** | `lib/utils.ts` | Pure functions (`cn`, formatters) | Zero internal dependencies |
| **Hooks** | `hooks/` | SWR data-fetching and utilities | API Client, Schemas |
| **Contexts** | `lib/contexts.tsx` | Cross-cutting client state | Never imports UI components |
| **UI Primitives** | `components/ui/` | shadcn/ui components | Never edit manually; update via `npx shadcn@latest add` |
| **Features** | `components/<domain>/` | Domain-specific UI | Primitives, Hooks, Contexts, Utilities |
| **Pages** | `app/` | Route definitions | Immediately delegate to Feature Components — pages must remain thin |

**The cardinal rule: dependencies only point inward.** A schema never imports a hook. A hook never imports a component. A page never contains business logic — it delegates to feature components immediately.

## Implementation Workflow

Follow this sequence for every feature or change:

1. **Locate the layer** — Determine which architectural layer the change belongs to. This dictates import boundaries and testing strategy.
2. **Design the types** — Define or extend Zod schemas in `lib/schemas/` first. All downstream types derive from these schemas via `z.infer`.
3. **Build the data path** — If new data is needed: add the API client function → create the SWR hook → seed hydration from the server component.
4. **Build the component** — Server Component by default. Add `"use client"` only when hooks, events, or browser APIs are strictly required.
5. **Apply the visual language** — Use the OKLCH colour tokens, Geist typography scale, 8pt spatial grid, and the correct Liquid Glass card variant.
6. **Write tests** — Vitest + RTL for components. MSW for network mocking. Test in both themes.
7. **Verify** — Run `pnpm lint`, check both themes visually, confirm no layer violations.

## Reference Guide

Load detailed implementation guidance based on the task at hand:

| Topic | Reference | Load When |
|-------|-----------|-----------|
| Layers & File Structure | `references/architecture.md` | Layer boundaries, directory layout, file conventions, barrel exports, route structure |
| Type Safety & Schemas | `references/type-system.md` | Zod contracts, discriminated unions, Result pattern, serialisation rules |
| Server vs Client Components | `references/server-components.md` | RSC boundaries, async params, directives, runtime selection |
| Data Fetching | `references/data-fetching.md` | SWR hooks, hydration seeding, waterfall prevention, optimistic UI, route handlers |
| Mutations & Forms | `references/mutations-and-forms.md` | Server Actions, react-hook-form, useTransition, error flow |
| Routing & Navigation | `references/routing-and-navigation.md` | File-based routing, parallel/intercepting routes, proxy, modals |
| Error Boundaries | `references/error-boundaries.md` | error.tsx, not-found, auth errors, navigation gotcha, hydration errors |
| Visual Language & Surfaces | `references/visual-language.md` | OKLCH palette, Geist typography, 8pt grid, surface dynamics, Liquid Glass cards |
| Tailwind & Theming | `references/tailwind-and-styling.md` | Tailwind v4 CSS-first config, @theme, @custom-variant, animations |
| UX Principles | `references/ux-principles.md` | Speed, navigation, info architecture, feedback, accessibility, empty states, copy |
| Testing | `references/testing.md` | Vitest + RTL, MSW, hook isolation, theme coverage, accessibility |
| Performance & Deployment | `references/performance-and-deployment.md` | Images, fonts, bundling, scripts, metadata, self-hosting, Docker, debug |

## Constraints

### Do

- Place all API data types in `lib/schemas/` as Zod schemas — derive TypeScript types via `z.infer<typeof Schema>`
- Use `"use client"` only when browser APIs, React state, or hooks are strictly required
- Fetch data in Server Components where possible — pass serialised data to client components as props
- Use SWR for all client-side data fetching — hooks go in `hooks/use-data.ts`
- Seed SWR caches by wrapping Server Components in `<SWRConfig value={{ fallback }}>`
- Use Server Actions for all mutations — never call the API client directly from client components for writes
- Return `ActionResult<T>` from all Server Actions to prevent unhandled client-side crashes
- Use `react-hook-form` with `@hookform/resolvers/zod` for all form validation
- Use discriminated unions for state modeling — prevent impossible states at the type level
- Apply OKLCH colour tokens from CSS variables — never hardcode colour values
- Use the Geist typographic scale — never invent ad-hoc font sizes
- Follow the 8pt spatial grid — all spacing uses multiples of 0.5rem
- Test every component in both Obsidian (dark) and Milk (light) themes

### Do Not

- Use `useEffect` for data fetching — use SWR
- Use `any` as a type — use `unknown` and narrow
- Write standalone TypeScript interfaces for API data — use `z.infer`
- Write `console.log` in production code — use the logging utility
- Manually edit `components/ui/` files — update via `npx shadcn@latest add`
- Use hex, RGB, or HSL colours — use OKLCH exclusively
- Use Tailwind v3 syntax or create `tailwind.config.ts` — use CSS-first v4 patterns
- Create separate "view" and "edit" screens — use inline editing
- Use harsh stoplight reds/greens — use Sage Green for success, Muted Rose for errors
- Block workflows with loading spinners for simple submissions — use optimistic UI
- Exceed ~150 lines in a single component — decompose

## Anti-patterns to Flag

When reviewing or writing wordloop-app code, actively flag these violations:

| Anti-pattern | Why It Matters |
|-------------|---------------|
| **Layer crossing** | A hook importing a component, or a schema importing a hook, breaks the dependency direction |
| **Fat pages** | Route files in `app/` that contain business logic instead of delegating to feature components |
| **useEffect fetching** | Bypasses SWR's caching, deduplication, and revalidation — leads to waterfall requests and stale data |
| **Raw fetch in client** | Client components calling `lib/api.ts` directly for mutations instead of using Server Actions |
| **Unserialisable props** | Passing Date objects, Maps, Sets, or functions (except Server Actions) from server to client components |
| **Missing Suspense** | `useSearchParams` without a Suspense boundary causes full-page CSR bailout |
| **router.push to close modal** | Creates broken history entries — use `router.back()` to close intercepting route modals |
| **Hardcoded colours** | Using hex/RGB/HSL instead of OKLCH CSS variables |
| **Missing barrel export** | Domain folder without an `index.ts` re-exporting public components |
| **Monolith component** | Component exceeding 150 lines without decomposition |
| **Tailwind v3 config** | Creating `tailwind.config.ts` or using `@apply` with v3 syntax |
