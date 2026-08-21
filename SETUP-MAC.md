# Setup: Claude Desktop on macOS

This is a patched fork of [skylight-mcp](https://github.com/TheEagleByte/skylight-mcp). It
diverges from the upstream project in ways that matter for setup:

- **Email/password login is dead upstream.** Skylight's backend rejects it with an
  app-version error regardless of correct credentials. This fork only supports the manual
  `SKYLIGHT_TOKEN` auth path below.
- **Plus-gated tools (meals, rewards, photos)** need `SKYLIGHT_HAS_PLUS=true` set manually —
  manual-token auth has no way to detect Plus status on its own.
- **Chore create/update/delete** are patched to match the real Skylight API, which uses a
  different (non-JSON:API, sometimes bulk-only) request shape than every other endpoint. See
  `CLAUDE.md` in this repo for the details if you're debugging.

Follow these steps in order.

## 1. Install a JS runtime

This fork is built and tested with [Bun](https://bun.sh), not Node/npm. Install it:

```bash
curl -fsSL https://bun.sh/install | bash
```

This installs `bun` to `~/.bun/bin/bun`. Restart your terminal (or `source ~/.zshrc` /
`source ~/.bashrc`) so `bun` is on your `PATH`, then confirm:

```bash
bun --version
```

## 2. Clone and build

```bash
git clone https://github.com/samabenie1/skylight-mcp-fork.git ~/Projects/skylight-mcp
cd ~/Projects/skylight-mcp
bun install
bun run build
```

This produces `dist/index.js`, which is what Claude Desktop actually runs. `dist/` is
gitignored — you must run `bun run build` after every `git pull` that touches `src/`.

## 3. Get your Skylight credentials

You need two values from your own logged-in Skylight session (not a shared secret — Bearer
tokens are short-lived and tied to a browser session):

**`SKYLIGHT_TOKEN`:**
1. Open [app.ourskylight.com](https://app.ourskylight.com) in a browser and log in.
2. Open DevTools → Network tab.
3. Trigger any action (e.g. click into Chores) so a request to `/api/frames/...` appears.
4. Click that request → Headers → find `Authorization: Bearer <token>`.
5. Copy everything after `Bearer ` — that's your `SKYLIGHT_TOKEN`.

**`SKYLIGHT_FRAME_ID`:**
- In the same request, look at the URL path: `/api/frames/{frameId}/...`. That numeric ID is
  your frame ID.

Tokens expire. If the server starts failing auth after working fine before, re-extract a
fresh token via the same steps — this is expected, not a bug.

## 4. Configure Claude Desktop

Edit (or create) `~/Library/Application Support/Claude/claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "skylight": {
      "command": "/Users/<your-username>/.bun/bin/bun",
      "args": ["/Users/<your-username>/Projects/skylight-mcp/dist/index.js"],
      "env": {
        "SKYLIGHT_TOKEN": "<paste your token>",
        "SKYLIGHT_AUTH_TYPE": "bearer",
        "SKYLIGHT_FRAME_ID": "<paste your frame id>",
        "SKYLIGHT_TIMEZONE": "America/New_York",
        "SKYLIGHT_HAS_PLUS": "true"
      }
    }
  }
}
```

Replace `<your-username>` with your actual macOS username (`whoami`), and adjust the clone
path if you put the repo somewhere other than `~/Projects/skylight-mcp`. Set
`SKYLIGHT_TIMEZONE` to your actual timezone (IANA name, e.g. `America/Chicago`,
`America/Los_Angeles`). Set `SKYLIGHT_HAS_PLUS` to `"false"` if this Skylight account doesn't
have a Plus subscription.

If the file already has other `mcpServers` entries, add `"skylight"` as another key inside the
existing `mcpServers` object rather than replacing the whole file.

## 5. Fully restart Claude Desktop

**Quit the app completely — Cmd+Q, not just closing the window** — then reopen it. Closing the
window on macOS typically leaves the app running in the background with its old process (and
old env config) still alive; Cmd+Q is what actually kills and restarts it so it re-reads the
config file. If you're not sure it worked, check Activity Monitor for a stale `Claude` process
before reopening.

## 6. Verify

Ask Claude something like "what chores are on the Skylight chart today?" — if it responds with
real data, the connection is working. If you get an auth error, re-check your token (step 3)
and that Claude Desktop actually restarted (step 5).

## Keeping this up to date

To pull future fixes from this fork:

```bash
cd ~/Projects/skylight-mcp
git pull
bun install
bun run build
```

Then fully quit and reopen Claude Desktop again (step 5) to pick up the rebuilt server.
