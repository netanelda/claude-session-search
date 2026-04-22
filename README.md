# claude-session-search

A [Claude Code skill](https://docs.anthropic.com/en/docs/claude-code/skills) that lets you search across all your past Claude Code sessions by keyword or natural language, summarize them in place, and optionally jump to any session in VS Code.

Claude Code doesn't have built-in session search. This skill fills that gap.

## What it does

1. **Search** - Finds sessions matching your query across `.jsonl` transcript files in `~/.claude/projects/`
2. **Context** - Shows each match with its date, project, VS Code title, first message, and matching snippets
3. **Summarize** - Reads a session transcript and summarizes it right in the current conversation — no tab switching needed
4. **Jump** - Optionally opens any session directly in VS Code via deep link when you want to resume it

## Demo

```
You: /session-search the conversation about migrating to the new API

Claude: Found 2 session(s):

### 1. ~/my-project "Migrate auth endpoints to v2 API"
   Session: `a1b2c3d4-...`
   Time: 2025-11-15 14:32 EST — 2025-11-15 16:05 EST
   Started with: We need to migrate the auth endpoints from v1 to v2...
   Matches: 6
   > [YOU] (2025-11-15 14:35 EST) The v2 API requires a different token format...
   > [CLAUDE] (2025-11-15 14:38 EST) I've updated the auth middleware to handle both v1 and v2 tokens...

### 2. ~/my-project "API rate limiting research"
   Session: `e5f6g7h8-...`
   Time: 2025-11-03 09:12 EST — 2025-11-03 09:45 EST
   Started with: What rate limiting options does the new API support?
   Matches: 2
   > [YOU] (2025-11-03 09:15 EST) Does the new API have built-in rate limiting?

Say "summarize #1" to read a summary here, or "open #1" to jump to it in VS Code.

You: summarize #1

Claude:
- **Topic**: Migrating auth endpoints from v1 to v2 API
- **Key changes**: Updated token format, added backward compatibility layer, new error handling
- **Decision**: Keep v1 endpoints alive for 30 days during transition
- **Left off**: Need to update integration tests for v2 format

Say "open #1" if you want to resume this session.
```

## Usage

```
/session-search exchange rate
/session-search --global auth bug
/session-search -p my-project template rendering
/session-search --list
/session-search find the conversation where we discussed switching databases
```

### Scope

| Flag | Description |
|------|-------------|
| _(none)_ | Search within the **current project's** sessions (default) |
| `--global` / `-g` | Search across all projects |
| `--project <name>` / `-p <name>` | Search a specific project (partial match) |
| `--list` | List all projects and their session counts |

### Natural language

You don't need exact keywords. The skill decomposes natural language queries into multiple search terms and cross-references results:

```
/session-search the conversation where we fixed the login bug
```

This searches for "login", "auth", "bug", "fix" etc. and ranks sessions that match multiple terms higher.

### Summarize vs Open

After results are shown, you have two options:

| Command | What it does |
|---------|-------------|
| **"summarize #2"** (default) | Reads the session and shows a summary right here — you never leave this conversation |
| **"open #2"** | Opens the session as a VS Code tab — use when you want to resume/continue that conversation |

Summarize is the recommended default. It lets you browse multiple sessions without losing your place. Only open a session when you've found the one you want to continue working in.

### Tab behavior (when opening)

- If a session is **already open** in a tab, it jumps to that tab (no duplicate)
- If it's **not open**, a new tab is created
- Your search conversation **stays open** — press **Cmd+Shift+[** to get back to it

## Install

Copy the skill to your Claude Code skills directory:

```bash
mkdir -p ~/.claude/skills/session-search
curl -o ~/.claude/skills/session-search/SKILL.md \
  https://raw.githubusercontent.com/netanelda/claude-session-search/main/SKILL.md
```

Or clone and symlink:

```bash
git clone https://github.com/netanelda/claude-session-search.git
ln -s "$(pwd)/claude-session-search" ~/.claude/skills/session-search
```

## How it works

- **Session transcripts** are stored as `.jsonl` files in `~/.claude/projects/<project-dir>/`
- **VS Code session titles** are in `~/Library/Application Support/Claude/claude-code-sessions/` (macOS)
- The skill maps CLI session IDs to VS Code titles for richer results
- The deep link `vscode://anthropic.claude-code/open?session=<CLI_SESSION_ID>` is an undocumented feature of the Claude Code VS Code extension that opens (or reveals) a session
- The search itself (grepping `.jsonl` files) works on any platform, but the jump-to-session deep link requires VS Code with the Claude Code extension

## Requirements

- Claude Code (CLI or VS Code extension)
- Python 3 (used for searching `.jsonl` files)
- macOS (for `open` command and VS Code session metadata path — Linux/Windows users would need to adjust the VS Code metadata path and `open` command)

## Limitations

- Search is substring-based, not semantic. The AI layer (natural language decomposition) helps but ultimately runs grep-style matches.
- Large session files (1MB+) take a moment to scan. Projects with hundreds of sessions may take a few seconds.
- VS Code session titles are only available for sessions that were opened in VS Code (not CLI-only sessions).
- Opening a session switches focus to its tab. Use **Cmd+Shift+[** to return to the search conversation.
- The `open` deep link is undocumented and may change in future Claude Code versions.

## License

MIT
