# Channels research

Date started: 2026-07-20. Status: competitive research. Review copy on Mahmoud's fork; not intended to merge into Agenta-AI/agenta.

## The feature under study

"Channels": connect an Agenta agent to a messaging surface (Slack, Discord, Telegram,
WhatsApp, Teams, email, WeChat/Feishu/DingTalk). A user @mentions the agent in a
channel, which opens a session. The whole team can talk to the agent in that thread;
every reply in the thread lands in the same session, and the same session can be
continued from any surface (the web app, another channel). Optionally, approvals
(human-in-the-loop confirmation of agent actions) flow through the same channel.

## Approach (mirrors docs/design/mcp-gateway-research/)

1. **Light pass** (this phase): parallel subagents survey vendors from their docs,
   landing pages, and code. One raw file per vendor group in `raw/`. Goal: agree on
   direction and the requirement set with Mahmoud.
2. **Deep pass**: after feedback, deeper research on the agreed direction.
3. **Synthesis**: `report.md` (explains the space step by step) and
   `first-principles-design.md` (architecture / RFC for Agenta).

## Extraction template (what each raw file answers)

- What the product is and how it positions the channel feature.
- Channels supported, and depth per channel (threads? DMs? group chats? reactions?).
- Connection UX: how you wire agent to channel (bot install, OAuth, token paste, QR).
- Conversation model: @mention-to-session mapping, thread-to-session continuity,
  continuing one session across surfaces.
- Multi-user behavior: several humans in one thread, identity mapping (channel user
  to platform user), RBAC / allowlists / pairing of unknown users.
- Approvals and human-in-the-loop through the channel.
- Context given to the agent: speaker identity, channel metadata, history windowing.
- Limitations, gotchas, security posture and incidents.
- Pricing/packaging where it shapes the feature.
- Sources with URLs; unverifiable claims marked **[unverified]**.
