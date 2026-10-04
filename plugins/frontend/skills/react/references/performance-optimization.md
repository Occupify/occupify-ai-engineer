# Performance & Rendering Optimization

This reference covers practical techniques to eliminate rendering bottlenecks and optimize bundle sizes in React + Vite production apps.

---

## 1. Route-Based Code Splitting

Never bundle all route components into the initial bundle. Use dynamic imports with `React.lazy` and `<Suspense>`:

```tsx
// src/app/router.tsx
import { lazy, Suspense } from 'react';
import { createBrowserRouter } from 'react-router-dom';
import { AppLayout } from '@/components/layout/app-layout';
import { FullPageSpinner } from '@/components/feedback/spinner';

// Lazy load route pages
const HomePage = lazy(() => import('@/features/home').then((m) => ({ default: m.HomePage })));
const PropertiesPage = lazy(() => import('@/features/properties').then((m) => ({ default: m.PropertiesPage })));
const PropertyDetailPage = lazy(() => import('@/features/properties').then((m) => ({ default: m.PropertyDetailPage })));

export const router = createBrowserRouter([
  {
    path: '/',
    element: <AppLayout />,
    children: [
      {
        index: true,
        element: (
          <Suspense fallback={<FullPageSpinner />}>
            <HomePage />
          </Suspense>
        ),
      },
      {
        path: 'properties',
        element: (
          <Suspense fallback={<FullPageSpinner />}>
            <PropertiesPage />
          </Suspense>
        ),
      },
      {
        path: 'properties/:id',
        element: (
          <Suspense fallback={<FullPageSpinner />}>
            <PropertyDetailPage />
          </Suspense>
        ),
      },
    ],
  },
]);
```

---

## 2. Practical Memoization Rules

### When to use `useMemo` and `useCallback`
- **Heavy calculations**: Sorting, filtering, or aggregating large datasets (> 100 items).
- **Referential equality dependencies**: Passing an object/function to a custom hook's dependency array.
- **Virtualized list row renderers**: Ensuring item props do not invalidate during fast scrolls.

### When NOT to memoize
- Wrapping every primitive string or boolean in `useMemo`.
- Wrapping simple click handlers that only call `setState` unless passed to a `React.memo` child.
- Premature optimization before profiling in React DevTools Profiler.

---

## 3. List Virtualization for Dense Feeds

For large data feeds (e.g. browsing hundreds of apartment listings in Occupify), render only the visible viewport DOM nodes with `@tanstack/react-virtual`:

```tsx
// src/features/properties/components/virtualized-property-list.tsx
import { useRef } from 'react';
import { useVirtualizer } from '@tanstack/react-virtual';
import { PropertyCard } from './property-card';
import type { Property } from '../types';

export function VirtualizedPropertyList({ properties }: { properties: Property[] }) {
  const parentRef = useRef<HTMLDivElement>(null);

  const rowVirtualizer = useVirtualizer({
    count: properties.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 180, // Estimated item height in px
    overscan: 5,
  });

  return (
    <div ref={parentRef} className="h-[800px] overflow-auto border rounded-lg">
      <div
        className="w-full relative"
        style={{ height: `${rowVirtualizer.getTotalSize()}px` }}
      >
        {rowVirtualizer.getVirtualItems().map((virtualRow) => {
          const property = properties[virtualRow.index];
          return (
            <div
              key={virtualRow.key}
              className="absolute top-0 left-0 w-full"
              style={{
                height: `${virtualRow.size}px`,
                transform: `translateY(${virtualRow.start}px)`,
              }}
            >
              <PropertyCard property={property} />
            </div>
          );
        })}
      </div>
    </div>
  );
}
```

---

## 4. Bundle Inspection & Auditing

Add an analyze script to `package.json`:
```json
{
  "scripts": {
    "build:analyze": "vite build --mode analyze"
  }
}
```

Audit criteria:
1. Ensure no single vendor chunk exceeds 300 kB (gzipped).
2. Look out for duplicate dependencies (e.g. multiple versions of `lodash` or `date-fns`).
3. Ensure tree-shaking is active (importing specific icons e.g. `lucide-react/dist/esm/icons/...` or using named imports verified by bundler).
