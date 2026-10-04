# Modern React Design Patterns

This guide provides concrete, production-ready implementations of core React design patterns used in enterprise web applications.

---

## 1. Compound Components Pattern

The Compound Components pattern allows multiple components to work together seamlessly by sharing implicit state via React Context, avoiding prop-drilling and rigid "Mega-Props" configurations.

### Production Example: Guarded Tabs Component

```tsx
// src/components/ui/tabs/tabs.tsx
import React, { createContext, useContext, useState, type ReactNode } from 'react';

interface TabsContextValue {
  activeTab: string;
  setActiveTab: (value: string) => void;
}

const TabsContext = createContext<TabsContextValue | null>(null);

// 🛡️ Error guard hook
function useTabsContext() {
  const context = useContext(TabsContext);
  if (!context) {
    throw new Error('Tabs compound subcomponents must be rendered within a <Tabs> provider.');
  }
  return context;
}

// 1. Root Component
export interface TabsProps {
  defaultValue: string;
  children: ReactNode;
  className?: string;
}

export function Tabs({ defaultValue, children, className }: TabsProps) {
  const [activeTab, setActiveTab] = useState(defaultValue);

  return (
    <TabsContext.Provider value={{ activeTab, setActiveTab }}>
      <div className={className}>{children}</div>
    </TabsContext.Provider>
  );
}

// 2. Tab List
export function TabList({ children, className }: { children: ReactNode; className?: string }) {
  return (
    <div role="tablist" className={`flex border-b border-gray-200 ${className || ''}`}>
      {children}
    </div>
  );
}

// 3. Tab Trigger
export interface TabTriggerProps {
  value: string;
  children: ReactNode;
  className?: string;
}

export function TabTrigger({ value, children, className }: TabTriggerProps) {
  const { activeTab, setActiveTab } = useTabsContext();
  const isActive = activeTab === value;

  return (
    <button
      role="tab"
      aria-selected={isActive}
      onClick={() => setActiveTab(value)}
      className={`px-4 py-2 font-medium transition-colors ${
        isActive ? 'border-b-2 border-blue-600 text-blue-600' : 'text-gray-500 hover:text-gray-700'
      } ${className || ''}`}
    >
      {children}
    </button>
  );
}

// 4. Tab Content
export interface TabContentProps {
  value: string;
  children: ReactNode;
  className?: string;
}

export function TabContent({ value, children, className }: TabContentProps) {
  const { activeTab } = useTabsContext();
  if (activeTab !== value) return null;

  return (
    <div role="tabpanel" className={`py-4 ${className || ''}`}>
      {children}
    </div>
  );
}

// Attach compound subcomponents
Tabs.List = TabList;
Tabs.Trigger = TabTrigger;
Tabs.Content = TabContent;
```

### Consumption
```tsx
<Tabs defaultValue="overview">
  <Tabs.List>
    <Tabs.Trigger value="overview">Tổng quan</Tabs.Trigger>
    <Tabs.Trigger value="pricing">Bảng giá</Tabs.Trigger>
  </Tabs.List>
  <Tabs.Content value="overview">Thông tin phòng trọ Occupify</Tabs.Content>
  <Tabs.Content value="pricing">Chi phí & Cọc thuê</Tabs.Content>
</Tabs>
```

---

## 2. Headless UI Pattern

Headless components handle state, keyboard interactions, and ARIA attributes without prescribing any visual styling.

### Integration with Primitives (e.g. Radix UI or Custom Headless Hooks)
```tsx
import * as DialogPrimitive from '@radix-ui/react-dialog';
import { type ReactNode } from 'react';

export function Modal({ trigger, title, children }: { trigger: ReactNode; title: string; children: ReactNode }) {
  return (
    <DialogPrimitive.Root>
      <DialogPrimitive.Trigger asChild>{trigger}</DialogPrimitive.Trigger>
      <DialogPrimitive.Portal>
        <DialogPrimitive.Overlay className="fixed inset-0 bg-black/50 backdrop-blur-sm" />
        <DialogPrimitive.Content className="fixed left-1/2 top-1/2 -translate-x-1/2 -translate-y-1/2 rounded-xl bg-white p-6 shadow-xl w-full max-w-lg">
          <DialogPrimitive.Title className="text-xl font-semibold mb-4">{title}</DialogPrimitive.Title>
          {children}
          <DialogPrimitive.Close className="absolute top-4 right-4 text-gray-400 hover:text-gray-600">
            ✕
          </DialogPrimitive.Close>
        </DialogPrimitive.Content>
      </DialogPrimitive.Portal>
    </DialogPrimitive.Root>
  );
}
```

---

## 3. Custom Hook Encapsulation

Encapsulate complex UI logic, browser APIs, or event listeners into single-responsibility hooks.

```typescript
// src/hooks/use-disclosure.ts
import { useState, useCallback } from 'react';

export function useDisclosure(initialState = false) {
  const [isOpen, setIsOpen] = useState(initialState);

  const open = useCallback(() => setIsOpen(true), []);
  const close = useCallback(() => setIsOpen(false), []);
  const toggle = useCallback(() => setIsOpen((prev) => !prev), []);

  return { isOpen, open, close, toggle } as const;
}
```

---

## 4. Slot Composition Pattern

Avoid passing boolean flags for icons or titles. Favor composition slots:

```tsx
interface CardProps {
  headerSlot?: React.ReactNode;
  footerSlot?: React.ReactNode;
  children: React.ReactNode;
}

export function Card({ headerSlot, footerSlot, children }: CardProps) {
  return (
    <div className="rounded-lg border border-gray-200 bg-white shadow-sm overflow-hidden">
      {headerSlot && <div className="border-b px-6 py-4 bg-gray-50">{headerSlot}</div>}
      <div className="p-6">{children}</div>
      {footerSlot && <div className="border-t px-6 py-3 bg-gray-50">{footerSlot}</div>}
    </div>
  );
}
```
