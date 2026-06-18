# UI Rules

Human-readable constraints for everything user-facing. The machine-readable design
system (tokens, theme values, component registry) lives in `context/ui-context.md` — this
file says how to use it. Builders read both before any UI task; QA enforces both.

## The golden rules

1. **Tokens only.** Zero hardcoded hex colors, pixel font sizes, or magic spacing
   values. Every visual value comes from a token in `context/ui-context.md`. This is a QA
   rejection, not a style nit.
2. **No static data as props.** Nav items, footer links, and other build-time
   lists live INSIDE the component as explicit JSX — pages pass only dynamic data.
   No `.map()` over static literal arrays.
3. **Pages orchestrate, components render.** A page composes sections; it never
   contains section JSX inline. Extract to `components/<feature>/<Concern>.tsx`.
4. **Check the registry first.** Before building any component, check the registry
   in `context/ui-context.md`. If it exists, reuse it exactly; if you build it, register it.

## Layout

- _Max content width: e.g. `1440px`, centered, horizontal padding per spacing tokens._
- _Breakpoints: e.g. mobile < 640 / tablet < 1024 / desktop._
- No horizontal scroll at any breakpoint; base font ≥ 16px on mobile.

## Typography

- _Font family loaded once (e.g. via `next/font`), exposed as a token._
- _Define the scale: heading / body / caption levels with weight and size tokens._

## Components

For each core component (buttons, cards, inputs, badges, tables, navbars), follow
the spec in `context/ui-context.md` exactly. Required interaction states for anything
interactive: default, hover, focus-visible, disabled, loading. Required data
states for anything async: loading, empty, error, success.

## Accessibility (blocking)

- Text contrast ≥ 4.5:1; visible focus styles; full keyboard reachability.
- Semantic HTML first; alt text on images; labels on inputs; aria only where
  semantics can't express it.
- Touch targets ≥ 44×44px; controls disabled while their async action runs.

## Motion and performance

- Respect `prefers-reduced-motion`.
- Images: responsive sizes, modern formats, lazy-load below the fold.
- Prevent layout shift: reserve space for async content.

## Do not

- Use raw utility color classes where a token exists.
- Introduce a new visual pattern when the registry already has one for the job.
- Add a component library or icon set not listed in `context/ui-context.md`.
