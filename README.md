# InstaShare Skill for Claude Code

A [Claude Code Skill](https://code.claude.com/docs/en/skills) that uploads the **current** Claude Code chat session to [instashare.to](https://instashare.to) and returns a public share link — in one shell call.

Ask Claude things like:

- "share this chat"
- "make a link to this conversation"
- "InstaShare this"
- "publish this session"

…and you get back a short URL like `https://instashare.to/c/aB3xQ7` and a one-shot delete URL.

## What it does

- Uploads the session the agent is actually running in, identified by its **session id** (`~/.claude/projects/<slug>/<session-id>.jsonl`). Without an id it falls back to the newest transcript by mtime, but aborts rather than guessing when several were touched recently — a sibling session in the same working directory can be newer than yours.
- POSTs the file to the InstaShare API, returns `{ url, deleteUrl }`.
- On reruns, **reuses the same URL** via a sidecar file — links you already pasted keep showing the latest transcript.
- Prints both the public URL and the revoke URL so the user stays in control.

The InstaShare server is open and the source is at [Instafill/instashare](https://github.com/Instafill/instashare) (Next.js + MongoDB).

## Install

### Option 1: Plugin marketplace (recommended)

In Claude Code:

```
/plugin marketplace add Instafill/instashare-skill
/plugin install instashare@instashare
```

### Option 2: Copy the skill folder

```bash
# macOS / Linux
git clone https://github.com/Instafill/instashare-skill.git
cp -r instashare-skill/skills/instashare ~/.claude/skills/
```

```powershell
# Windows
git clone https://github.com/Instafill/instashare-skill.git
Copy-Item -Recurse instashare-skill\skills\instashare $env:USERPROFILE\.claude\skills\
```

Restart Claude Code (or open a new session) and the `instashare` skill becomes available.

## Configuration

| Env var | Default | Purpose |
| --- | --- | --- |
| `INSTASHARE_API_URL` | `https://instashare.to` | Override the API endpoint — e.g. `http://localhost:3000` when developing the server locally. |

No credentials. Anyone with the printed URL can read the chat; anyone with the delete URL can revoke it.

## Privacy

Shared transcripts are public to anyone with the link. They contain verbatim tool inputs and outputs — file contents Claude read, command outputs, etc. Review before sharing, and use the printed delete URL to revoke a share you didn't mean to publish.

## License

Apache-2.0. See [LICENSE](./LICENSE).
