# UI Remediation and Bug-Fixing Directives

## 1. Surgical Modification Standard
- Never rewrite entire component files when fixing visual or styling defects.
- Use line-targeted edits (`replace_file_content` or `multi_replace_file_content`) to address the specific root cause.
- Preserve all existing imports, docstrings, and comments unrelated to the bug fix.

## 2. Design Token Integrity
- Enforce 100% semantic Tailwind CSS tokens (`bg-primary`, `text-muted-foreground`, `p-4`, `rounded-md`).
- **PROHIBITED**: Arbitrary bracket values (e.g. `bg-[#2a75d3]`, `w-[317px]`, `text-[13px]`, `p-[17px]`). Map every style to Tailwind theme utilities or CSS variables.

## 3. Layout & Geometry Resilience
- **Flexbox Min-Width**: Always set `min-w-0` on flex items that contain truncated text, code, or headings to prevent container blowout.
- **Horizontal Overflow**: Root out unintended page-wide horizontal scrollbars. Avoid band-aid `overflow-x-hidden` on body if an unconstrained child container is the culprit.
- **Z-Index Scale**: Adhere strictly to the design system z-index scale:
  - `z-10` / `z-20`: Dropdowns, popovers, tooltips
  - `z-30`: Sticky headers, top navigation
  - `z-40`: Modal/drawer backdrops
  - `z-50`: Modal dialogs, sheets, alert dialogs
  - `z-60`: Toast notifications

## 4. Headless State & Public Boundary Isolation
- UI repair must never mutate server-state hooks (`useQuery`, `useMutation`), Zustand stores, or API client callers.
- UI components receive data through props or custom feature hooks; trigger actions via hook mutation callbacks.
- Never modify or remove `data-testid` attributes used by automated test suites.

## 5. Accessibility & Contrast Verification
- All interactive controls (`button`, `a`, `input`, `select`) must have visible `:focus-visible` rings for keyboard navigation.
- Validate color contrast using the local checker: `python3 .agents/plugins/frontend/skills/antislop-human/contrast-check.py <fg_hex> <bg_hex>`.
- Enforce WCAG AA standard: minimum 4.5:1 for standard text, 3:1 for large text (18px+ or 14px bold) and graphical icons.

## 6. Content & Data Immutability During UI Redesign (Zero Content Drift)
- **STRICT MANDATE**: When tasked with redesigning, restyling, or modernizing UI components, **NEVER** alter, invent, translate, or remove existing text, titles, subtitles, placeholders, labels, tags, numbers, or mock data.
- The redesign mandate applies strictly to **layout geometry, visual hierarchy, surface contrast, spacing, typography tokens, and micro-interactions**.
- Preserve every single copy string verbatim unless the user explicitly requests copy editing.

## 7. Light Theme Visual Prominence & The Canvas-Contrast Rule
- **The Background-Matching Trap ("Bị trùng màu với background")**: When building a Light Theme hero banner, showcase header, or primary focal card, **NEVER** use the exact same hue or flat white/slate as the surrounding page canvas (e.g. placing a flat white/slate-50 card on a `bg-slate-50` `#F8FAFC` page). This destroys visual hierarchy, flattening the section so it fails to stand out.
- **The Tinted-Surface Pattern**:
  - Differentiate prominent light hero containers using a **soft, luminous brand tint** (e.g., Occupify Azure/Ice-Blue wash `from-[#EAF2FA] via-[#F0F6FC] to-[#DDECF9]` with defined border `border-[#BFDBFE]`).
  - Place **pure white elevated elements inside it** (e.g., pure white search bar with `shadow-md border border-slate-200`, pure white trust chips with `shadow-2xs`). This creates an "island in the stream" effect where the interactive search box pops with high visual contrast.
- Ensure light heroes always possess distinct perimeter definition (`border`, subtle depth, soft atmospheric glow) so the hero is the unmistakable visual anchor of the page.

## 8. Authentic Freelance Job Market Design Standards
- **Authentic Marketplace Density (Upwork / Contra / Malt)**:
  - Deliver clear, high-density metadata: Escrow verification badges, rate transparency (fixed vs hourly tags), commitment hours/day, delivery duration, skill chips, and competition signals (`Ít cạnh tranh / <5 Bids`).
  - Emphasize trust and safety: Escrow protection guarantees, milestone release indicators, and verified client rating badges.
  - Sticky Freelancer Workbench: Keep sidebar widgets (Profile readiness, Connects allowance, Escrow wallet, Contract status) sticky (`lg:sticky lg:top-20`) so key tools remain accessible while scrolling through jobs.
- **Anti-Slop Adherence**:
  - **FORBIDDEN**: Random purple/cyan radial gradient mesh orbs floating on page backgrounds.
  - **FORBIDDEN**: Generic AI icons (Sparkles, Magic Wands, Robots) used as default badges or buttons. Use domain-accurate icons (`ClockCountdownIcon`, `CoinsIcon`, `BriefcaseMetalIcon`, `ShieldCheckIcon`).
  - **FORBIDDEN**: Excessive glassmorphism blur stacked across every card, header, and sidebar.

