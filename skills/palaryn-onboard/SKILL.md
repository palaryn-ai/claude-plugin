---
name: palaryn-onboard
description: "Use this skill when the user asks to onboard someone to Palaryn, create an onboarding message, generate invitation text, or prepare login credentials for a new team member. Triggers on: 'onboard', 'invite to Palaryn', 'create account message', 'send Palaryn credentials'."
version: 1.0.0
---

You are generating a personalized onboarding message for a new Palaryn team member. The message should be in Polish, informal, and friendly.

Ask the user for the following details (if not already provided):
1. **Imie** — first name of the new team member
2. **Email** — their login email
3. **Haslo** — temporary password (or ask if user wants you to generate one)
4. **Rola** — their role/team (optional, for context)

Then generate a message in the following format:

---

Hej {imie}! 👋

Masz dostep do **Palaryn** — naszego gateway'a dla agentow AI. Przez niego lecą wszystkie requesty HTTP, więc masz automatyczny DLP, kontrolę kosztów i pełny audit log.

**Twoje dane logowania:**
- Login: `{email}`
- Hasło: `{haslo}`
- Dashboard: https://app.palaryn.com

⚠️ Zmień hasło po pierwszym logowaniu! (Settings → Password)

**Szybki setup MCP w Claude Code:**

Zainstaluj plugin Palaryn w Claude Code — po instalacji MCP skonfiguruje się automatycznie z OAuth. Wystarczy się zalogować kontem powyżej.

Alternatywnie, ręcznie:
```bash
claude mcp add --transport http palaryn https://app.palaryn.com/mcp
```

**Co daje dashboard:**
- 📊 Real-time tracking kosztów per workspace
- 🔒 Konfiguracja polityk DLP i rate limitów
- 📋 Pełne audit logi wszystkich requestów
- 👥 Zarządzanie członkami workspace'a

**Dostępne toole MCP:**
- `mcp__palaryn__http_get` — requesty GET
- `mcp__palaryn__http_post` — requesty POST
- `mcp__palaryn__http_request` — dowolna metoda (PUT, PATCH, DELETE, etc.)

Jak będziesz mieć pytania, pisz śmiało!

---

Adjust the tone and details based on the user's input. If the user provides additional context (like specific projects or permissions), incorporate that into the message.
