# WAHA Skill

[![Author](https://img.shields.io/badge/Author-Daniel_Rudaev-000000?style=flat)](https://github.com/daniel-rudaev)
[![Studio](https://img.shields.io/badge/Studio-D1DX-000000?style=flat)](https://d1dx.com)
[![WhatsApp](https://img.shields.io/badge/WAHA-Skill-25D366?style=flat&logo=whatsapp&logoColor=white)](https://waha.devlike.pro)
[![License](https://img.shields.io/badge/License-MIT-green?style=flat)](./LICENSE)

WAHA (WhatsApp HTTP API) skill for AI agents. Covers session management, all message types (text, media, polls, buttons, contacts, location), webhooks, groups, Docker deployment, and production configuration. Built from production WAHA Plus deployments.

## What's Included

| Topic | What it covers |
|-------|---------------|
| Engines | WEBJS vs NOWEB vs GOWS — capacity, use case, recommendation |
| Docker Deployment | Production docker-compose, all key environment variables, volumes |
| Authentication | X-Api-Key, HMAC webhook verification (SHA512), key generation |
| Session Management | States, create/start/stop/restart/logout/delete, QR code and pairing code |
| Chat ID Formats | Direct (`@c.us`), group (`@g.us`), channel (`@newsletter`), status |
| Sending Messages | Text, image, video, voice, file, location, contact vCard, poll |
| Message Actions | Reaction, forward, edit, delete, typing indicator, presence, read receipts |
| Webhook Events | Full event reference — message, message.ack, session.status, groups, polls, calls |
| Webhook Retry Policies | constant, linear, exponential — delay patterns |
| Groups API | Create, list, get info, add/remove participants, promote/demote admin, invite |
| Contacts & Chats | Check number, list chats, list messages |
| Gotchas | 15 hard-won lessons — `@` URL encoding, QR expiry, offline presence, voice format, session volumes |

## Install

### Claude Code

```bash
git clone https://github.com/D1DX/waha-skill.git
cp -r waha-skill ~/.claude/skills/waha
```

Or as a git submodule:

```bash
git submodule add https://github.com/D1DX/waha-skill.git path/to/skills/waha
```

### Other AI Agents

Copy `SKILL.md` into your agent's prompt or knowledge directory. The skill is structured markdown — works with any LLM agent that reads reference files.

## Structure

```
waha-skill/
└── SKILL.md    — Main skill (Docker, sessions, messaging, webhooks, groups, gotchas)
```

## Sources

- **WAHA API:** Verified against [WAHA documentation](https://waha.devlike.pro/docs/overview/introduction/) and production WAHA Plus deployments (v2025.5+, March 2026).
- **GOWS engine:** Verified from [WAHA engine comparison](https://waha.devlike.pro/docs/how-to/engines/) documentation.
- **Webhook events:** Verified from live webhook captures and [WAHA webhook documentation](https://waha.devlike.pro/docs/how-to/webhooks/).

## Credits

Built by [Daniel Rudaev](https://github.com/daniel-rudaev) at [D1DX](https://d1dx.com).

## License

MIT License — Copyright (c) 2026 Daniel Rudaev @ D1DX
