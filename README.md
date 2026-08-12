# beeper-whatsapp

A universal CLI + agent skill for the **Beeper Desktop local API** — read, search, and
send messages across WhatsApp, Telegram, Signal, iMessage, Instagram, Matrix, and every
other network you have connected to Beeper.

- **Zero dependencies.** One self-contained Python 3.9+ script, standard library only.
- **Works with any agent that can run a shell command** — Claude Code, Cursor, Codex,
  Aider, cron jobs, plain scripts.
- **Ships as a Claude Code skill** (`SKILL.md`) so agents know how and when to use it.
- **Local only.** Talks to `http://localhost:23373` on your own machine. No cloud, no
  telemetry, no third-party service ever sees your messages.

Requires the **Beeper Desktop app to be installed and running**. If it isn't, every
command exits `2` with `Beeper Desktop is not running.`

---

## Install

### Option A — skills.sh

```bash
npx skills add <owner>/<repo>
```

(Replace `<owner>/<repo>` with this repository's path on GitHub. See
[skills.sh](https://skills.sh) for the current CLI.)

### Option B — manual

```bash
git clone <repo-url> ~/.claude/skills/beeper-whatsapp
chmod +x ~/.claude/skills/beeper-whatsapp/scripts/beeper
```

Any location works — the script finds its token relative to its own path — but
`~/.claude/skills/beeper-whatsapp/` is where Claude Code picks up the skill
automatically.

Optionally put it on your `$PATH`:

```bash
ln -s ~/.claude/skills/beeper-whatsapp/scripts/beeper /usr/local/bin/beeper
```

---

## Token setup

Generate a token in **Beeper Desktop → Settings → Integrations** (also called Desktop
API / Developer), then save it next to the script:

```bash
printf 'bdapi_xxxxxxxxxxxx\n' > ~/.claude/skills/beeper-whatsapp/token
chmod 600 ~/.claude/skills/beeper-whatsapp/token
~/.claude/skills/beeper-whatsapp/scripts/beeper accounts   # verify
```

Or skip the file entirely and use the environment:

```bash
export BEEPER_ACCESS_TOKEN=bdapi_xxxxxxxxxxxx
```

Resolution order is `$BEEPER_ACCESS_TOKEN` first, then `<skill dir>/token`.
`token.example` in this repo shows the expected one-line format. The real `token`
file is git-ignored.

| Env var | Purpose |
|---|---|
| `BEEPER_ACCESS_TOKEN` | Use this token instead of the `token` file |
| `BEEPER_API_URL` | Point at a different host/port (default `http://localhost:23373`) |

---

## Quick usage

```bash
B=~/.claude/skills/beeper-whatsapp/scripts/beeper

$B accounts                                  # which networks are connected
$B chats --limit 10 --unread                 # unread inbox sweep
$B find-chat "Alice"                         # locate a chat by title
$B messages '<chatID>' --limit 20            # read a thread
$B search-messages "invoice" --limit 10      # full-text search everywhere
$B send '<chatID>' "hello"                   # send a message
$B --json search "Bob"                       # raw JSON for scripting
```

Run with no arguments for the full command list. `--json` works before or after any
subcommand and returns raw API JSON for programmatic consumption.

**Quote chatIDs.** They look like
`'!EXAMPLEchatid:ba_XXXXXXXXXXXX.local-whatsapp.localhost'` — the leading `!` triggers
shell history expansion.

---

## Features

23 subcommands covering the whole Beeper Desktop API surface:

**Read** — `accounts`, `chats`, `find-chat`, `chat`, `messages`, `search-messages`,
`search`, `contacts`, `list-contacts`, `download`, `info`

**Write** — `send`, `edit`, `react`, `unreact`, `new-chat`, `archive`, `unarchive`,
`remind`, `unremind`, `upload`, `focus`

**Escape hatch** — `raw <METHOD> <path>` for any endpoint the CLI doesn't wrap.

Extras: auto-pagination past the API's 20-message page limit, ISO8601 → epoch
conversion for reminders, attachment download straight from the local cache, HTML
flattening in formatted output, and account-scoped filtering on every read command.

Exit codes: `0` success, `1` API/HTTP error (message on stderr), `2` Beeper Desktop
unreachable.

---

## Alternative: MCP

Beeper Desktop also ships an MCP server (streamable HTTP) at
`http://localhost:23373/v0/mcp`. Use it instead of (or alongside) the CLI if your
client speaks Model Context Protocol.

### Claude Code

```bash
claude mcp add beeper http://localhost:23373/v0/mcp -t http -s user
```

`-s user` registers it for every project on this machine. Claude Code runs the OAuth
flow against Beeper Desktop on first use (`/mcp` to check status).

### Cursor / VS Code / Windsurf / Claude Desktop

Add to the client's MCP config (`~/.cursor/mcp.json`, `.vscode/mcp.json`, or
`claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "beeper": {
      "type": "http",
      "url": "http://localhost:23373/v0/mcp"
    }
  }
}
```

### Clients that cannot do OAuth

Pass the bearer token as a static header instead:

```json
{
  "mcpServers": {
    "beeper": {
      "type": "http",
      "url": "http://localhost:23373/v0/mcp",
      "headers": {
        "Authorization": "Bearer bdapi_xxxxxxxxxxxx"
      }
    }
  }
}
```

The same header works for direct REST calls:

```bash
curl -H "Authorization: Bearer $BEEPER_ACCESS_TOKEN" \
     http://localhost:23373/v1/accounts
```

If a client only supports stdio MCP servers, bridge it with `mcp-remote`:

```json
{
  "mcpServers": {
    "beeper": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "http://localhost:23373/v0/mcp"]
    }
  }
}
```

**CLI vs MCP:** the CLI works anywhere a shell does and costs no context until called;
MCP gives structured tool definitions to clients that prefer them. They can coexist.

---

## Security

- **The token grants full read *and* write access to every chat on every connected
  account.** Anyone holding it can read your entire message history and send messages
  as you.
- Keep it on the machine that runs Beeper. Never commit it, paste it into a chat, or
  send it to a remote service.
- The `token` file is git-ignored here — but if you keep your skills directory in a
  **dotfiles repo, add an ignore rule there too**. That is the usual way this leaks.
- `chmod 600` the token file so other local users can't read it.
- **Rotate** in Beeper Desktop → Settings → Integrations: revoke the old token, create
  a new one, rewrite the `token` file, and re-run `beeper accounts` to verify. MCP
  clients that used OAuth re-authorize themselves; clients using the static
  `Authorization` header must be updated by hand.
- A rejected token produces `Token rejected — regenerate in Beeper Desktop → Settings
  → Integrations and update <path to your token file>`.
- Agents driving this CLI should never echo or store OTPs, PINs, or 2FA codes found in
  messages — see the safety rules in `SKILL.md`.

---

## API surface (for building your own client)

Full OpenAPI 3.1 spec: `GET http://localhost:23373/v1/spec` (also reported as
`endpoints.spec` by `GET /v1/info`).

Endpoints: `/v1/accounts`, `/v1/chats` (GET list, POST create), `/v1/chats/search`,
`/v1/chats/{chatID}`, `/v1/chats/{chatID}/archive`, `/v1/chats/{chatID}/reminders`
(POST/DELETE), `/v1/chats/{chatID}/messages` (GET/POST),
`/v1/chats/{chatID}/messages/{messageID}` (PUT),
`/v1/chats/{chatID}/messages/{messageID}/reactions` (POST/DELETE),
`/v1/messages/search`, `/v1/search`, `/v1/accounts/{accountID}/contacts[/list]`,
`/v1/assets/download`, `/v1/assets/serve`, `/v1/assets/upload[/base64]`, `/v1/focus`,
`/v1/info`.

There is also a WebSocket live feed at `/v1/ws` (same bearer token; send
`subscriptions.set` with `chatIDs: ["*"]`, receive `chat.upserted`, `chat.deleted`,
`message.upserted`, `message.deleted`).

---

## License

MIT — see [LICENSE](LICENSE).
