---
name: react
description: Production React + Vite architecture and design pattern consultation skill. Use when architecting scalable React SPAs, configuring Vite 5/6 tooling, implementing feature-driven project structure (Bulletproof React), designing Compound Components or Headless UI, separating server/client state (TanStack Query + Zustand), and auditing code for React antipatterns.
---

# Production React + Vite Workspace Skill

This skill delivers enterprise-grade architecture consultation, design pattern implementation, and code auditing for modern React applications bundled with Vite.

## Skill Specifications & Manifests

- **Configuration & Triggers**: [config.json](./config.json)  
  Defines triggers, agent roles (`ReactArchitectAgent`, `PatternAuditorAgent`), source hierarchies, and architectural policies.
- **Architectural Directives & Rules**: [system_prompt.md](./system_prompt.md)  
  Contains operational directives, strict TypeScript constraints, prohibited antipatterns, and code generation guidelines.
- **Consultation & Execution Protocol**: [workflow.md](./workflow.md)  
  Specifies the 5-phase consultation lifecycle, audit checklist, and verification protocols.

## Progressive Knowledge References

Detailed architectural reference manuals are located in `references/`:

- **[Project Structure & Modularity](./references/project-structure.md)**: Feature-First (Bulletproof React) layout, colocation rules, and strict `index.ts` boundary contracts.
- **[Vite 5/6 Tooling & Configuration](./references/vite-tooling.md)**: Native path aliases (`resolve.alias`), `manualChunks` vendor splitting, environment variable typing, and build optimization.
- **[React Design Patterns](./references/design-patterns.md)**: Compound Components, Headless UI integration (Radix/React Aria), Custom Hook encapsulation, and Slot composition.
- **[State Management Taxonomy](./references/state-management.md)**: Clear segregation between Server State (TanStack Query), Global UI State (Zustand), URL State, and Local UI State.
- **[Performance & Rendering](./references/performance-optimization.md)**: Dynamic route-based lazy loading (`React.lazy` + `Suspense`), memoization rules, and bundle inspection.

## Quick Execution Summary

1. **Scope Calibration**: Distinguish between workspace initialization (skeleton with `.gitkeep` & sample configs only) and full feature implementation. Never over-implement unrequested dummy logic.
2. **Architecture Discovery**: Ingest domain requirements, determine feature boundaries, and enforce feature-driven modularity.
3. **Pattern Formulation**: Select appropriate patterns (Compound Components for multi-part UI, Headless hooks for complex behavior, Zustand for shared UI state).
4. **Tooling & Setup**: Enforce robust `vite.config.ts` with explicit path aliases, typed environment variables, and optimized chunking.
5. **Code & Deprecation Audit**: Verify absence of antipatterns and ensure zero `@deprecated` library symbols (run `pnpm scan:deprecations`). Always consult official documentation for replacement APIs.
6. **Dumb Data & Mock Isolation**: Enforce 100% pure TypeScript types in `types/`. House all mock datasets exclusively in `api/` alongside async callers. Strictly prohibit hardcoded mock lists or mock tokens (`MOCK_AUTH_TOKENS`) in `components/`.
7. **Side-Effect & Store Encapsulation**: UI components must NEVER directly mutate client stores (e.g. `useAuthStore` action setters like `setTokens`, `clearTokens`). All store mutations, token management, cache sync, and session lifecycles MUST be strictly encapsulated within TanStack Query mutation hooks (`hooks/use-*.ts` `onSuccess` / `onSettled`). Pure API callers (`api/`) must remain pure async network callers without touching stores.
8. **Mandatory Dual-Agent Review**: Always enforce at least 2 distinct agent roles (`ReactArchitectAgent` + `PatternAuditorAgent`) to independently design and audit any code before final delivery.
