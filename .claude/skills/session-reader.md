You are an agent that reads and analyzes Claude Code session logs.

## Skill: Session Reader

**Name:** session-reader
**When to use:** Any time you need to retrieve what happened in a previous session — failures, tool calls, decisions, patterns — to inform the current task.

---

## File Layout

```
~/.claude/
  history.jsonl                     ← index of all sessions (one line per prompt)
  projects/
    <project-slug>/                 ← one dir per working directory
      <session-uuid>.jsonl          ← one file per conversation
```

**Project slug** = working directory path with every `/` replaced by `-`, prepended with `-`.
Example: `/mnt/c/Users/Jumana Sakhnini/Repository/browse-site-optimizer`
→ `-mnt-c-Users-Jumana-Sakhnini-Repository-browse-site-optimizer`

There is **no** `.claude-sessions` directory and **no** `audit.jsonl`.

---

## Record Types in Session JSONL

| type | Contains |
|------|----------|
| `user` | User message or tool result |
| `assistant` | AI response (text + tool calls) |
| `ai-title` | Auto-generated session title |
| `system` | Duration metrics |

`assistant` content blocks: `text`, `tool_use`, `thinking`
`user` content: string (user message) OR array of `tool_result` blocks

---

## Procedure

### Find recent sessions
```bash
python3 -c "
import json, os, datetime
with open(os.path.expanduser('~/.claude/history.jsonl')) as f:
    entries = [json.loads(l) for l in f if l.strip()]
entries.sort(key=lambda x: x['timestamp'], reverse=True)
for e in entries[:20]:
    ts = datetime.datetime.fromtimestamp(e['timestamp']/1000).strftime('%Y-%m-%d %H:%M')
    print(f'{ts}  {e[\"sessionId\"][:8]}  {e[\"display\"][:60]}')
"
```

### Read a specific session
```bash
python3 << 'EOF'
import json, os
path = os.path.expanduser("~/.claude/projects/<slug>/<uuid>.jsonl")
with open(path) as f:
    for line in f:
        obj = json.loads(line)
        t = obj.get('type')
        if t == 'user':
            content = obj['message']['content']
            if isinstance(content, str):
                print(f"[USER] {content[:200]}")
        elif t == 'assistant':
            for block in obj['message'].get('content', []):
                if block.get('type') == 'text':
                    print(f"[ASSISTANT] {block['text'][:300]}")
                elif block.get('type') == 'tool_use':
                    print(f"[TOOL {block['name']}] {str(block.get('input',''))[:100]}")
EOF
```

### Find the current session (most recent file by mtime)
```bash
python3 -c "
import os, glob
base = os.path.expanduser('~/.claude/projects')
files = glob.glob(f'{base}/**/*.jsonl', recursive=True)
files = [f for f in files if not f.endswith('history.jsonl')]
latest = max(files, key=os.path.getmtime)
print(latest)
"
```

### Extract tool errors and failures from a session
```bash
python3 << 'EOF'
import json, os
path = os.path.expanduser("~/.claude/projects/<slug>/<uuid>.jsonl")
with open(path) as f:
    for line in f:
        obj = json.loads(line)
        if obj.get('type') == 'user':
            content = obj['message']['content']
            if isinstance(content, list):
                for block in content:
                    if isinstance(block, dict) and block.get('type') == 'tool_result':
                        if block.get('is_error'):
                            print(f"[ERROR] tool_use_id={block.get('tool_use_id')} content={str(block.get('content',''))[:300]}")
EOF
```

### Search across sessions for a keyword
```bash
python3 -c "
import os, glob
base = os.path.expanduser('~/.claude/projects')
for proj in os.listdir(base):
    for fpath in glob.glob(f'{base}/{proj}/*.jsonl'):
        with open(fpath) as f:
            if 'KEYWORD' in f.read():
                print(fpath)
"
```

---

## Anti-patterns to avoid

- Looking in `.claude-sessions/` — does not exist
- Looking for `audit.jsonl` — does not exist
- Using `timestamp / 1000` for `history.jsonl` but forgetting session JSONL uses ISO 8601 strings
- Assuming `content` is always a string — it can be a list of `tool_result` blocks
- Hardcoding a session UUID — always find the right session dynamically via history.jsonl or mtime
- Printing entire session files — truncate with `[:300]` per block to avoid token overflow

---

## After Each Run (always execute this — no exceptions)

After completing the task above, immediately do the following without waiting to be asked:

1. Use the Session Reader Skill to read the current session log.
2. Compare what actually happened against what this skill predicted or instructed.
3. For each difference found, update the relevant section of this skill file:
   - Wrong file paths or slug encoding → fix immediately
   - New record types encountered → add to the table
   - Bash scripts that failed → fix or replace
4. Add one line to the `## Revision History` section: today's date, what changed, and why.

If nothing new was learned, add a one-line history entry confirming the skill behaved as expected.

## Revision History

- 2026-06-29 — Rewrote from outdated version. Previous version referenced `.claude-sessions/` and `audit.jsonl` which do not exist. Corrected file layout, slug encoding, record types, and procedure scripts.
- 2026-06-29 — Improvement pass: added "find current session" script (latest file by mtime), added "extract tool errors" script for failure analysis, added anti-patterns for hardcoded UUIDs and token overflow.
