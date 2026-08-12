---
name: beeper-whatsapp
description: Read, search, and send messages across WhatsApp, Telegram, Signal, and every other network connected to Beeper, via the local Beeper Desktop API. Use whenever the user mentions Beeper, WhatsApp, or reading/searching/sending chat messages.
---

# Beeper — chat control across every connected network

Beeper Desktop exposes a local REST API on `http://localhost:23373/v1/…`. This skill
wraps it in a single stdlib-only Python CLI so any agent with a shell can read,
search, and send messages across every connected chat network (WhatsApp, Telegram,
Signal, iMessage, Instagram, Matrix, …).

**The Beeper Desktop app must be running.** If it is not, every command exits 2 with
`Beeper Desktop is not running. Open the Beeper Desktop app and retry.`

## The CLI

```bash
<skill dir>/scripts/beeper <command> [args] [--json]
```

The primary install location is `~/.claude/skills/beeper-whatsapp/`, so in practice:

```bash
~/.claude/skills/beeper-whatsapp/scripts/beeper accounts
```

`--json` (before or after the subcommand) prints raw API JSON instead of the
human-readable formatting — use it when you need to parse fields programmatically.

**Always single-quote chatIDs and messageIDs in the shell.** They look like
`'!EXAMPLEchatid:ba_XXXXXXXXXXXX.local-whatsapp.localhost'` — the leading `!`
triggers zsh/bash history expansion and the `:` and `-` confuse unquoted parsing.

## Discover the user's accounts first

There is no fixed account list — every user's Beeper is connected to different
networks. **Run `beeper accounts` once at the start of a session** and use what it
returns for the rest of the conversation:

```bash
$B accounts        # → accountIDs, network, phone/handle, display name
```

Then pass `--account <accountID>` (or the positional `accountID` where a command
requires it) to scope a command to one network. Omit it to search across everything.

AccountIDs are either a bare network name (`telegram`, `whatsapp`, `signal`) or a
network-prefixed local ID (`local-whatsapp_ba_XXXXXXXXXXXX`) — take them verbatim
from `accounts` output, never guess or hand-construct them.

**Users often have several accounts on the same network** — e.g. a work WhatsApp and
a personal WhatsApp, or two Telegram accounts. The same person then frequently has a
chat on *both*. When `find-chat` returns duplicates across accounts, check
`lastActivity` and **ask the user which account and which chat they mean before
acting** — especially before anything that writes.

## Command reference

### Read

| Command | What it does | Example |
|---|---|---|
| `accounts` | List connected accounts | `beeper accounts` |
| `chats` | Recent chats. `--account`, `--limit N` (1-200), `--unread`, `--inbox primary\|low-priority\|archive`, `--cursor`, `--direction` | `beeper chats --limit 10 --unread` |
| `find-chat <query>` | Search chats by title; `--scope participants` matches member names instead; `--account`, `--limit` | `beeper find-chat "Alice"` |
| `chat <chatID>` | Chat details + participant list (`--max-participants N`) | `beeper chat '!EXAMPLEchatid:ba_XXXX.local-whatsapp.localhost'` |
| `messages <chatID>` | Recent messages, oldest→newest. `--limit N` auto-paginates (API pages are 20) | `beeper messages '!EXAMPLEchatid:…' --limit 30` |
| `search-messages <q>` | Full-text search. `--chat`, `--account`, `--sender me\|others\|<id>`, `--after`/`--before` ISO8601, `--chat-type`, `--media any\|image\|video\|link\|file`, `--limit` (max 20) | `beeper search-messages "invoice" --limit 10` |
| `search <q>` | Unified search: chats, group matches, and messages in one call | `beeper search "Alice"` |
| `contacts <accountID> <q>` | Search that account's address book | `beeper contacts telegram "Bob"` |
| `list-contacts <accountID>` | Paginated contact dump; `--query`, `--limit`, `--cursor` | `beeper list-contacts whatsapp --limit 50` |
| `download <mxcURL \| chatID msgID>` | Fetch an attachment; `--out <file\|dir>`, `--index N` for multi-attachment messages. Prints the saved path | `beeper download '!EXAMPLEchatid:…' 160518 --out ~/Desktop/pic.jpg` |
| `info` | App/server version, MCP endpoint | `beeper info` |

### Write — only on explicit user instruction

| Command | Example |
|---|---|
| `send <chatID> <text>` (`--reply-to <msgID>`, `--attach <file>`) | `beeper send '!EXAMPLEchatid:…' "On my way"` |
| `edit <chatID> <msgID> <newText>` | `beeper edit '!EXAMPLEchatid:…' 177785 "corrected text"` |
| `react <chatID> <msgID> <emoji>` / `unreact …` | `beeper react '!EXAMPLEchatid:…' 177785 "👍"` |
| `new-chat <accountID> <participantID…>` (`--mode start\|create`, `--type`, `--title`, `--message`) | `beeper new-chat whatsapp 15551234567 --message "hi"` |
| `archive <chatID>` / `unarchive <chatID>` | `beeper archive '!EXAMPLEchatid:…'` |
| `remind <chatID> <ISO8601>` / `unremind <chatID>` | `beeper remind '!EXAMPLEchatid:…' 2026-08-13T09:30:00` |
| `upload <file>` | `beeper upload ~/Desktop/quote.pdf` → uploadID (or just use `send --attach`) |
| `focus` (`--chatID`, `--messageID`, `--draft`) | `beeper focus --chatID '!EXAMPLEchatid:…' --draft "typed but not sent"` |
| `raw <METHOD> <path> [--data JSON] [--param k=v]` | `beeper raw GET /v1/chats --param accountIDs=whatsapp` |

## Recipes

**Reply to a person by name**
```bash
B=~/.claude/skills/beeper-whatsapp/scripts/beeper
$B accounts                               # 0. learn the user's accounts (once per session)
$B find-chat "Alice"                      # 1. get the chatID (confirm which account!)
$B messages '<chatID>' --limit 15         # 2. read the thread for context
# 3. show the user the drafted reply and the chat title, then only if they approve:
$B send '<chatID>' "…"
```

**Morning unread sweep**
```bash
$B chats --unread --limit 30
# then for anything that matters:
$B messages '<chatID>' --limit 10
```

**Find every mention of a topic across all accounts**
```bash
$B search-messages "site visit" --limit 20
$B search-messages "payment" --account local-whatsapp_ba_XXXXXXXXXXXX --after 2026-08-01T00:00:00
```

**Pull an attachment out of a chat**
```bash
$B --json messages '<chatID>' --limit 20 | grep srcURL   # or read the formatted output
$B download '<chatID>' <messageID> --out ~/Desktop/
```

## Safety rules

1. **Never send, edit, react, archive, or set a reminder unless the user explicitly
   asked for that action in the current conversation.** Reading and searching are
   free; anything that writes to a chat is not.
2. **Confirm the recipient before sending.** State the chat title *and* which account
   it is on, and show the exact message text, then wait for approval. A wrong chatID
   sends a private message to the wrong person irreversibly.
3. **Never store, echo, forward, or summarize OTPs, PINs, passwords, or 2FA codes**
   found in messages. Skip them silently. Do not write them to any note or file.
4. Messages are personal data — do not bulk-export chats into files unless asked.
5. `edit` only works on messages the user sent; WhatsApp also enforces a time window.
6. If a command returns HTTP 403, the token lacks the `write` scope — tell the user
   to re-authorize in Beeper Desktop rather than retrying.

## Notes / API quirks

- `GET /v1/chats` has no `limit`, so the `chats` command uses `/v1/chats/search`
  (which supports `limit`, `unreadOnly`, `inbox`, `accountIDs`).
- `GET /v1/chats/{id}/messages` has no `limit` either — it returns 20 newest-first
  per page; the CLI auto-paginates via `oldestCursor` to satisfy `--limit`.
- `search-messages` is capped at 20 results per call by the API; page with `--cursor`.
- Attachment `srcURL`s are usually already-cached `file://` paths; `download` copies
  them directly and only calls the API for `mxc://` / `localmxc://` URLs.
- Message bodies contain light HTML (`<br>`, `<strong>`, `&amp;`). Formatted output
  flattens it; `--json` returns the raw text unchanged.
- Reminders take epoch milliseconds; the CLI converts ISO8601 for you (no timezone
  offset = your local time).
- Token lives at `<skill dir>/token` (mode 600) — i.e.
  `~/.claude/skills/beeper-whatsapp/token` for the standard install. Override with
  `$BEEPER_ACCESS_TOKEN`. Base URL override: `$BEEPER_API_URL`.
- Exit codes: `0` success, `1` API/HTTP error (message on stderr), `2` Beeper Desktop
  unreachable.
