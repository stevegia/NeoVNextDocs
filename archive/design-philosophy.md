# NeoVLab Design Philosophy
### "GITS Section 9 Dark Teal" — A Living Document

*Written by someone who has stared at this interface long enough to have opinions. Strong ones.*

---

## 1. The Aesthetic Vision

NeoVLab is not a dashboard. Do not call it a dashboard. Dashboards are beige rectangles with rounded corners and a SaaS pricing table. NeoVLab is a **cyberpunk operations terminal** — the kind of thing Major Kusanagi would run on a rig she built herself in a server room in 2034 Neo-Tokyo. Every visual decision should answer the question: *does this look like it belongs in Section 9?*

The reference aesthetic is Ghost in the Shell: Stand Alone Complex — not the neon-vomit of Cyberpunk 2077, not the clean minimalism of modern SaaS, not the retro-kitsch of synthwave posters. It is **cold precision** with **electric accents**. It is dark glass at night, backlit by teal phosphorescence. It is information-dense but never cluttered. It is a tool built for someone who knows exactly what they are doing.

**The emotional profile of the interface:**
- **Confident.** No apologetic rounded corners on every element. Structure has weight.
- **Precise.** Density is a virtue. Compact mode should feel like a professional instrument, not a cramped one.
- **Alive.** Glass surfaces catch imaginary light. Accents glow. The background breathes.
- **Trustworthy.** Data is legible. Hierarchy is clear. Nothing is decorative at the expense of function.

The five themes (Cyberpunk Teal, Neon Purple, Synthwave Red, Ghost White, Matrix Green) are not skins — they are moods. Each theme should feel like the same Section 9 terminal running in a different unit's ops center: same discipline, different light signature. The CSS variable system must guarantee this.

---

## 2. Core Design Tokens

Every color, spacing unit, and radius in NeoVLab flows through CSS custom properties. **Hardcoded hex values in component code are a code smell.** They break theme switching, they resist density scaling, and they signal that the author was in a hurry and left a debt behind.

### 2.1 Surface and Glass Tokens

```
--glass-bg                rgba(255,255,255,0.06)   Base glass fill — use for panels and cards
--glass-bg-hover          rgba(255,255,255,0.09)   Hover state on interactive glass surfaces
--glass-blur              16px                      Standard backdrop-filter blur radius
--glass-border            rgba(255,255,255,0.09)   Side and bottom border on glass elements
--glass-border-top        rgba(255,255,255,0.18)   Top border — catches the "light source above" illusion
--glass-border-strong     rgba(255,255,255,0.20)   Focused or highlighted border state
--glass-inset-highlight   rgba(255,255,255,0.12)   box-shadow inset highlight for depth
```

The glass effect is not just `backdrop-filter: blur(16px)`. The trio of `blur + saturate(1.6) + brightness(1.04)` is load-bearing. Remove any one and the surfaces lose their luminosity. The `::before` pseudo-element on `.glass-panel` and `.glass-card` adds a subtle vertical luminance gradient (white-to-transparent, top 28%) that simulates overhead light bouncing off the glass. Do not remove it.

### 2.2 Accent Tokens (per-theme)

```
--accent-primary          Primary brand color (#22d3ee in teal default)
--accent-secondary        Darker variant for gradients and secondary emphasis
--accent-glow             rgba version of primary at ~35% — used for glows, halos, tinted bg fills
--accent-halo             rgba version of primary at ~15% — outermost glow ring
--accent-gradient         The diagonal gradient used on primary buttons and active indicators
--accent-gradient-hover   Lighter version for hover states
```

These six tokens are the only accent-related things you should ever reference. If you find yourself writing `text-cyan-400` or `border-purple-500` in component code, stop. You are fighting the theme system. The theme system will win, and your component will look broken on the other four themes.

The `--accent-glow` token deserves special attention: it is used both as a `box-shadow` color and as a tinted background fill (e.g., the count badge in job lane headers: `background: var(--accent-glow)`). It reads as a very faint tinted glass, which is exactly right.

### 2.3 Background Tokens

```
--bg-base                 Deepest background (#03080f in teal)
--bg-surface              Slightly lighter surface for nested containers
--bg-overlay-from         Scrim gradient top — used by .aero-bg::after
--bg-overlay-to           Scrim gradient bottom — ensures legibility over scene images
```

The `--bg-overlay-from/to` pair is what keeps the interface legible over scene backgrounds. Each theme has its own tinted overlay — the teal theme uses a blue-dark scrim, the synthwave-red theme uses a near-black with a red tint, and so on. This is not just an overlay; it is the mechanism by which the theme "infects" the background scene.

### 2.4 Typography Tokens

```
--text-primary            rgba near-white at ~92% opacity
--text-secondary          rgba tinted lighter color at ~62% opacity
--text-muted              rgba desaturated at ~45% opacity
```

Three tiers of text, no more. Every string of text in this UI should be classifiable as one of these three: primary (data, labels the user needs to act on), secondary (supporting context), or muted (metadata, timestamps, subtle hints). Use `--text-muted` sparingly — it should feel like the text is trying not to call attention to itself.

### 2.5 Shadow Tokens

```
--shadow-glow             accent glow layered over accent halo — for active/hover states
--shadow-depth            deep card shadow + inset highlight — for panel elevation
--shadow-card             lighter card shadow — for cards inside panels
```

### 2.6 Danger Token

```
--danger                  #ef4444  — used for destructive actions and error states
--danger-glow             rgba(239,68,68,0.2) — used for danger button hover glow
```

Note: danger is theme-invariant. It is always red because red means stop in every ops center on Earth. Do not theme-ize danger colors.

---

## 3. Component Hierarchy

The glass system has three surface levels. Using the wrong level creates visual noise — surfaces that should recede instead pop, or surfaces that should hold content instead disappear.

### Level 1: `.glass-panel`
**Use for: top-level page containers, primary content areas, major layout sections.**

`glass-panel` is the heaviest surface. It has `--radius-panel` (16px), `--shadow-depth`, and the full `blur/saturate/brightness` backdrop filter stack. It is what the user's eye lands on first. Every page should have exactly one or two glass-panels — the main content area and maybe a side detail panel. More than two panels at the same level and the layout starts to feel like a game of Tetris.

### Level 2: `.glass-card`
**Use for: list items, info cards, collapsible sections, secondary content groups within a panel.**

`glass-card` uses `--radius-card` (12px) and `--shadow-card`. It has a hover state that lifts it slightly with a glow. This is the workhorse surface. Artifact cards, job cards, worker cards, integration status cards — all `glass-card`.

### Level 3: `.glass-input`
**Use for: text inputs, native selects that haven't been replaced with CustomSelect, number inputs.**

Dark (near-black at 0.2 opacity), blurred, with the standard glass border. Focus state adds a `box-shadow: 0 0 0 2px var(--accent-glow)` ring — this is the only "focus ring" pattern in use. Do not invent alternatives.

### The Button Split: `.glass-button` vs `Button` primitive

`.glass-button` (CSS class) renders with the `--accent-gradient` fill — it is always an accent-colored CTA. It is appropriate for primary actions, submit buttons, navigation shortcuts. Use it when you want the button to command attention.

The `Button` primitive (`src/components/primitives/Button.tsx`) offers four variants:
- `primary` — gradient fill, same as glass-button, use for the single most important action on a page
- `secondary` — glass-bg fill, for supporting actions
- `ghost` — transparent, for inline or contextual actions
- `danger` — red fill, for destructive irreversible actions

**The proliferation problem:** the codebase currently has at least three ad-hoc button patterns being used alongside these primitives: raw `<button className="px-2 py-1 text-xs rounded ...">` fragments in filter bars, inline `<button className="glass-button ...">` without using the Button primitive, and the RadioGroup internal button in SettingsAppearanceSection. This needs consolidation.

### `.catalog-glass-wrap`
A specialized variant for the media catalog container — heavier box-shadow (48px, 0.5 opacity), `overflow: hidden`, flexbox column. Not for general use. It exists because the catalog grid needs to function as a contained scroll region that floats over the background.

### `.topbar-capsule`
A modifier class applied over `.glass-panel` to force full pill rounding on the top navigation bar. The `!important` overrides are intentional — they lock the capsule geometry regardless of density padding. Do not apply this class to anything that is not the top navigation bar.

---

## 4. Typography System

NeoVLab does not currently specify an explicit font family import. This is debt. The interface relies on the operating system's default sans-serif, which on Windows 11 (the primary target platform) resolves to Segoe UI. This is actually acceptable — Segoe UI is clean, legible, and has good CJK coverage — but it is an accident, not a decision. A future improvement would be to explicitly declare an Inter or Geist font with appropriate fallbacks.

### Current Type Scale (density-driven)

| Token | Compact | Comfortable | Spacious | Role |
|---|---|---|---|---|
| `--density-font-sm` | 11px | 12px | 13px | Labels, metadata, badges |
| `--density-font-md` | 13px | 14px | 15px | Body text, inputs, button text |
| `--density-font-lg` | 15px | 16px | 18px | Section headings, modal titles |

Outside the density system, the following hardcoded Tailwind classes are in active use across pages and should be treated as the "header" scale:
- `text-2xl font-bold` — page-level H1 headings (all admin pages use this consistently)
- `text-xl font-semibold` — section-level H2 headings (SettingsStorageSection, SettingsInfrastructureSection, SettingsIntegrationsSection)
- `text-base font-semibold` — subsection H3 (SettingsAppearanceSection)
- `text-sm font-medium` — card headings, table labels

The CSS global rule sets `h1, h2 { font-weight: 300; letter-spacing: -0.015em }` and `h3 { font-weight: 400; letter-spacing: -0.01em }`. This applies when you use bare HTML heading elements. Tailwind overrides this when you use `font-bold` or `font-semibold` directly — meaning the Tailwind-classed headings and the bare-element headings in the same app have inconsistent weight. The CSS rule is aspirational (light-weight headings read elegantly in glass UIs) but it is being overridden by utility classes throughout. This is a known inconsistency.

### Typography Dos
- Use `text-[var(--text-primary)]` for actionable labels, data values, primary content
- Use `text-[var(--text-secondary)]` for supporting text, form labels, descriptions
- Use `text-[var(--text-muted)]` for timestamps, IDs, metadata, hints
- Use `font-mono` for job IDs, file paths, config keys, any machine-generated string
- Use `text-[var(--accent-primary)]` for brand text, active states, important highlights

### Typography Don'ts
- Do not use `opacity-60` as a substitute for `--text-secondary`. They look similar, but opacity is not themeable
- Do not use Tailwind gray/slate color classes (`text-gray-500`, `text-slate-400`) — the index.css applies overrides for these but it is still debt
- Do not use bare `<h1>` through `<h4>` without also specifying a Tailwind text-size class — the CSS resets will apply unexpected weights

---

## 5. Spacing System

### Density Tokens

NeoVLab has three density modes controlled by `data-density` on `<html>`. The density tokens scale padding, gaps, font sizes, icon sizes, row heights, and layout insets together. This means a single `setDensity("spacious")` call should reflow the entire interface coherently without any component needing to know about it.

| Token | Compact | Comfortable | Spacious | Use |
|---|---|---|---|---|
| `--density-padding-xs` | 2px | 4px | 6px | Tight inline spacing |
| `--density-padding-sm` | 4px | 6px | 10px | Button padding, small gaps |
| `--density-padding-md` | 6px | 10px | 14px | Input padding, card internals |
| `--density-padding-lg` | 8px | 14px | 20px | Panel padding, section gaps |
| `--density-gap` | 4px | 8px | 12px | flex/grid gaps within components |
| `--density-row-height` | 36px | 44px | 56px | Table rows, list items |
| `--density-icon-size` | 16px | 18px | 20px | Icon width/height via CSS |
| `--layout-inset` | 10px | 14px | 20px | Outer page margin |
| `--topbar-height` | 52px | 62px | 72px | TopBar height for sticky offset calculations |

### The `--panel-pin-top` Calculation

The `--panel-pin-top` token (`calc(topbar-height + layout-inset * 2 + 8px)`) is the offset for sticky side panels (like the job detail panel in JobsPage). It ensures the sticky panel top aligns below the topbar regardless of density mode. This is the correct pattern — any new sticky content panels should reference this token, not a hardcoded pixel offset.

### Layout Conventions

- Page containers: `p-8 max-w-5xl mx-auto` or `p-8 max-w-6xl mx-auto` (all admin pages use this — consistent)
- Section gaps: `space-y-12` between major Settings sections, `space-y-6` between subsections
- Card grids: use Tailwind's responsive grid utilities with gap derived from the context — `gap-3` or `gap-4` is the norm

**The `p-8` problem:** Most admin pages use `p-8` as their outer padding, hardcoded instead of using `px-[var(--layout-inset)]`. When density mode changes, the page container padding stays at 32px. The MediaDetailPage uses `px-[var(--density-padding-lg)]` which is the correct approach but creates inconsistency.

---

## 6. Color and Theme System

### Architecture

Themes are implemented as CSS custom property overrides scoped to `[data-theme="name"]` on `<html>`. The `PreferencesContext` applies `document.documentElement.dataset.theme = preferences.theme` reactively whenever the user changes their theme. This is instant, CSS-driven, zero-JS repaint — the correct approach.

The default `:root` block defines the Cyberpunk Teal values, so the interface renders correctly even before React hydrates.

### The Five Themes

| Theme | Primary Accent | BG Base | Character |
|---|---|---|---|
| `cyberpunk-teal` | `#22d3ee` | `#03080f` | Section 9 default. Electric cyan. The canonical experience. |
| `neon-purple` | `#c084fc` | `#06020f` | Deep violet ops center. Hacker's midnight. |
| `synthwave-red` | `#f87171` | `#0d0203` | Retro-futurist. Battle mode. |
| `ghost-white` | `#e2e8f0` | `#080a0d` | Clinical. Ghost Protocol. Almost monochrome. |
| `matrix-green` | `#4ade80` | `#000300` | Terminal phosphor. Absolute black ground. |

Each theme sets 14 tokens: the 6 accent tokens, the 3 glass surface tokens, the 2 bg tokens, the 2 overlay scrim tokens, and the 3 text tokens. This is what makes themes feel like a complete visual shift rather than just a color swap.

### Adding a New Theme

1. Add a new `AppTheme` union member in `PreferencesContext.tsx`
2. Add a `ThemeDef` entry in `SettingsAppearanceSection.tsx` (with visual preview data)
3. Add a `[data-theme="name"]` block in `aero-theme.css` that overrides all 14 tokens
4. If the WPF titlebar theme-change bridge is implemented (see Issue 1), add the theme's `accentColor` and `bgColor` to the message payload in the theme-switch handler

### What Themes Currently Don't Control

- The `--danger` and `--danger-glow` tokens are defined globally in `:root` and are not overridden per-theme (intentional)
- The `--radius-*` tokens are global and not per-theme (intentional — geometry should not change with color theme)
- The `--anim-duration-*` tokens are controlled by `data-animation`, not by `data-theme` (correct separation)
- The WPF window chrome is hardcoded to teal (current debt — see Issue 1)

---

## 7. Animation and Motion

### Philosophy

Motion in NeoVLab should serve the cyberpunk ops-center aesthetic: data appearing, states transitioning, surfaces responding to interaction. It should never feel playful or bouncy. There is no Spring physics here. Everything eases in, lands, and stops. The easing is `cubic-bezier(0.4, 0, 0.2, 1)` — a material-design-derived curve that feels physical without being exaggerated.

### The Three Tiers

**None:** `0ms` across the board. `animation-duration: 0ms !important` and `transition-duration: 0ms !important` applied globally. This exists for accessibility (prefers-reduced-motion equivalent, user-controlled) and for performance testing.

**Subtle (default):** Fast interactions at 100ms, normal state changes at 150ms, slow reveals at 250ms. This is the intended experience — everything feels responsive and crisp.

**Full:** 150ms / 300ms / 400ms. Enables the GITS-inspired keyframe classes: `.glow-pulse`, `.scanline-shimmer`, `.data-stream`, `.stagger-entrance`. These are currently defined in animations.css but are not widely applied in the codebase. They are waiting to be used.

### Current Animation Tokens

```
--anim-duration-fast      100ms (subtle) / 0ms (none) / 150ms (full)
--anim-duration-normal    150ms (subtle) / 0ms (none) / 300ms (full)
--anim-duration-slow      250ms (subtle) / 0ms (none) / 400ms (full)
--anim-easing             cubic-bezier(0.4, 0, 0.2, 1)
```

### Available Keyframe Classes (Full mode only)

- `.glow-pulse` — repeating glow intensity oscillation (for live/active indicators)
- `.scanline-shimmer` — left-to-right shimmer (for loading states or highlighted data)
- `.data-stream` — fade-up-and-out cycling (for stream events, telemetry)
- `.stagger-entrance` — single-shot slide-up-from-4px entrance animation

These classes are dormant until `data-animation="full"` is set. Components can apply them freely — they will self-disable in non-full modes.

### What Is Missing

There is no entrance animation for pages. Route transitions are instantaneous. In Full mode, a stagger-entrance applied to page content on mount would feel appropriately cinematic. This is a gap.

---

## 8. Icon System

All icons use `lucide-react`. The `Icon` primitive wrapper at `src/components/primitives/Icon.tsx` enforces:
- **strokeWidth 1.5** — the outline weight that reads as precise and technical without being hair-thin
- **size via `--density-icon-size`** CSS variable when no explicit size is given — icons scale with density
- **`color="currentColor"`** — icons inherit text color automatically

### Rules for Icon Usage

- **Always use the `Icon` wrapper** for icons that should density-scale
- Use explicit `size` prop only when the icon is in a fixed-geometry context (e.g., a 20px decorative icon in a card header where density scaling would break the layout)
- Do not use `strokeWidth={2}` or higher — it reads too bold against the light glass surfaces
- Use `className="text-[var(--accent-primary)]"` on `Icon` to tint it with the current theme's accent

### What Is Missing

The `Icon` wrapper does not expose a `label` (accessible title) prop. Screen readers see nothing for decorative icons, but also nothing for meaningful icon-only buttons. Any interactive icon-only element needs an explicit `aria-label` on its button wrapper, not on the Icon itself — this is correct but worth documenting explicitly.

---

## 9. The Rules

These are not suggestions. Every one of them exists because its violation is currently visible in the codebase as debt.

### Thou Shalt

- **Use CSS variables for every color reference in component code.** `var(--accent-primary)`, `var(--text-secondary)`, `var(--glass-border)`. Every time.
- **Use the primitive components** (`Button`, `Input`, `Select` or `CustomSelect`, `Modal`) for their respective element types. They exist so you don't have to think about focus states, density scaling, and glass styling every time.
- **Use `glass-panel` for top-level page content containers, `glass-card` for items within panels.**
- **Use `font-mono` for any machine-generated string**: IDs, file paths, hashes, config keys, IP addresses, version strings.
- **Use `p-8 max-w-[5xl|6xl] mx-auto` as the standard page container pattern** until a proper layout component exists.
- **Reference `--panel-pin-top` for sticky panel offset calculations.**
- **Test all five themes** before considering any visual change "done." A component that looks great on cyberpunk-teal and breaks on ghost-white has not been finished.
- **Add `data-animation` conditional classes** for any new animation — do not use Tailwind `animate-*` classes that bypass the animation tier system.

### Thou Shalt Not

- **Never hardcode hex colors** (`#22d3ee`, `#c084fc`) in component TSX or CSS. The design system exists so you do not have to type these.
- **Never use Tailwind color classes** (`text-cyan-400`, `bg-blue-500/20`, `border-purple-300`) in component code. These are outside the theme system and will be wrong on four of the five themes.
- **Never use `opacity-60` as a semantic text color substitute.** It is not themeable. Use `text-[var(--text-secondary)]` or `text-[var(--text-muted)]`.
- **Never use raw `<input>` or `<select>` without `glass-input` class or the Input/Select primitive.** Naked form controls break the aesthetic entirely.
- **Never introduce a fourth glass surface level.** The hierarchy is panel > card > input. If you need more nesting, your component has too many layers.
- **Never apply `backdrop-filter` without `-webkit-backdrop-filter`.** WebView2 (Chromium) supports both. The duplicate is mandatory for correctness.
- **Never put hardcoded padding like `p-8` inside components** that will be reused at different scales. Components take their layout from their parent. Padding belongs on page containers, not on primitives.
- **Never skip `font-mono`** on machine identifiers. Job IDs as plain sans-serif text look like a user error.
- **Never use `!important` outside of `aero-theme.css` modifier classes** (`.topbar-capsule`). If you need `!important`, you are fighting the cascade, which means your component is structured incorrectly.

---

## 10. Known Debt and Roadmap

This section is honest about what is currently wrong. It is not a list of shame — it is a prioritized backlog of things that need to exist before this interface can be called "finished."

### Critical Debt

**D-001: WPF window chrome is theme-unaware.**
The titlebar `BorderBrush`, `DropShadowEffect Color`, button `Foreground`, and hover state backgrounds are all hardcoded to `#22d3ee`. When the user switches to Neon Purple, the React app turns purple and the window frame stays teal. The bridge infrastructure to fix this exists — `WebMessageReceived` is already wired. What is missing is the `theme-changed` message type and the WPF-side color animation. Effort: medium. Visual impact: very high.

**D-002: Background scene library "works" but has a hidden dependency problem.**
The `BackgroundManager` renders the selected background correctly for gradient presets. For image scenes from the API (`/platform/backgrounds`), the backend requires either a `Backgrounds:Path` config entry or a `backgrounds/` directory adjacent to the content root. Neither is documented in `appsettings.json` (no key is present) and neither is guaranteed to exist in a fresh deployment. Result: the API silently returns an empty array, the "Scene Library" section in SettingsAppearanceSection shows nothing, and users think the feature is broken. The fix is partly documentation, partly a default path convention, partly a visible empty state with instructions in the UI.

**D-003: Duplicate gradient data.**
The `GRADIENT_PRESETS` array in `SettingsAppearanceSection.tsx` and the `PRESET_GRADIENTS` record in `BackgroundManager.tsx` contain identical gradient strings. This is a synchronization hazard — change one and you must remember to change the other, or the preview in Settings will show a different gradient than what actually renders. These should be extracted to a shared constant module.

**D-004: Status badge implementations are duplicated.**
`JobsPage.tsx` and `SmokeOutputsPage.tsx` both define their own local `StatusBadge` component with slightly different color mappings (one uses `/15` opacity on backgrounds, the other uses `/20`). There is no shared `StatusBadge` in `primitives/`. This needs a single canonical implementation with the full status color map.

**D-005: Font family is not explicitly declared.**
`index.html` has no `<link>` to a web font. `index.css` has no `font-family` declaration. The interface relies on OS default sans-serif. On Windows 11 this is Segoe UI and looks fine. On other platforms it will be whatever the system provides.

### Design Inconsistencies

**D-006: Heading size inconsistency between sections.**
`SettingsStorageSection` and `SettingsIntegrationsSection` and `SettingsInfrastructureSection` use `text-xl font-semibold` for their H2 headings. `SettingsAppearanceSection` uses `text-base font-semibold` for its subsection H3s. But the parent `SettingsPage` renders these sections without a wrapping heading structure, so the hierarchy `text-2xl` page title → `text-xl` section → `text-base` subsection is correct but only if sections are always rendered under the `text-2xl` page title. `ThemesPage` uses `text-2xl` for its page title and then delegates to `SettingsAppearanceSection` which uses `text-base` — fine. But if sections are ever rendered standalone, the heading levels will be semantically wrong.

**D-007: Filter bar buttons are not using the design system.**
JobsPage and SmokeOutputsPage both have status filter pill buttons implemented as raw `<button className="px-2 py-1 text-xs rounded ...">` with hardcoded `bg-blue-500/20 text-blue-300`. These should use `var(--accent-glow)` / `var(--accent-primary)` to match the theme.

**D-008: Job detail panel uses ad-hoc glass styling.**
The detail panel in JobsPage (`w-full xl:w-[44%] border border-white/10 rounded-lg p-6 bg-gradient-to-br from-[#0e223acc]...`) is a manually recreated glass surface, not `.glass-panel` or `.glass-card`. It uses hardcoded hex colors for its gradient background. On non-teal themes this will look visually disconnected.

**D-009: `toggle-enabled` class references a hardcoded blue.**
`index.css` defines `.toggle-enabled` with `background: rgba(96, 165, 250, 0.2); color: #93c5fd`. This is hardcoded blue — not even the teal accent. It is not theme-aware. Should use `var(--accent-glow)` / `var(--accent-primary)`.

**D-010: `input:focus` uses hardcoded blue glow.**
`index.css` line 58: `box-shadow: 0 0 0 2px rgba(96, 165, 250, 0.2)`. This is blue regardless of theme. Should be `var(--accent-glow)`.

**D-011: `ArtifactBrowserPage` download link uses raw `text-blue-400`.**
The Download link in the artifacts table uses `text-blue-400 hover:text-blue-300`. This should be `text-[var(--accent-primary)]`.

**D-012: ComfyUI progress sections use hardcoded emerald.**
Multiple components use `border-emerald-400/30 bg-emerald-500/10 text-emerald-200` for GPU progress sections. These are hardcoded to green regardless of theme. On Matrix Green they blend in; on synthwave-red they clash badly.

**D-013: SmokeOutputsPage image hover border is hardcoded teal.**
`hover:border-cyan-300/40` — hardcoded teal accent on image cards. Will be wrong on non-teal themes.

**D-014: No shared tab bar component.**
`JobsPage`, `MediaDetailPage`, and `SmokeTestsPage` all implement their own tab bars using different markup patterns. The patterns are similar but not identical — different padding, different active indicator approaches (border-bottom in MediaDetailPage and SmokeTestsPage, inline style in JobsPage). The `Tabs` primitive exists at `primitives/Tabs.tsx` but is not being used by these pages.

### Future Ambitions

**R-001: WPF titlebar live theme animation**
When the user switches themes, the React app posts `{ type: "theme-changed", accentColor, bgColor }` to the WPF host. The host receives it in `OnWebMessageReceived`, parses the new colors, and animates the `WindowBorder.BorderBrush`, the `DropShadowEffect.Color`, and button foregrounds over 300ms using WPF's `ColorAnimation` / `Storyboard`. This makes the entire application — chrome and content — feel like one coherent thing.

**R-002: Explicit font declaration**
Import Inter (or Geist) from a web font source, declare `font-family: 'Inter', 'Segoe UI', system-ui, sans-serif` in `:root`. The current accidental-Segoe-UI situation is acceptable on Windows but is not a decision.

**R-003: Shared gradient data module**
Extract `GRADIENT_PRESETS` to `src/data/backgroundPresets.ts` and import it in both `SettingsAppearanceSection` and `BackgroundManager`. One source of truth.

**R-004: Page transition animations**
In Full animation mode, new page content should use `.stagger-entrance` on mount. A React wrapper component (`AnimatedPage`) that applies the animation class conditionally based on `animationLevel` from preferences would be straightforward to implement.

**R-005: `StatusBadge` primitive**
One canonical `StatusBadge` in `primitives/` with the full status color map, using CSS variable colors where possible and semantic semantic semantic semantic intent colors (emerald = success, rose = error, amber = pending, blue = running, white/glass = cancelled) where theme-invariant clarity matters more than theming.

---

*This document reflects the state of the interface as of 2026-03-25. It will be wrong in specifics by the time you read it, but the principles should hold. The goal is always the same: an interface that looks like Section 9 built it, feels like a precision instrument, and never makes the operator stop to figure out what a button does.*
