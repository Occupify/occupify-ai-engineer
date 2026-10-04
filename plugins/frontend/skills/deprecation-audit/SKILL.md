---
name: deprecation-audit
description: Automated third-party API deprecation scanning, official documentation verification, and active replacement consultation. Use to audit codebase for @deprecated symbols, inspect .d.ts signatures, query official documentation, and safely update imports.
---

# API Deprecation Audit & Zero-Guess Migration Skill

This skill delivers automated detection, official documentation verification, and clean remediation for deprecated symbols (`@deprecated`) across React, TypeScript, and third-party UI/icon libraries.

## Skill Specifications & Manifests

- **Configuration & Triggers**: [config.json](./config.json)  
  Defines triggers, agent roles (`DeprecationAuditorAgent`), detection rules, and official documentation source hierarchies.
- **Auditing Directives & Rules**: [system_prompt.md](./system_prompt.md)  
  Contains operational directives, zero-guess protocols, official source requirements, and remediation procedures.

## Core Capabilities

1. **AST-Based Deprecation Scanning**: Uses the TypeScript Compiler API (`TypeChecker.getSymbolAtLocation` + `symbol.getJsDocTags()`) to inspect every imported identifier, component, and hook without false positives.
2. **Official Source Verification**: Cross-references deprecated symbol notices against installed `.d.ts` declaration files and official library portals (e.g. `phosphoricons.com`, `react.dev`, `tanstack.com`).
3. **Automated Replacement**: Replaces obsolete APIs with current LTS/active alternatives (e.g. `@phosphor-icons/react`'s deprecated `Lock` $\rightarrow$ `LockSimple` / `LockKey` / `LockIcon`).
4. **Build & CI Gatekeeping**: Provides a runnable CLI tool (`pnpm scan:deprecations`) suitable for pre-commit hooks and CI/CD pipelines.

## Quick Execution Summary

1. **Scan**: Run the workspace deprecation scanner:
   ```bash
   pnpm scan:deprecations
   ```
2. **Analyze**: Ingest scanner output: file path, line number, deprecated symbol, declaration package, and JSDoc deprecation notice.
3. **Verify**: Check the package `.d.ts` declaration file or official documentation for the recommended replacement. **Never guess API names.**
4. **Remediate**: Update imports and JSX/TS usages to the active symbol.
5. **Verify Clean**: Re-run `pnpm scan:deprecations && pnpm build` to confirm 0 deprecations and 0 compilation errors.
