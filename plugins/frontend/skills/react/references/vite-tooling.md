# Vite 5/6 Tooling & Configuration Best Practices

This reference covers optimal configuration for building fast, type-safe, and cache-efficient React applications with Vite.

---

## 1. Production `vite.config.ts`

```typescript
import path from 'node:path';
import { defineConfig, type PluginOption } from 'vite';
import react from '@vitejs/plugin-react';
import { visualizer } from 'rollup-plugin-visualizer';

export default defineConfig(({ mode }) => {
  const isAnalyze = mode === 'analyze';

  return {
    plugins: [
      react(),
      isAnalyze && (visualizer({ open: true, filename: 'dist/stats.html', gzipSize: true }) as PluginOption),
    ].filter(Boolean),

    resolve: {
      alias: {
        '@': path.resolve(__dirname, './src'),
      },
    },

    server: {
      port: 3000,
      open: true,
      proxy: {
        '/api': {
          target: 'http://localhost:8080',
          changeOrigin: true,
          secure: false,
        },
      },
    },

    build: {
      target: 'es2022',
      sourcemap: mode === 'development',
      rollupOptions: {
        output: {
          manualChunks(id) {
            // Isolate large vendor dependencies for optimal HTTP caching
            if (id.includes('node_modules')) {
              if (id.includes('react') || id.includes('react-dom') || id.includes('react-router-dom')) {
                return 'vendor-react';
              }
              if (id.includes('@tanstack/react-query')) {
                return 'vendor-query';
              }
              if (id.includes('lucide-react') || id.includes('@radix-ui')) {
                return 'vendor-ui';
              }
            }
          },
        },
      },
    },
  };
});
```

---

## 2. Synchronized `tsconfig.json`

To ensure full autocomplete, compiler correctness, and editor alignment without extra plugins, match `tsconfig.json` with Vite's aliases:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "useDefineForClassFields": true,
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,

    /* Bundler mode */
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx",

    /* Linting & Type Safety */
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,

    /* Path Aliases */
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["src"]
}
```

---

## 3. Type-Safe Environment Variables

Declare custom `VITE_` environment variables in `src/vite-env.d.ts`:

```typescript
/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_API_BASE_URL: string;
  readonly VITE_APP_ENV: 'development' | 'staging' | 'production';
  readonly VITE_ENABLE_ANALYTICS: string;
}

interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

### Accessing via Config Object (Prevent direct `import.meta.env` leaks)
```typescript
// src/config/env.ts
export const env = {
  apiBaseUrl: import.meta.env.VITE_API_BASE_URL || '/api',
  isProduction: import.meta.env.VITE_APP_ENV === 'production',
  enableAnalytics: import.meta.env.VITE_ENABLE_ANALYTICS === 'true',
} as const;
```
