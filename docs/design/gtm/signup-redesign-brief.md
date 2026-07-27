# Design brief: Agenta sign-in / sign-up screen

## What I need from you

Redesign the sign-in / sign-up screen for Agenta. I am attaching a screenshot of the current screen. Give me **4 to 5 visually distinct directions** as full-screen desktop mockups, and for each one a short note on how it collapses on mobile. Do not converge on one safe answer; I want a real spread, from conservative to bold. For each direction, tell me in one or two sentences what it is trying to communicate and why it works.

## What Agenta is (new positioning)

Agenta recently pivoted. The old product was an LLMOps platform (prompt management, evaluation, observability). The current screen still reflects that. The new product is:

> **The open-source workspace for building and running agents.**
>
> Agenta is an open-source workspace where you build specialized agents that automate and augment your work. You build agents by chatting with them: you describe the work, connect the apps they need, and improve them through feedback. You can work with your agents directly in chat and share them with your team. For recurring work, you build background agents that run on a schedule or when an event occurs.

The audience is technical: developers, technical founders, and AI tinkerers, plus the PMs and domain experts who build agents alongside them. The tone is confident and plain-spoken, engineer to engineer. Short declarative claims, no hype adjectives, no exclamation points.

## The current screen (see attached screenshot)

- Two columns. Left half: the sign-in/sign-up form. Agenta logo top-left, a "Welcome to Agenta AI" headline, an outdated LLMOps tagline, then an email-first form (the user enters an email, and the matching sign-in methods appear: password, one-time code, Google/GitHub, or SSO).
- Right half (desktop only): a marketing banner with an outdated headline ("Build Robust AI Applications"), an outdated paragraph, and an illustration of the old product on a graph-paper background.
- On mobile the banner disappears and only the form shows.

## What must keep working

- The email-first form flow stays as it is functionally. You are redesigning the frame around it: the copy, the layout, the marketing panel, the imagery, the overall feel. The form itself just needs to remain clear, centered, and comfortable (it currently sits in a ~400px column).
- The Agenta logo appears on the screen.
- The screen serves both sign-in (returning users) and sign-up (new users) in one flow, so the copy cannot assume the visitor is new.

Everything else is open. If a direction works better with no side banner at all, or a top-anchored layout, or something else entirely, propose it.

## Copy to work with

Mix and match; you can also propose tighter variants of these. Do not write new positioning claims from scratch, but trimming and recombining is welcome.

Taglines / headlines:
- "The open-source workspace for building and running agents"
- "Build agents that automate your work"
- "Welcome to Agenta"
- "Ship agents that actually work"
- "Build agents and AI automations that work"

Supporting lines:
- "Build agents by chatting with them. Share them with your team, connect them to the apps you use, and run them in the background."
- "Build agents that automate your work by chatting with them."
- "Describe the work. Connect your apps. Run it in the background."
- "Build reliable agents to automate your work."

## Brand starting points (guidance, not rules)

Agenta has a brand system in progress. Treat it as a starting point and feel free to push beyond it, but these are the current instincts:

- Warm monochrome canvas (light `#F6F5F3` or near-black `#1E1C1D`), with **one yellow accent moment per screen**: `#F2F25C`. Terracotta `#D97757` exists as a secondary data accent.
- Display type is a literary serif (GT Alpina); one word inside a headline can be highlighted in a nerdy bitmap font (PP Mondwest) inside a soft chip. Body text is Inter.
- Marketing panels tend to be square with hairline borders; rounding is reserved for interactive elements.
- Line-style icons only, no emoji.

If a direction calls for breaking these (for example, a fully typographic dark screen with no yellow, or a warmer illustrated feel), show it anyway and say why.

## Directions I would like to see explored (not limited to these)

1. **Typographic**: no illustration at all. The banner (or the whole screen) carries a large serif headline with one highlighted word, generous space, one accent moment.
2. **Product-idea illustration**: instead of a screenshot, a stylized composition of what the product does, for example small cards or frames suggesting "chat with an agent", "connected apps", "runs on a schedule". No real screenshots exist for the new product yet, so nothing that pretends to be one.
3. **Single column**: drop the side banner entirely; a centered form with one strong line above it, and the brand carried by type, spacing, and background texture.
4. **Dark vs light**: at least one direction in the dark treatment and one in the light, since the new landing page leans dark and the product app is light.
5. **Wildcard**: one direction that ignores my assumptions and takes its own angle on "an open-source workspace where agents do your work".

## Deliverables

- 4 to 5 distinct full-screen desktop options.
- For each: a one-line rationale, the copy it uses, and how it degrades on mobile.
- Flag anything in a direction that would need a new asset (a font, an illustration, a photo) so I know the cost of choosing it.
