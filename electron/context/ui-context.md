# UI Context — Design System

Machine-readable design system: tokens, theme, and the living component registry.
`context/ui-rules.md` says how to use these values; this file IS the values. Lock this
before the first UI task is built — retrofitting a design system is how features get
rebuilt (lesson learned from previous projects).

## Theme

- Mode: **light | dark — both supported; light is the default.** Dark mode toggles
  via the `dark` class on `<html>` (`@variant dark`); window chrome respects the OS
  theme.
- Personality: **clean, professional, desktop-native** — calm neutrals, a single
  accent, generous whitespace, dense-enough for productivity views. Matches the web
  and mobile clients of the same backend.
- Reference: the current renderer (`app-shell.tsx`, `home-page.tsx`) uses a neutral
  surface + white cards aesthetic; the tokens below codify and replace those ad-hoc
  Tailwind palette classes (`neutral-*`).

## Tokens

Single source of truth. Defined once in `src/renderer/src/styles/globals.css` via
Tailwind v4 `@theme`, referenced everywhere by name — components never restate values.
Today `globals.css` only holds `@import "tailwindcss"`; the first UI task adds this
`@theme` block (and the dark overrides) before using any of it.

```css
@import "tailwindcss";

@theme {
  /* Colors — light defaults */
  --color-background: #fafafa;          /* app surface (neutral-50)        */
  --color-surface: #ffffff;             /* cards / panels                   */
  --color-foreground: #171717;          /* primary text (neutral-900)       */
  --color-muted: #f5f5f5;               /* subtle fills (neutral-100)       */
  --color-muted-foreground: #737373;    /* secondary text (neutral-500)     */
  --color-border: #e5e5e5;              /* borders / dividers (neutral-200) */

  --color-accent: #4f46e5;              /* primary actions / active nav     */
  --color-accent-foreground: #ffffff;   /* text on accent                   */

  --color-success: #16a34a;
  --color-warning: #d97706;
  --color-destructive: #dc2626;

  /* Chart / data-viz series — neutral grayscale ramp */
  --chart-1: #d4d4d4;
  --chart-2: #737373;
  --chart-3: #525252;
  --chart-4: #404040;
  --chart-5: #262626;

  /* Typography */
  --font-sans: "Inter", system-ui, -apple-system, "Segoe UI", sans-serif;
  --font-mono: "JetBrains Mono", ui-monospace, monospace;

  /* Radii */
  --radius-sm: 6px;
  --radius-md: 8px;     /* default card/control radius (rounded-lg today)  */
  --radius-lg: 12px;
}

/* Dark theme overrides */
@layer base {
  .dark {
    --color-background: #0a0a0a;
    --color-surface: #171717;
    --color-foreground: #fafafa;
    --color-muted: #262626;
    --color-muted-foreground: #a3a3a3;
    --color-border: #2e2e2e;
    --color-accent: #6366f1;
  }
}
```

### Color usage guide

| Use | Token |
| --- | --- |
| App / window background | `--color-background` |
| Cards, panels, popovers | `--color-surface` |
| Primary text | `--color-foreground` |
| Secondary / metadata text | `--color-muted-foreground` |
| Subtle fills, hover backgrounds | `--color-muted` |
| Primary actions / active nav / focus ring | `--color-accent` (+ `--color-accent-foreground`) |
| Borders / dividers | `--color-border` |
| Success / warning / error states | `--color-success` / `--color-warning` / `--color-destructive` |
| Chart / data-viz series | `--chart-1` … `--chart-5` |

## Typography scale

Loaded once (Inter via a bundled font or system fallback), exposed as `--font-sans`.

| Style | Size / weight | Use |
| --- | --- | --- |
| Heading 1 | 24px / 600 (`text-2xl font-semibold`) | page titles |
| Heading 2 | 18px / 600 | section titles |
| Body | 14px / 400 | default desktop text |
| Body strong | 14px / 500 | emphasis, labels |
| Caption | 12px / 400 | metadata, hints, table meta |
| Mono | 13px / 400 (`--font-mono`) | ids, code, logs |

## Spacing

Scale (px): **4 / 8 / 12 / 16 / 24 / 32 / 48 / 64** (Tailwind `1 / 2 / 3 / 4 / 6 / 8 / 12 / 16`).

- Card padding: 16 (compact) / 24 (default).
- Section vertical gaps: 24 (`space-y-6`).
- Grid/list item gaps: 12 (`gap-3`).
- App shell content padding: 32 (`p-8`), centered, `max-w-5xl`.
- Form field gaps: 16.

## Layout

- Window default size 1200×800 (`src/main/index.ts`); the renderer is responsive
  within it. No horizontal scroll at any window width.
- Centered content column: `max-w-5xl` for primary views; full-width for data tables.
- Breakpoints (Tailwind defaults): `sm` 640 / `md` 768 / `lg` 1024 — desktop-first,
  but layouts must not break when the window is narrowed.

## Component registry (living document)

Updated by Builders after every new component; checked by Builders before every new
component; audited by QA and /review. One entry per component:

```markdown
### ComponentName
- Path: `shared/components/ui/component-name.tsx` (or `features/<name>/components/…`)
- Purpose: one line
- Variants/props: the public API surface
- States: default / hover / focus-visible / disabled / loading (interactive)
- Used by: pages/features using it
```

_(empty — first entry added by the first UI task. `AppShell` and `HomePage` are the
current placeholder shell, not yet tokenized design-system components — the first UI
task tokenizes them.)_

## Invariants

1. No visual value outside this file (no hardcoded hex/px/magic spacing in components).
2. New tokens are added here first, then used — never inline-then-backfill.
3. The registry never goes stale: building a component and not registering it is a
   QA rejection.
4. Conditional classes go through `cn()` (`shared/libs/utils.ts`); icons come from
   `lucide-react` — no other icon set without a token-registry entry.
