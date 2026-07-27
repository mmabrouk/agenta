# Handoff: Agenta sign-in / sign-up redesign

## Overview
Redesign of the combined sign-in / sign-up screen for Agenta's new positioning ("the open-source workspace for building and running agents"). One screen serves new and returning users. The chosen direction is a conservative two-column layout: auth form left, marketing panel right. Three states are specified (first visit, returning via OAuth, returning via email code), each in light and dark.

## About the design files
`signin-designs.html` is a **design reference created in HTML** — a self-contained canvas showing all states in both modes (open it in any browser; pan/zoom). It is a prototype of intended look and behavior, **not production code**. Recreate these designs in the target codebase's existing environment (the Agenta web app: Next.js/React) using its established patterns; do not ship this HTML.

## Fidelity
**High-fidelity.** Colors, type, spacing, radii, and shadows are final and use the Agenta brand tokens (included under `tokens/`). Recreate pixel-perfectly.

⚠️ **Font licensing**: GT Alpina (display serif) and PP Mondwest are trial fonts in the design workspace. License them before production. Inter loads from Google Fonts.

## The canvas
In `signin-designs.html`, the newest work is at the top:
- **3a** — returning visit, last used = email code (light + dark)
- **2a** — first visit, final form (light + dark)
- **2b** — returning visit, last used = Google (light + dark)
- Turn 1 below (1a–1e) is exploration history; **implement 2a / 2b / 3a only.**

## Layout (desktop, 1440×900 reference)
- Page background: light `#F6F5F3` / dark `#1E1C1D`. 12px outer gutter, 12px gap between columns.
- **Left column**: 560px fixed. Logo top-left (full lockup, 23px tall; black mark on light, white on dark), padding 28px 36px 40px. Form column is 400px, horizontally centered, vertically centered in remaining space.
- **Right column**: fills the rest. A flat panel — light: `#FFFFFF` with 1px `#E5E5E3` ring; dark: `#242220` with 1px `rgba(229,229,227,0.1)` ring. Content vertically centered, padding 96px, 26px vertical gaps. No border radius on the panel (square, per brand).

### Right panel content (identical in all states)
1. Chip link → `https://github.com/Agenta-AI/agenta`: GitHub mark (13px) + "Open source · GitHub" + ↗ arrow. Pill (999px radius), 5px 12px padding, Inter 500 12/18 +3% tracking. Light: bg `#F6F5F3`, 1px `#F0EFED` ring, text `#676770`. Dark: bg `rgba(255,255,255,0.06)`, ring `rgba(229,229,227,0.1)`, text `rgba(255,255,255,0.64)`.
2. Headline: "Build agents that automate your work" — GT Alpina Light 300, 48/52, max-width 520px. Light `#242424` / dark `#FFFFFF`.
3. Three feature rows, 1px hairline separators (`#F0EFED` light / `rgba(229,229,227,0.1)` dark), 16px vertical padding, 14px gap, 20px line icons (1.5px stroke, currentColor), Inter 400 15/22:
   - Build agents by chatting with them (chat-bubble icon)
   - Connect the apps you use (grid icon)
   - Run background agents on a schedule or event (clock icon)

## Form (400px column)
Heading above form: GT Alpina 300 34/40 — "Welcome to Agenta" (first visit) / "Welcome back" (returning). First visit only: sub-line Inter 400 15/22 secondary — "Sign in or create an account."

Form stack, 22px gaps, Inter throughout:

1. **Data residency**
   - Label row: "DATA RESIDENCY" (Inter 500 12/18, +3% tracking, uppercase) + "Learn more" link, space-between.
   - Two pills, 8px gap, each flex:1, 40px tall, 8px radius, globe icon 14px + "EU" / "US" (Inter 500 14/20).
   - Selected: light = `#242424` bg, white text; dark = `#FFFFFF` bg, `#242424` text.
   - Unselected: light = white bg + ring shadow `0 0 0 1px rgba(72,70,68,0.08), 0 1px 3px rgba(72,70,68,0.08)`, text `#676770`; dark = transparent, 1px `rgba(229,229,227,0.1)` border, text `rgba(255,255,255,0.64)`.
   - Returning: the remembered region is preselected and carries a small "Last used" tag inside the pill (10/15, 999px radius; on light-selected: bg `rgba(255,255,255,0.16)` white text; on dark-selected: bg `rgba(2,1,17,0.08)` text `#676770`).

2. **OAuth buttons** — "Continue with Google", "Continue with GitHub". 44px tall, 8px radius, Inter 500 14/20, 16px monochrome brand glyphs, 10px gap between buttons. Light: white bg + ring shadow (above), text `#242424`. Dark: `rgba(255,255,255,0.04)` bg, 1px `rgba(229,229,227,0.1)` border, white text.

3. **Divider** — 1px hairlines + "or" (Inter 400 12/18, faint: `#A3A19F` light / `rgba(255,255,255,0.4)` dark).

4. **Email** — input 44px, 8px radius, placeholder "Enter your email address" (`#A3A19F` / `rgba(255,255,255,0.4)`), same surface treatment as OAuth buttons; then **Continue** button, 44px, yellow keycap: `linear-gradient(180deg,#F2F25C,#E7E712)`, `inset 0 2px 6.4px rgba(255,255,255,0.8)`, text `#242424`, Inter 500 14/20. The only yellow on screen — same in both modes.

5. **Terms** — Inter 400 12/18 secondary: "By continuing, you agree to the Terms of Service and Privacy Policy." Links same color, underline on hover.

## States & behavior

### Email-first flow (unchanged from current product)
Entering an email and pressing Continue reveals the matching methods (password / one-time code / SSO) exactly as today. This redesign only reframes the screen.

### First visit (2a)
Order: residency (EU default) → Google → GitHub → or → email → Continue → terms.

### Returning, last used = OAuth (2b)
- Heading: "Welcome back" (no sub-line).
- The last-used provider button moves above the divider, 48px tall, stronger ring (light: `0 0 0 1px #A3A19F, 0 1px 3px rgba(72,70,68,0.1)`; dark: `rgba(255,255,255,0.08)` bg, 1px `rgba(255,255,255,0.28)` border) and an inline "Last used" tag (Inter 500 11/16 pill, light: `#F6F5F3` bg + `#F0EFED` ring; dark: `rgba(255,255,255,0.08)` + hairline ring).
- Below the divider: the other provider, email input, Continue.
- Residency pills stay in place; remembered region preselected + tagged.

### Returning, last used = email code (3a)
- Heading: "Welcome back".
- The email block moves above the divider: input 48px with the stronger ring and the "Last used" tag positioned inside the input, right-aligned (12px from edge, vertically centered; add ~96px right padding on the input). Continue directly below.
- Google and GitHub move below the divider (normal 44px treatment).

### Memory (client-side)
Store locally (localStorage or cookie, no server call), written after a successful auth:
- `lastAuthMethod`: `"google" | "github" | "email"`
- `lastRegion`: `"eu" | "us"`
Presence of `lastAuthMethod` switches the screen into the returning state. "Last used" tags are the only signal — no explanatory sentences anywhere.

### Duplicate-account prevention (server-side, required)
Link accounts by verified email. If a user authenticates with a different method for an email that already has an account, sign them into the existing account (link the new method) — never create a second account, never dead-end with an error.

## Responsive / mobile
- Right marketing panel drops entirely; left column becomes full-width, form stays ≤400px centered.
- All hit targets are ≥40px (44–48px buttons/inputs).
- In returning states the last-used method sits above the fold.

## Design tokens
See `tokens/colors.css`, `tokens/typography.css`, `tokens/effects.css` (canonical). Key values:
- Ink: `#242424` heading, `#676770` secondary, `#A3A19F` faint
- Paper: `#FFFFFF`, `#F6F5F3` page, `#F0EFED` soft hairline, `#E5E5E3` hairline
- Dark: `#1E1C1D` page, `#242220` panel, hairline `rgba(229,229,227,0.1)`, text `#FFFFFF` / `rgba(255,255,255,0.64)`
- Yellow: `#F2F25C` → `#E7E712` (primary button gradient only)
- Radii: 8px buttons/inputs, 999px tags/chips; panels square
- Fonts: GT Alpina Light 300 (headlines), Inter (everything else)

## Assets
- `assets/logos/` — Agenta full lockup + symbol. Naming = target background: `-light` is the black mark for light backgrounds, `-dark` the white mark for dark backgrounds, `-dark-accent` the yellow mark.
- Icons: line icons at 1.5px stroke (Lucide-compatible); Google/GitHub glyphs are monochrome, inherit text color.

## Files
- `signin-designs.html` — self-contained design canvas (all states, light + dark)
- `tokens/*.css` — brand tokens
- `assets/logos/*` — logo set
