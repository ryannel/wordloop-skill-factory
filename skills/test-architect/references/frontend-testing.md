# Frontend Testing Strategy

This module defines how the Test Architect principles (High-Fidelity Emulation, Risk-Based Engineering, Observability) apply to frontend applications, specifically modern stacks like Next.js App Router and React.

---

## Table of Contents

1. [The Frontend Testing Trophy](#the-frontend-testing-trophy)
2. [Mapping to the Validation Grid](#mapping-to-the-validation-grid)
3. [Network Emulation (MSW)](#network-emulation-msw)
4. [Component Testing Philosophy](#component-testing-philosophy)
5. [Hook & Data Testing](#hook--data-testing)
6. [Visual and Accessibility Validation](#visual-and-accessibility-validation)

---

## The Frontend Testing Trophy

Frontend systems should follow the **Testing Trophy** shape, where the bulk of investment goes into static analysis and integration testing.

- **Static Analysis (High ROI):** TypeScript strict mode, ESLint, and Zod schemas prevent entire classes of runtime errors before code even executes.
- **Integration Tests (The Core):** React Testing Library + MSW. Renders the UI, simulates user events, and intercepts network traffic at the service worker level.
- **E2E Tests (Targeted):** Playwright or Cypress running against an actual compiled frontend and a containerized backend (`Testcontainers`). Reserved for critical user journeys.
- **Unit Tests (Rare):** Surgical tests for pure functions (utilities, formatters, complex state reducers).

---

## Mapping to the Validation Grid

In a frontend context, the test taxonomy translates as follows:

| Taxonomy | Frontend Equivalent | Emulation Strategy |
|----------|---------------------|--------------------|
| **Small (Unit)** | `lib/utils.ts`, Zod schemas, pure transformers | None (pure JS execution in memory) |
| **Medium (Service Test)** | Feature components, SWR data-fetching hooks | **MSW (Mock Service Worker)** emulates the backend API |
| **Large (System Test)** | Playwright testing the full site flow | **Testcontainers** running the actual backend locally |
| **Live System Test** | UI testing against staging/production | Real network traffic, isolated test accounts |

> **Crucial Distinction:** In backend Service Tests, we rely on Testcontainers to emulate databases. In frontend Component/Service Tests, we rely on **MSW** to emulate the backend API. MSW is the frontend's equivalent of a high-fidelity emulator because it intercepts requests at the network layer.

---

## Network Emulation (MSW)

Do not mock `fetch`, `axios`, or specific data hooks directly.

**Antipattern (Implementation Mocking):**
```tsx
// BAD: Tightly coupled to implementation. Misses URL formulation, header, and serialization errors.
vi.spyOn(global, 'fetch').mockResolvedValue(new Response(...));
```

**Architectural Standard:**
Use MSW (Mock Service Worker). MSW acts as an in-browser (or in-Node) proxy. Your component executes a real HTTP request, and MSW intercepts it, returning the emulated response.

```tsx
// GOOD: Testing real fetch behavior via MSW interception
server.use(
  http.post('/api/v1/meetings', async ({ request }) => {
    const body = await request.json();
    if (!body.title) {
      return HttpResponse.json({ error: 'Title required' }, { status: 400 });
    }
    return HttpResponse.json({ id: '123', title: body.title }, { status: 201 });
  })
);
```

---

## Component Testing Philosophy

Tests must verify behavior from the **user's perspective**, not the developer's.

1. **Query by accessible roles:** Use `getByRole`, `getByLabelText`, and `getByText`. This inherently validates that the UI is accessible.
2. **Never query by implementation:** Do not use CSS selectors (`.card-wrapper`), component instance types, or generic test IDs (`data-testid`) unless absolutely no accessible alternative exists.
3. **Trigger real browser events:** Use `@testing-library/user-event` instead of `fireEvent`. `userEvent` simulates the full cascade of browser events (focus, keydown, keypress, keyup, change), matching real browser behavior.

---

## Hook & Data Testing

Data-fetching hooks (e.g., SWR, React Query) represent the boundary between your UI and the backend network.

When testing hooks in isolation:
1. **Fresh cache per test:** Wrap the hook in a fresh cache provider to prevent test pollution. Test pollution in caching layers is the leading cause of flaky frontend tests.
2. **Disable retries:** Turn off deduplication and automatic error retries during testing to prevent tests from hanging.
3. **Assert loading states:** Explicitly test the `isLoading`, `data`, and `error` transitions driven by MSW network handlers.

---

## Visual and Accessibility Validation

Frontend testing requires validating non-functional UI requirements:

- **Theme Coverage:** If the application defines multiple themes (e.g., light/dark, Obsidian/Milk), feature components must be rendered and snapshot-tested in all theme contexts to catch contrast or CSS variable drift.
- **Accessibility Guardrails:** Integrate tools like `jest-axe` into component tests. Asserting `toHaveNoViolations()` catches missing aria-labels, poor contrast, and invalid DOM hierarchies automatically in the CI pipeline.

---

## See Also

- `architectural-blueprints.md` — Detailed breakdown of the Testing Trophy
- `emulation-strategy.md` — Testcontainers vs MSW boundaries
- `taxonomy-and-naming.md` — How S/M/L classification governs the pipeline
