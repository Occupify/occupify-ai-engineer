# Feature-Based Project Structure (Bulletproof React)

This reference outlines the production directory layout, colocation rules, and boundary contracts for enterprise React + Vite applications.

---

## 1. Directory Blueprint

```text
src/
├── app/                     # App-wide routing, providers, layouts
│   ├── router.tsx           # Router configuration & lazy route loading
│   └── provider.tsx         # Unified global provider tree (QueryClient, Theme, etc.)
│
├── assets/                  # Static assets (SVGs, logos, images)
│
├── components/              # Shared, domain-agnostic UI primitives
│   ├── ui/                  # Base design system primitives (Button, Input, Modal, Badge)
│   └── feedback/            # Toast, Spinner, ErrorBoundary, Skeleton
│
├── config/                  # Global constants, env validation, site config
│   └── env.ts               # Type-safe environment variable reader
│
├── features/                # Domain-driven feature modules (The core business domains)
│   └── auth/                # Business capability: authentication domain
│       ├── api/             # Remote data access (loginApi, registerApi)
│       ├── components/      # Domain-specific UI (LoginForm, SignUpModal, AuthButton)
│       ├── hooks/           # Domain hooks (useAuth, useLogin, useSignUp)
│       ├── types/           # Domain models, credentials, and DTOs
│       └── index.ts         # Public API export gatekeeper for auth
│
├── pages/                   # Routed composition layer (or app/ if using Next.js App Router)
│   └── landing/             # Landing page route composition
│       ├── components/      # Landing-specific presentational sections (Hero, Features, Pricing)
│       ├── LandingPage.tsx  # Assembles sections and integrates auth features
│       └── index.ts         # Public route entry export for router lazy-loading
│
├── hooks/                   # Cross-cutting reusable custom hooks (useDebounce, useMediaQuery)
│
├── lib/                     # Pre-configured third-party library clients
│   ├── api-client.ts        # Axios or fetch wrapper with auth interceptors
│   └── react-query.ts       # QueryClient instance and default cache policies
│
├── stores/                  # Shared global client stores (Zustand)
│
├── types/                   # Cross-feature shared TypeScript interfaces
│
└── utils/                   # Pure utility functions (formatting, dates, math)
```

---

## 2. Layer Anatomy & Responsibilities

### 2.1. Feature Modules (`src/features/<feature-name>/`)

Each folder under `src/features/<feature-name>/` represents an isolated, self-contained business domain.

```text
src/features/auth/
├── api/                     # Remote data access & DUMB DATA REPOSITORY
│   ├── signin.ts            # API caller and MOCK_AUTH_TOKENS
│   ├── signin.test.ts       # Unit test for API caller & Zod validation
│   ├── get-current-user.ts  # Query hook & MOCK_CURRENT_USER payload
│   └── get-schools.ts       # Onboarding school options dumb data (MOCK_SCHOOLS)
│
├── components/              # Feature-specific UI components (no hardcoded mock data)
│   ├── LoginForm.tsx
│   ├── LoginForm.test.tsx   # Component unit & interaction test
│   ├── SignUpModal.tsx
│   └── AuthButton.tsx
│
├── hooks/                   # Feature-specific custom hooks
│   ├── useAuth.ts
│   └── useAuth.test.ts      # Custom hook integration test with QueryClient
│
├── __tests__/               # Feature workflow integration test suite
│   └── auth-flow.test.tsx   # Multi-component feature user flow test
│
├── types/                   # PURE TypeScript types & interfaces ONLY (Zero runtime constants/arrays)
│   └── index.ts             # AuthUser, LoginCredentials, AuthResponse, SchoolOption
│
└── index.ts                 # Explicit Public API export boundary (Gatekeeper)
```

### 2.2. Routed Pages (`src/pages/<page-name>/`)

The composition layer for specific URL routes. Pages assemble presentational sections and integrate domain capabilities from `src/features/`.

```text
src/pages/landing/
├── components/              # Page-specific presentational sections
│   ├── HeroSection.tsx      # Presentational Hero banner
│   ├── FeaturesSection.tsx  # Value proposition showcase
│   └── PricingSection.tsx   # Tier comparison table
│
├── LandingPage.tsx          # Assembles layout sections and connects domain features
└── index.ts                 # Public route entry re-export for lazy loading
```

---

## 3. Strict Boundary Rules

### Rule 1: The `index.ts` Gatekeeper
Only items exported from `src/features/<feature-name>/index.ts` or `src/pages/<page-name>/index.ts` can be imported externally. Internal deep imports are strictly prohibited.

```typescript
// ✅ Good: Consuming via public API contract
import { AuthButton, useAuth } from '@/features/auth';
import LandingPage from '@/pages/landing';

// ❌ Prohibited: Piercing through internal subdirectories
import { LoginForm } from '@/features/auth/components/LoginForm';
```

### Rule 2: Colocation Over Centralization
- If a component, hook, or type is used by a single feature, **it must remain inside that feature**.
- Page-specific visual sections (e.g. `HeroSection`, `PricingSection` on `landing`) **belong in `src/pages/<page>/components/`**, not in `src/components/ui/`.
- Only promote code to `src/components/` or `src/hooks/` when it is genuinely shared by two or more independent domains.

### Rule 3: Unidirectional Dependency Flow
- **Pages may import from Features & Shared code**: `src/pages/*` imports from `src/features/*`, `src/components/ui/`, `src/hooks/`, `src/stores/`.
- **Features MUST NEVER import from Pages**: Features are portable domain engines with zero knowledge of route pages.
- **Shared code MUST NEVER import from Features or Pages**: `src/components/ui/` and `src/utils/` must remain domain-agnostic.
- **Page Isolation**: A page module must never import private components from a sibling page (`src/pages/landing` ⇸ `src/pages/dashboard`).

### Rule 4: Dumb Data Isolation (Pure Types & Mock Segregation)
- **`types/` must be 100% pure**: It must contain ONLY TypeScript types (`type`, `interface`). Never place runtime arrays, mock constants, or dummy data inside `types/index.ts`. Placing `const MOCK_DATA = [...]` in `types/` pollutes type bundles, prevents tree-shaking, and breaks code isolation.
- **`api/` is the sole mock repository**: Any simulated network payload, dumb list, or mock entity collection belongs exclusively in `src/features/<feature>/api/` alongside the async caller functions that simulate network responses.
- **`components/` never hardcode dumb data**: Components must receive data through props, custom hooks, or API callers; never hardcode 50-item dummy arrays directly inside visual components.

```typescript
// ❌ Prohibited: Placing runtime dumb data inside types/
// src/features/auth/types/index.ts
export const SCHOOL_OPTIONS = ['HUST', 'VNU', 'FPT']; // VIOLATION!
export type SchoolOption = typeof SCHOOL_OPTIONS[number];

// ✅ Good: Pure types in types/, dumb data and API callers in api/
// src/features/auth/types/index.ts
export type SchoolOption = string;

// src/features/auth/api/get-schools.ts
export const MOCK_SCHOOL_OPTIONS = ['HUST', 'VNU', 'FPT'] as const;
export async function getSchools(): Promise<readonly string[]> {
  return new Promise((resolve) => setTimeout(() => resolve(MOCK_SCHOOL_OPTIONS), 60));
}
```

### Rule 5: Test Colocation Architecture (Feature-Driven Testing)
- **Feature-Level Colocation**: Tests that exercise domain logic or UI must reside directly inside the feature folder:
  - `src/features/<feature>/api/*.test.ts`: Unit tests for API callers, Zod parsing, and Query keys.
  - `src/features/<feature>/components/*.test.tsx`: Visual component unit and interaction tests.
  - `src/features/<feature>/hooks/*.test.ts`: Custom hook integration tests with QueryClient wrappers.
  - `src/features/<feature>/__tests__/*.test.tsx`: Multi-component user journey integration tests within the feature.
- **Shared UI Primitives**: Unit tests for generic design system components reside in `src/components/ui/*.test.tsx`.
- **Global Test Infrastructure**: Store shared test utilities, custom `render` wrapper, and Mock Service Worker (MSW) handlers in `src/tests/` (`setup.ts`, `test-utils.tsx`, `mocks/`).
- **Cross-Feature E2E Tests**: Multi-page user journey tests spanning across different domains reside in `tests/e2e/` (Playwright) at the repository root.
- **Zero-Cost Deletion Benefit**: When a feature domain is refactored, moved, or deleted, deleting `src/features/<feature>/` removes 100% of its related tests without leaving orphaned test files in a centralized test folder.
