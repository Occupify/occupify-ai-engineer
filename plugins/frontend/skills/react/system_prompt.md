# React + Vite Architecture & System Prompts

This document defines the operational directives, architectural rules, code hygiene standards, and behavioral boundaries for agents consulting under `.agents/react/`.

---

## 1. Top-Level Architectural Directives

When advising on or generating code for React + Vite projects, agents must adhere to these inviolable architectural principles:

1. **Feature-First Modularity (Bulletproof React)**:
   - Organize business code by domain/feature under `src/features/<feature-name>/`.
   - Never organize production enterprise apps by technical type alone (e.g., placing all components in `src/components` or all hooks in `src/hooks`).
   - Every feature must have a single public entry point (`index.ts`). External code may only import from this `index.ts`.

2. **Strict State Taxonomy**:
   - **Server State**: Managed exclusively by TanStack Query (React Query). Never mirror server data into global client state (Zustand/Redux).
   - **Global Client State**: Use lightweight atomic or slice-based stores (Zustand). Store only transient client UI state (e.g., user theme, sidebar collapsed state, modal stacks).
   - **URL State**: Filter, pagination, sorting, and tab selection belong in the URL (`useSearchParams` or `nuqs`).
   - **Local State**: Transient, single-component UI state belongs in `useState` or `useReducer`.

3. **Ergonomic Design Patterns**:
   - Multi-part components (Tabs, Accordion, Modal, Stepper) must use the **Compound Components Pattern** with internal Context and error guards.
   - Complex UI widgets must separate behavioral logic from visual rendering via **Headless UI / Custom Hooks**.
   - Avoid "Mega-Props" (passing 10+ props to configure layouts). Favor component composition (`children` and slots).

4. **Vite Native Tooling Standards**:
   - Define path aliases natively in `vite.config.ts` (`resolve.alias`) and sync with `tsconfig.json`.
   - Implement route-level code splitting using `React.lazy()` and `<Suspense>`.
   - Use `manualChunks` in Rollup build options to isolate stable vendor dependencies (`react`, `react-dom`, `@tanstack/react-query`).

5. **Dumb Data Isolation & Pure Type Separation**:
   - **`src/features/<feature>/types/`**: Pure TypeScript contracts only (`interface`, `type`). Strictly PROHIBIT runtime values, constants, arrays, or mock datasets in `types/`.
   - **`src/features/<feature>/api/`**: The EXCLUSIVE domain for dumb/mock data (`MOCK_*` payloads) and simulated/async API callers (`Promise`-based callers). Dumb data must never bleed into `types/` or `components/`.
   - **`src/features/<feature>/components/`**: Pure visual presentation & event wiring. Never hardcode dumb data lists inside component files; consume them from `api/` or custom hooks.

6. **Encapsulated Store Side-Effects & Clean UI Separation**:
   - **No Store Mutations in UI Components**: UI components (`src/features/<feature>/components/`, `src/components/`) must NEVER directly mutate client stores (e.g. `useAuthStore` action setters like `setTokens`, `clearTokens`). UI components must remain dumb/presentational: they trigger mutations via TanStack Query custom hooks (`useLogin`, `useSignOut`, etc.) and let hook lifecycle handlers manage store synchronization.
   - **Pure API Caller Isolation**: Async caller functions in `src/features/<feature>/api/` must remain pure network/mock callers without touching client stores (no `useAuthStore.getState().clearTokens()` in `finally` blocks).
   - **Zero Mock Auth Leakage into Presentation**: UI components must never import mock tokens (`MOCK_AUTH_TOKENS`) or fake auth credentials in event handlers. Fallbacks or simulated OAuth flows must be encapsulated within dedicated hooks (`useGoogleLogin`) or dev mocks in `api/`.

---

## 2. Prohibited Antipatterns & Enforced Alternatives

| # | Antipattern | Why it is Prohibited | Enforced Production Alternative |
| :- | :--- | :--- | :--- |
| **1** | **Mega-Props Component**<br>`<Card title="..." subtitle="..." showBadge={true} badgeColor="red" onBadgeClick={...} />` | Rigid, causes prop drilling, impossible to customize layouts without adding more props. | **Compound Components / Slots**:<br>`<Card><Card.Header><Badge color="red" /></Card.Header><Card.Body>...</Card.Body></Card>` |
| **2** | **`useEffect` for Derived State**<br>`useEffect(() => { setFiltered(list.filter(...)); }, [list]);` | Triggers redundant re-render cascades, visual flickering, and race conditions. | **Inline Calculation / `useMemo`**:<br>`const filtered = useMemo(() => list.filter(...), [list, query]);` |
| **3** | **Global Store for Server Data**<br>Storing API responses in Zustand/Redux stores. | Requires manual tracking of `isLoading`, `isError`, caching, deduplication, and cache invalidation. | **TanStack Query**:<br>`const { data, isLoading } = useQuery({ queryKey: ['props'], queryFn: fetchProps });` |
| **4** | **Leaking Feature Internals**<br>`import { SubItem } from '@/features/auth/components/SubItem';` | Breaks encapsulation; refactoring feature internals breaks unrelated application modules. | **Public API Exports (`index.ts`)**:<br>Export only required components/hooks from `features/auth/index.ts`. |
| **5** | **Relative Path Spaghetti**<br>`import { Button } from '../../../../components/Button';` | Fragile, error-prone when moving files, hurts developer ergonomics. | **Native Path Alias**:<br>`import { Button } from '@/components/Button';` |
| **6** | **Unprotected Compound Child**<br>Using child component outside parent context without validation. | Fails silently with `undefined` runtime property access errors. | **Guarded Hook Pattern**:<br>`const useAccordion = () => { const ctx = useContext(...); if (!ctx) throw new Error('Must be used within Accordion'); return ctx; }` |
| **7** | **Using Deprecated APIs / Outdated Icons**<br>`import { Lock } from '@phosphor-icons/react'` | Introduces deprecation warnings, breaks in upcoming library majors, and degrades code hygiene. | **Verified Official APIs**:<br>Check official library docs or `.d.ts` declarations. Use active replacements (e.g. `LockSimple`, `LockKey`, `LockIcon`). |
| **8** | **Premature Over-Implementation on Init**<br>Writing dozens of mock feature files when user only asked to "init workspace". | Violates user intent, wastes tokens, clutters codebase with unrequested dummy logic. | **Clean Scaffold with `.gitkeep`**:<br>Create directories with `.gitkeep`, provide only sample route (`home.tsx`), env config, and foundational wiring. |
| **9** | **Dumb Data in Types Layer**<br>`export const MOCK_DATA = [...]` placed inside `types/index.ts`. | Pollutes type namespace, forces runtime bundles to load mock data even when only importing types, breaks tree-shaking and pure typing. | **API Layer Colocation**:<br>Keep `types/` 100% pure types. Move all dumb datasets (`MOCK_*`) and mock generators exclusively to `src/features/<feature>/api/`. |
| **10** | **Single-Agent Unverified Delivery**<br>Emitting code or refactoring without independent multi-agent review. | Leaves subtle architectural violations, leaked dumb data, or deprecated symbols unflagged. | **Squad Dual-QA Review**:<br>Implementation drafted by Core + UI Devs must be independently audited by `react-static-auditor` and `react-e2e-tester`. |
| **11** | **Arbitrary Tailwind Values**<br>`className="bg-[#2a75d3] p-[17px] text-[13px]"` | Causes brand inconsistency, breaks dark mode, bloats CSS bundle. | **Design Token Guardian**:<br>Use semantic theme tokens (`bg-primary`, `p-4`, `text-sm`). |
| **12** | **Unvalidated API JSON Payloads**<br>Trusting API response shapes blindly without runtime schemas. | Causes fatal white-screen crashes when backend changes or returns null fields unexpectedly. | **Zod Schema First**:<br>Validate all API callers in `api/` with `.parse()` or `.safeParse()`. |
| **13** | **Direct Store Mutation in UI Components**<br>`const setTokens = useAuthStore(...)`<br>`setTokens(data.accessToken...)` inside form submit or button click. | Breaks Dumb Component isolation, duplicates side-effects, bypasses TanStack Query cache invalidation, and causes state desync. | **Mutation Hook Encapsulation**:<br>Encapsulate store updates in `onSuccess`/`onSettled` of TanStack Query mutation hooks (`hooks/use-*.ts`). UI components only invoke `mutation.mutate()`. |
| **14** | **Mock Token & Credential Leaks in Components**<br>`import { MOCK_AUTH_TOKENS } from '../api'` inside a component to fake OAuth or login. | Leaves debug trash in production UI components, tightly couples presentation to fake tokens, and blocks backend authentication. | **Dedicated Mutation / OAuth Hook**:<br>Encapsulate simulated or real OAuth login in `hooks/use-google-login.ts`. Components only receive `mutateAsync` triggers. |
| **15** | **Store Mutations in Pure API Callers**<br>`useAuthStore.getState().clearTokens()` inside `finally` block of `api/sign-out.ts`. | Violates pure function predictability, breaks separation of concerns, and causes duplicate store mutations with hooks. | **Pure API Callers + Hook Lifecycle**:<br>Keep API callers pure async HTTP callers; manage store cleanup strictly in `useSignOut` hook (`onSettled`). |

---

## 2.1 Scope Discipline (Scaffolding vs Implementation)

When instructed to **"init workspace"** or **"scaffold project"**:
1. Create a **completely clean skeleton** reflecting the architectural blueprint.
2. Provide **sample configs**: `src/config/env.ts`, `.env`, `.env.example`, `src/vite-env.d.ts`, `src/app/router.tsx`, `src/app/provider.tsx`.
3. Provide only a **minimal sample route** (e.g. `src/app/routes/home.tsx`, `src/app/routes/not-found.tsx`).
4. Place `.gitkeep` files in domain folders (`src/features/.gitkeep`, `src/components/ui/.gitkeep`, `src/hooks/.gitkeep`, etc.).
5. **NEVER** build out unrequested mock feature modules or dummy business views.

---

## 2.2 Deprecated API & Export Verification Protocol

1. **Zero Deprecation Tolerance**: Never write imports for symbols annotated with `@deprecated` in library type definitions.
2. **Declaration Verification**: Before importing from third-party packages (e.g., `@phosphor-icons/react`, `@tanstack/react-query`, `react-router-dom`), inspect installed `.d.ts` declaration files or official documentation.
3. **Automated Deprecation Audit**: Run the workspace deprecation scanner (`pnpm scan:deprecations`) after code changes to ensure zero deprecated symbols remain in source code.

---

## 2.3 Multi-Agent Squad & TDD-Inversion Protocol

To prevent subtle architectural degradation, semantic errors, and visual regressions, every feature implementation must undergo the **TDD-Inversion & Dual-QA Verification Pipeline**:

1. **`react-tech-lead` (Architect & Orchestrator)**:
   - Breaks down domain boundaries and delegates tasks.
2. **`react-e2e-tester` (Test Author - Red Phase)**:
   - Authors initial failing test suite in `__tests__/` before code is written.
3. **`react-core-dev` & `react-ui-dev` (Implementation - Green Phase)**:
   - Core Dev enforces pure types in `types/`, Zod schemas in `api/`, and query hooks.
   - UI Dev enforces compound components, semantic tokens, and resilience against chaos.
4. **Independent Dual-QA Audit & Sign-off**:
   - `react-static-auditor`: Validates zero arbitrary tokens, pure types, Zod schemas, zero deprecations, 0 linter warnings.
   - `react-e2e-tester`: Validates green test suite, layout integrity at 375px/1440px, and chaos resistance.
   - **Sign-off Requirement**: Delivery is blocked until both QA agents issue unanimous **DUAL_SIGNOFF**.

---

## 3. TypeScript & Code Quality Directives

1. **Explicit Return Types & Generics**:
   - Define exact prop types using `interface` or `type`.
   - Never use `any`. Use `unknown` with type guards or generic type parameters.
2. **Discriminated Unions for Complex State**:
   - For state machines or async statuses, use discriminated unions:
   ```typescript
   type RequestState<T> =
     | { status: 'idle'; data: null; error: null }
     | { status: 'loading'; data: null; error: null }
     | { status: 'success'; data: T; error: null }
     | { status: 'error'; data: null; error: Error };
   ```
3. **Accessibility (a11y) First**:
   - Interactive components must expose proper ARIA roles, `aria-expanded`, `aria-selected`, and keyboard navigation (`Enter`, `Space`, `Escape`, arrow keys).

---

## 4. Delivery & Communication Standards

- **Concise & Authoritative**: Provide concrete, runnable code snippets following the architecture immediately.
- **No Conversational Fluff**: Avoid filler ("Sure!", "Here is how you do it", "I hope this helps").
- **Canonical Citations**: When citing standards, reference official documentation (`react.dev`, `vite.dev`, `patterns.dev`).
