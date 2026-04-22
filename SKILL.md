---
name: session-search
description: Search across Claude Code session transcripts for keywords or concepts, show matches with context, summarize sessions in place, and optionally jump to any session in VS Code
user_invocable: true
---

# Session Search

Search Claude Code session history, summarize found sessions without leaving the current conversation, and optionally open them in VS Code.

## Scope

- **Default: current project only.** Search sessions in the `~/.claude/projects/` subdirectory that matches the current working directory.
- `--global` or `-g` = search across ALL projects' sessions.
- `--project <name>` or `-p <name>` = search a specific project folder (partial match on the directory name).
- `--list` = list all projects and their session counts, no search.

To determine the current project directory: take the current working directory, replace `/` with `-`, and look for a matching folder under `~/.claude/projects/`. For example, `/Users/jane/my-project` becomes `-Users-jane-my-project`.

## Natural Language Queries

The user's input may be a natural language description rather than an exact keyword. When you receive the query:

1. **Decompose** the query into 2-5 likely search terms. Think about synonyms and related words.
   - "the conversation where we fixed the auth bug" -> search for: "auth", "authentication", "login", "bug", "fix"
   - "when we discussed switching to Postgres" -> search for: "postgres", "postgresql", "database", "migration"
2. **Run multiple searches in parallel** using the script below, one per search term.
3. **Cross-reference results**: sessions that match multiple terms are ranked higher.
4. **Present the best matches** — deduplicate sessions that appear in multiple searches, note how many terms each matched.

If the query looks like an exact keyword or phrase (short, specific, no filler words), just search for it directly.

## Search Script

Run this Python script via Bash. Substitute the search term, and set the project_filter based on scope:

```bash
python3 << 'PYEOF'
import json, os, glob
from datetime import datetime, timezone

query = "SEARCH_TERM_HERE"
project_filter = "PROJECT_DIR_FILTER_HERE"  # full dir name for project scope, empty string for global

base = os.path.expanduser("~/.claude/projects")
vscode_sessions_dir = os.path.expanduser("~/Library/Application Support/Claude/claude-code-sessions")

# Build a lookup: CLI session ID -> VS Code title
cli_to_title = {}
if os.path.isdir(vscode_sessions_dir):
    for root, dirs, files in os.walk(vscode_sessions_dir):
        for fname in files:
            if fname.endswith(".json"):
                try:
                    with open(os.path.join(root, fname)) as jf:
                        meta = json.load(jf)
                        cli_id = meta.get("cliSessionId", "")
                        title = meta.get("title", "")
                        if cli_id and title:
                            cli_to_title[cli_id] = title
                except:
                    pass

def fmt_ts(iso_str):
    if not iso_str:
        return "unknown"
    try:
        dt = datetime.fromisoformat(iso_str.replace("Z", "+00:00")).astimezone()
        return dt.strftime("%Y-%m-%d %H:%M %Z")
    except:
        return iso_str[:16]

results = []

for jsonl_path in sorted(glob.glob(os.path.join(base, "*", "*.jsonl"))):
    project_dir = os.path.basename(os.path.dirname(jsonl_path))

    if project_filter and project_filter.lower() not in project_dir.lower():
        continue

    session_id = os.path.basename(jsonl_path).replace(".jsonl", "")
    first_ts = None
    last_ts = None
    first_user_msg = None
    matching_snippets = []

    with open(jsonl_path, "r") as f:
        for line in f:
            try:
                obj = json.loads(line.strip())
            except:
                continue

            ts = obj.get("timestamp")
            if ts:
                if not first_ts:
                    first_ts = ts
                last_ts = ts

            text = ""
            role = ""

            if obj.get("type") in ("user", "assistant"):
                msg = obj.get("message", {})
                content = msg.get("content", "")
                if isinstance(content, list):
                    text = " ".join(
                        c.get("text", "")
                        for c in content
                        if isinstance(c, dict) and c.get("type") == "text"
                    )
                elif isinstance(content, str):
                    text = content
                role = obj["type"]
            elif obj.get("type") == "queue-operation" and obj.get("operation") == "enqueue":
                text = obj.get("content", "")
                role = "user"

            if not text.strip():
                continue

            if role == "user" and not first_user_msg:
                first_user_msg = text.strip()[:150]

            if query.lower() in text.lower():
                snippet = text.strip()
                match_pos = snippet.lower().find(query.lower())
                if len(snippet) > 200:
                    start = max(0, match_pos - 80)
                    snippet = ("..." if start > 0 else "") + snippet[start:start+200] + "..."
                match_ts = ts if ts else ""
                matching_snippets.append((role, snippet, match_ts))

    if matching_snippets:
        proj_name = project_dir
        if project_dir.startswith("-Users-"):
            segments = project_dir.split("-")
            user_segments_end = 3  # -Users-username
            proj_parts = segments[user_segments_end:]
            if proj_parts:
                proj_name = "~/" + "/".join(proj_parts)
            else:
                proj_name = "~ (home)"

        title = cli_to_title.get(session_id, "")

        results.append({
            "project": proj_name,
            "session_id": session_id,
            "started": fmt_ts(first_ts),
            "ended": fmt_ts(last_ts),
            "title": title,
            "first_msg": first_user_msg or "(no user message)",
            "match_count": len(matching_snippets),
            "snippets": matching_snippets[:3],
        })

results.sort(key=lambda r: r["started"], reverse=True)

if not results:
    print(f'No sessions found matching: "{query}"')
else:
    print(f'Found {len(results)} session(s) matching "{query}":\n')
    for i, r in enumerate(results, 1):
        title_line = f' "{r["title"]}"' if r["title"] else ""
        print(f'### {i}. {r["project"]}{title_line}')
        print(f'   Session: `{r["session_id"]}`')
        print(f'   Time: {r["started"]} — {r["ended"]}')
        print(f'   Started with: {r["first_msg"]}')
        print(f'   Matches: {r["match_count"]}')
        for role, snip, ts in r["snippets"]:
            tag = "YOU" if role == "user" else "CLAUDE"
            ts_label = f' ({fmt_ts(ts)})' if ts else ""
            print(f'   > [{tag}]{ts_label} {snip}')
        print()

PYEOF
```

## Presenting Results

Show results clearly. Each match should display:
1. Number, date, project, and VS Code title (if available)
2. The session ID
3. What the session started with (first user message)
4. Up to 3 matching snippets labeled [YOU] or [CLAUDE]

If you ran multiple searches (natural language decomposition), merge and deduplicate results. Sessions matching more terms should appear first. Mention which terms matched.

## After Showing Results

**Default action is summarize, not open.** After presenting results, offer these options:

- **"summarize #2"** (default/recommended) — Read the session transcript and provide a summary right here, without leaving this conversation
- **"open #2"** — Open the session tab in VS Code (only when the user wants to resume or continue that conversation)

Always present summarize as the primary option. For example:
> Say **"summarize #2"** to read a session summary here, or **"open #2"** to jump to it in VS Code.

## Summarizing a Session

When the user asks to summarize a session (e.g. "summarize #2"), extract the conversation from the JSONL file using this script:

```bash
python3 << 'PYEOF'
import json

session_path = "SESSION_JSONL_PATH_HERE"
messages = []

with open(session_path) as f:
    for line in f:
        try:
            obj = json.loads(line.strip())
        except:
            continue

        text = ""
        role = ""

        if obj.get("type") in ("user", "assistant"):
            msg = obj.get("message", {})
            content = msg.get("content", "")
            if isinstance(content, list):
                text = " ".join(
                    c.get("text", "")
                    for c in content
                    if isinstance(c, dict) and c.get("type") == "text"
                )
            elif isinstance(content, str):
                text = content
            role = obj["type"]
        elif obj.get("type") == "queue-operation" and obj.get("operation") == "enqueue":
            text = obj.get("content", "")
            role = "user"

        if text.strip():
            # Truncate very long messages to keep output manageable
            if len(text) > 500:
                text = text[:500] + "..."
            messages.append(f"[{role.upper()}] {text.strip()}")

# Print all messages, capped at a reasonable total
output = "\n---\n".join(messages)
if len(output) > 15000:
    output = output[:15000] + "\n\n... (truncated, session continues)"
print(output)

PYEOF
```

After extracting the conversation, provide a concise summary covering:
1. **What was discussed** — the main topic(s) and goals
2. **What was decided or built** — key outcomes, code changes, decisions made
3. **Where it left off** — was there unfinished work, open questions, or next steps?

Keep the summary to 5-10 bullet points. The user wants to quickly understand what happened in that session, not re-read the whole thing.

After summarizing, remind the user they can say **"open #2"** if they want to resume that session in VS Code.

## Opening a Session

When the user asks to open a session, run:

```bash
open "vscode://anthropic.claude-code/open?session=SESSION_ID_HERE"
```

**Tab behavior:**
- If the session is **already open** in a tab, it simply jumps to that existing tab (no duplicate created).
- If the session is **not open**, a new tab is created with the full conversation history.
- Opening switches focus to the new tab. To get back to this search conversation, press **Cmd+Shift+[** (previous tab) or click this tab in the tab bar.

## File Locations

- **Session transcripts**: `.jsonl` files in `~/.claude/projects/<project-dir>/`
- **VS Code session metadata** (titles): `~/Library/Application Support/Claude/claude-code-sessions/` — JSON files with `cliSessionId` and `title` fields
- The deep link URI uses the **CLI session ID** (bare UUID from the `.jsonl` filename), not the VS Code `local_` prefixed ID
- Session `.jsonl` files can be large (1MB+). Only search and show snippets — never read entire files into context unless summarizing.
