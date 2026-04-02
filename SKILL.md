---
name: waha-whatsapp
description: WAHA (WhatsApp HTTP API) Plus — session management, sending messages (text, media, polls, buttons, contacts, location), webhooks, groups, Docker deployment, and production configuration. Auto-triggers on WAHA API calls, WhatsApp automation, and message sending tasks.
disable-model-invocation: false
user-invocable: true
argument-hint: "task description"
---

# WAHA WhatsApp API Skill

Complete reference for WAHA Plus — self-hosted WhatsApp HTTP API.
Use this for any task involving WhatsApp messaging, WAHA sessions, webhook setup, or Docker deployment.

**As of March 2026.** WAHA 2025.5+, recommended engine: GOWS.

---

## 1. Engines

| Engine | Protocol | Sessions/Container | Use Case |
|--------|----------|-------------------|----------|
| WEBJS | Puppeteer/Chromium | ~50 | Most features (events, buttons) |
| NOWEB | Baileys/Node.js | ~500 | Good balance |
| GOWS | whatsmeow/Go+gRPC | ~500+ | **Recommended** — lowest resources, most reliable |

---

## 2. Docker Deployment

### Production docker-compose

```yaml
services:
  waha:
    image: devlikeapro/waha-plus
    restart: unless-stopped
    ports:
      - "127.0.0.1:3000:3000"
    environment:
      - WAHA_WORKER_ID=waha1
      - WHATSAPP_DEFAULT_ENGINE=GOWS
      - WHATSAPP_RESTART_ALL_SESSIONS=True
      - WAHA_AUTO_START_DELAY_SECONDS=15
      - WAHA_PRINT_QR=False
      - WAHA_LOG_LEVEL=info
      - WAHA_LOG_FORMAT=JSON
      - TZ=UTC
      # Webhooks
      - WHATSAPP_HOOK_URL=https://your-webhook.example.com/waha
      - WHATSAPP_HOOK_EVENTS=message,message.ack,session.status
      - WHATSAPP_HOOK_HMAC_KEY=your-hmac-secret
      - WHATSAPP_HOOK_RETRIES_POLICY=exponential
      - WHATSAPP_HOOK_RETRIES_DELAY_SECONDS=2
      - WHATSAPP_HOOK_RETRIES_ATTEMPTS=15
      # Media
      - WHATSAPP_FILES_LIFETIME=180
      - WHATSAPP_DOWNLOAD_MEDIA=true
      - WAHA_BASE_URL=https://waha.yourdomain.com
    volumes:
      - .sessions:/app/.sessions
      - whatsapp-files:/tmp/whatsapp-files
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    env_file:
      - .env
volumes:
  whatsapp-files:
```

### Key Environment Variables

**Core:** `WHATSAPP_API_PORT` (3000), `WHATSAPP_DEFAULT_ENGINE` (GOWS), `WAHA_WORKER_ID`, `WAHA_BASE_URL` (public URL for media.url in webhooks).

**Sessions:** `WHATSAPP_RESTART_ALL_SESSIONS` (True for prod), `WHATSAPP_START_SESSION` (comma-separated), `WAHA_AUTO_START_DELAY_SECONDS`.

**Security:** `WAHA_API_KEY` (`sha512:{hash}` or plain), `WAHA_DASHBOARD_USERNAME/PASSWORD`.

**Media:** `WHATSAPP_DOWNLOAD_MEDIA` (true), `WHATSAPP_FILES_LIFETIME` (180s), `WAHA_MEDIA_STORAGE` (LOCAL/S3), S3 vars for R2: `WAHA_S3_REGION`, `WAHA_S3_BUCKET`, `WAHA_S3_ACCESS_KEY_ID`, `WAHA_S3_SECRET_ACCESS_KEY`, `WAHA_S3_ENDPOINT`.

**Volumes:** `/app/.sessions` (persist auth!), `/tmp/whatsapp-files` (media).

---

## 3. Authentication

All requests need: `X-Api-Key: YOUR_KEY`

```bash
# Generate key
KEY=$(uuidgen | tr -d '-')
HASH=$(echo -n "$KEY" | shasum -a 512 | awk '{print $1}')
# WAHA_API_KEY=sha512:$HASH  OR  WAHA_API_KEY_PLAIN=$KEY
```

### HMAC Webhook Verification

When `WHATSAPP_HOOK_HMAC_KEY` is set, WAHA signs payloads:
- Header: `X-Webhook-Hmac` (SHA512 of raw body)
- Algorithm: `X-Webhook-Hmac-Algorithm: sha512`

```javascript
const crypto = require('crypto');
function verifyHmac(body, signature, secret) {
  const computed = crypto.createHmac('sha512', secret).update(body).digest('hex');
  return crypto.timingSafeEqual(Buffer.from(computed), Buffer.from(signature));
}
```

---

## 4. Session Management

### States: `STOPPED → STARTING → SCAN_QR_CODE → WORKING` (or `→ FAILED`)

```bash
# Create + start session
curl -X POST http://localhost:3000/api/sessions \
  -H "X-Api-Key: KEY" -H "Content-Type: application/json" \
  -d '{
    "name": "default", "start": true,
    "config": {
      "webhooks": [{
        "url": "https://your-server/webhook",
        "events": ["message", "message.ack", "session.status"],
        "hmac": {"key": "secret"},
        "retries": {"policy": "exponential", "delaySeconds": 2, "attempts": 15}
      }]
    }
  }'

# List / Get / Start / Stop / Restart / Logout / Delete
curl http://localhost:3000/api/sessions -H "X-Api-Key: KEY"
curl http://localhost:3000/api/sessions/default -H "X-Api-Key: KEY"
curl -X POST http://localhost:3000/api/sessions/default/start -H "X-Api-Key: KEY"
curl -X POST http://localhost:3000/api/sessions/default/stop -H "X-Api-Key: KEY"
curl -X POST http://localhost:3000/api/sessions/default/restart -H "X-Api-Key: KEY"
curl -X POST http://localhost:3000/api/sessions/default/logout -H "X-Api-Key: KEY"
curl -X DELETE http://localhost:3000/api/sessions/default -H "X-Api-Key: KEY"

# Get QR code
curl http://localhost:3000/api/default/auth/qr -H "X-Api-Key: KEY" -H "Accept: image/png" -o qr.png
# QR as JSON (base64)
curl http://localhost:3000/api/default/auth/qr -H "X-Api-Key: KEY" -H "Accept: application/json"

# Pairing code (alternative to QR)
curl -X POST http://localhost:3000/api/default/auth/request-code \
  -H "X-Api-Key: KEY" -H "Content-Type: application/json" \
  -d '{"phoneNumber": "12132132130"}'
```

**QR expiry:** 60s first code, 20s subsequent, max 6 before restart needed.

---

## 5. Chat ID Formats

| Type | Format | Example |
|------|--------|---------|
| Direct | `{phone}@c.us` | `12132132130@c.us` |
| Group | `{id}@g.us` | `120363XXXXX@g.us` |
| Channel | `{id}@newsletter` | `120363XXXXX@newsletter` |
| Status | `status@broadcast` | `status@broadcast` |

**Phone format:** Country code + number. No `+`, no spaces, no dashes.

---

## 6. Sending Messages

All send endpoints accept: `session`, `reply_to` (message ID to quote), `mentions` (array or `["all"]`).

### Text
```bash
curl -X POST http://localhost:3000/api/sendText \
  -H "X-Api-Key: KEY" -H "Content-Type: application/json" \
  -d '{"chatId": "12132132130@c.us", "text": "Hello!", "linkPreview": true}'
```

### Image / Video / Voice / File

```bash
# Image (by URL or base64)
curl -X POST http://localhost:3000/api/sendImage \
  -H "X-Api-Key: KEY" -H "Content-Type: application/json" \
  -d '{"chatId": "12132132130@c.us", "file": {"mimetype": "image/jpeg", "url": "https://..."}, "caption": "Photo"}'

# Video
curl -X POST http://localhost:3000/api/sendVideo \
  -d '{"chatId": "...@c.us", "file": {"mimetype": "video/mp4", "url": "https://..."}, "caption": "Watch"}'

# Voice (must be OPUS in OGG, or use "convert": true)
curl -X POST http://localhost:3000/api/sendVoice \
  -d '{"chatId": "...@c.us", "file": {"mimetype": "audio/ogg; codecs=opus", "url": "https://..."}}'

# File/Document
curl -X POST http://localhost:3000/api/sendFile \
  -d '{"chatId": "...@c.us", "file": {"mimetype": "application/pdf", "url": "https://...", "filename": "report.pdf"}}'
```

### Location / Contact / Poll

```bash
# Location
curl -X POST http://localhost:3000/api/sendLocation \
  -d '{"chatId": "...@c.us", "latitude": 38.89, "longitude": -77.09, "title": "Office"}'

# Contact vCard
curl -X POST http://localhost:3000/api/sendContactVcard \
  -d '{"chatId": "...@c.us", "contacts": [{"fullName": "John", "phoneNumber": "+1555123", "whatsappId": "1555123"}]}'

# Poll
curl -X POST http://localhost:3000/api/sendPoll \
  -d '{"chatId": "...@c.us", "poll": {"name": "How are you?", "options": ["Great", "OK", "Bad"], "multipleAnswers": false}}'
```

### Reaction / Forward / Edit / Delete

```bash
# React (empty string to remove)
curl -X POST http://localhost:3000/api/reaction \
  -d '{"chatId": "...@c.us", "messageId": "MSG_ID", "reaction": "👍"}'

# Forward
curl -X POST http://localhost:3000/api/forwardMessage \
  -d '{"chatId": "DEST@c.us", "messageId": "MSG_ID"}'

# Edit (note: @ encoded as %40 in URL)
curl -X PUT http://localhost:3000/api/default/chats/CHAT_ID%40c.us/messages/MSG_ID \
  -d '{"text": "Updated text"}'

# Delete
curl -X DELETE http://localhost:3000/api/default/chats/CHAT_ID%40c.us/messages/MSG_ID
```

### Typing / Presence

```bash
curl -X POST http://localhost:3000/api/startTyping -d '{"chatId": "...@c.us"}'
curl -X POST http://localhost:3000/api/stopTyping -d '{"chatId": "...@c.us"}'

# Presence: online, offline, typing, recording, paused
curl -X POST http://localhost:3000/api/default/presence -d '{"status": "online"}'
# CRITICAL: Send offline after online/typing/recording or phone stops getting push notifications
curl -X POST http://localhost:3000/api/default/presence -d '{"status": "offline"}'
```

### Read Receipt
```bash
curl -X POST http://localhost:3000/api/sendSeen \
  -d '{"chatId": "...@c.us", "messageIds": ["MSG_ID"]}'
```

---

## 7. Webhook Events

### Webhook Envelope
```json
{
  "id": "evt_...", "timestamp": 1741249702485,
  "event": "message", "session": "default",
  "me": {"id": "71111@c.us", "pushName": "Bot"},
  "payload": { ... },
  "engine": "GOWS"
}
```

### Event Reference

| Event | When |
|-------|------|
| `session.status` | State changes (STOPPED/STARTING/SCAN_QR_CODE/WORKING/FAILED) |
| `message` | Incoming from others only |
| `message.any` | All messages incl. outbound (has `source`: `app`/`api`) |
| `message.ack` | Delivery/read status (-1=ERROR, 0=PENDING, 1=SERVER, 2=DEVICE, 3=READ, 4=PLAYED) |
| `message.reaction` | Emoji reaction (empty `text` = removed) |
| `message.edited` | Message edited |
| `message.revoked` | Message deleted |
| `group.v2.join` | You joined a group |
| `group.v2.leave` | You left a group |
| `group.v2.participants` | Someone joins/leaves/promoted/demoted |
| `group.v2.update` | Group info changed |
| `poll.vote` | Poll vote received |
| `call.received` | Incoming call |
| `presence.update` | Contact typing/online/offline |
| `label.upsert/deleted` | Label changes (Plus) |

### Retry Policies
| Policy | Pattern (delay=2s) |
|--------|-------------------|
| `constant` | 2, 2, 2, 2 |
| `linear` | 2, 4, 6, 8 |
| `exponential` | 2, ~4, ~8, ~16 (20% jitter) |

---

## 8. Groups API

```bash
# Create
curl -X POST http://localhost:3000/api/default/groups \
  -d '{"name": "My Group", "participants": [{"id": "12132132130@c.us"}]}'

# List
curl "http://localhost:3000/api/default/groups?limit=10" -H "X-Api-Key: KEY"

# Get info
curl http://localhost:3000/api/default/groups/GROUP_ID%40g.us -H "X-Api-Key: KEY"

# Add/Remove participants
curl -X POST http://localhost:3000/api/default/groups/GID%40g.us/participants/add \
  -d '{"participants": [{"id": "12132132130@c.us"}]}'
curl -X POST http://localhost:3000/api/default/groups/GID%40g.us/participants/remove \
  -d '{"participants": [{"id": "12132132130@c.us"}]}'

# Promote/Demote admin
curl -X POST http://localhost:3000/api/default/groups/GID%40g.us/admin/promote \
  -d '{"participants": [{"id": "12132132130@c.us"}]}'

# Invite code / Join
curl http://localhost:3000/api/default/groups/GID%40g.us/invite-code -H "X-Api-Key: KEY"
curl -X POST http://localhost:3000/api/default/groups/join \
  -d '{"code": "https://chat.whatsapp.com/INVITECODE"}'
```

---

## 9. Contacts & Chats

```bash
# Check if number exists on WhatsApp
curl -X POST http://localhost:3000/api/checkNumberStatus \
  -d '{"phone": "12132132130"}'

# List chats
curl "http://localhost:3000/api/default/chats?limit=100&sortBy=messageTimestamp&sortOrder=desc"

# List messages from chat
curl "http://localhost:3000/api/default/chats/CHAT_ID%40c.us/messages?limit=10&downloadMedia=true"
```

---

## 10. Critical Gotchas

1. **`@` in URLs → `%40`** — chat IDs in path params must be URL-encoded
2. **QR codes expire fast** — 60s first, 20s subsequent, max 6 before restart
3. **MUST send `offline` after `online`/`typing`** — or phone stops getting push notifications
4. **Voice messages need OPUS in OGG** — use `"convert": true` or `/media/convert/voice`
5. **Free edition = single session only** — named `default`, 422 for other names
6. **`message` event excludes your own** — use `message.any` for both directions
7. **Media URLs use localhost by default** — set `WAHA_BASE_URL` for webhook consumers
8. **Media files deleted after 180s** — increase `WHATSAPP_FILES_LIFETIME` or use S3
9. **`/app/.sessions` volume is critical** — without it, sessions lose auth on restart
10. **`WHATSAPP_RESTART_ALL_SESSIONS=True`** — must be set for production
11. **Session names: max 54 chars** — alphanumeric, hyphens, underscores only
12. **Buttons are fragile** — polls are recommended interactive alternative
13. **Phone format: no `+`, no spaces, no dashes** — just `12132132130`
14. **`sendList` only works for 1:1 chats** — not groups
15. **Test vs Production webhooks** — use production URL in external systems

