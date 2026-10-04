# API Deprecation Audit & Verification Directives

This document defines the operational directives, source verification hierarchy, and remediation procedures for auditing and eliminating deprecated library symbols.

---

## 1. Top-Level Operational Directives

1. **Zero Deprecation Tolerance**:
   - Production code must never import or consume symbols flagged with `@deprecated` in library type declarations.
   - Deprecated symbols trigger runtime warnings, break across minor/major version bumps, and degrade codebase reliability.

2. **Zero-Guess Replacement Protocol**:
   - **Never guess** replacement symbol names.
   - When a symbol is flagged as deprecated:
     - First, read the JSDoc `@deprecated` annotation directly from the package's `.d.ts` file in `node_modules/`.
     - Second, cross-reference with official library documentation (e.g. `phosphoricons.com`, `react.dev`, `tanstack.com/query`).
     - Confirm the active symbol exists and is exported before editing imports.

3. **Mandatory Post-Code Verification**:
   - After authoring or refactoring code involving third-party libraries, always run the workspace deprecation scanner:
     ```bash
     pnpm scan:deprecations
     ```
   - If the scanner exits with non-zero status, remediate every reported issue before presenting work to the user.

---

## 2. Common Library Deprecation Matrix

| Library | Deprecated Symbol | Reason | Official Non-Deprecated Replacement |
| :--- | :--- | :--- | :--- |
| `@phosphor-icons/react` | `Lock` | Ambiguous naming in v2.x | `LockSimple`, `LockKey`, `LockIcon` |
| `@phosphor-icons/react` | `CheckFat` | Replaced by standard weight variants | `Check`, `CheckCircle` |
| `react` | `React.FC` (with implicit children) | Implicit children antipattern in modern React | Explicit prop types with `React.ReactNode` |
| `react-router-dom` | `useHistory` | Deprecated since React Router v6 | `useNavigate` |
| `@tanstack/react-query` | `onSuccess` / `onError` in `useQuery` | Removed in TanStack Query v5 | Return handling in `useEffect` or state deriver / mutation callbacks |

---

## 3. Step-by-Step Remediation Workflow

```text
[Source Code Changes]
        │
        ▼
[Run Scanner: pnpm scan:deprecations]
        │
   ┌────┴──────────────────────────┐
   ▼                               ▼
[Found 0 Issues]            [Issues Detected]
   │                               │
   ▼                               ▼
[Proceed to Build]         [Inspect Symbol & JSDoc Tag]
                                   │
                                   ▼
                           [Check Official Docs / .d.ts]
                                   │
                                   ▼
                           [Replace with Active Symbol]
                                   │
                                   ▼
                           [Re-scan until Clean]
```
