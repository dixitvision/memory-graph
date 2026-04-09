---
name: memory-graph
description: >
  A connected knowledge graph that replaces full conversation re-reading with instant context recall.
  Reduces token usage by 70–90% across Cowork, Code, and Chat by storing facts as graph nodes/edges
  on disk and loading only the relevant subgraph per task — never the full history.

  ALWAYS use this skill at the START of any multi-step task to recall what is already known.
  ALWAYS use this skill at the END of any task to store what was learned.
  Trigger on: any task where context from a previous session is needed, "remember this",
  "what did we do last time", "don't re-explore", "save this", "recall", or any task
  where you would otherwise ask the user something you've already been told.
---

# Memory Graph Skill

## The Core Problem

Every time Claude works on a task, it reads the full conversation history — even when
99% of it is irrelevant. A GitHub bio update that took 2 minutes spends 40k tokens
re-reading prior sessions. This skill eliminates that.

## How It Works

Instead of reading the conversation, Claude reads a compact graph:

```
conversation history  →  50,000 tokens  (slow, wasteful)
memory/graph.json     →     400 tokens  (instant, targeted)
```

The graph stores facts as **nodes** (entities) connected by **edges** (relationships).
You only load the nodes relevant to the current task.

---

## File Structure

```
memory/
  graph.json          ← The knowledge graph (source of truth)
  hot-cache.md        ← Top 30 nodes in human-readable form (= CLAUDE.md)
  workflows/
    github.md         ← GitHub task patterns
    browser.md        ← Browser automation patterns
    documents.md      ← File creation patterns
  tools-manifest.md   ← Exact ToolSearch queries per tool category
```

The working directory is wherever CLAUDE.md lives (usually the session root).

---

## Step 1: RECALL — Read Before You Act

At the start of ANY task, before using any tools:

```
1. Read memory/hot-cache.md (or CLAUDE.md)   → ~150 tokens, covers 90% of cases
2. If the answer isn't there → read memory/graph.json and filter by task type
3. If still not found → ask the user ONCE, then store the answer
```

**Never explore what you already know.** If graph says `github: dixitvision`,
don't navigate to GitHub to check. Trust the graph.

### Reading the graph efficiently

Don't load the entire graph. Filter by node type:

```python
import json
graph = json.load(open('memory/graph.json'))

# Get only what you need
user = graph['nodes'].get('user:primary')
repos = {k:v for k,v in graph['nodes'].items() if v['type'] == 'repo'}
last_task = graph['session_log'][-1] if graph['session_log'] else None
```

Or read it with Bash and pipe through python to extract just the relevant field:
```bash
python3 -c "import json,sys; g=json.load(open('memory/graph.json')); print(g['nodes'].get('user:primary',{}).get('github','not found'))"
```

---

## Step 2: ROUTE — Match Task to Workflow

Once you have context, match the task type to a pre-built workflow.
Read the relevant workflow file — don't reinvent the process.

| Task pattern | Workflow file | Token cost |
|---|---|---|
| "update github bio/repo/README" | memory/workflows/github.md | ~300 (API calls) |
| "click/browse/fill form" | memory/workflows/browser.md | ~2,000 (batched) |
| "create doc/pdf/pptx/xlsx" | memory/workflows/documents.md | skill-specific |
| "remember this / save this" | → STORE operation (Step 4) | ~100 |

---

## Step 3: EXECUTE — Use Cached Patterns

Read memory/tools-manifest.md before loading any tools.
It tells you the EXACT ToolSearch query to load each tool category in ONE call.

**Never do this:**
```
ToolSearch("AskUserQuestion")      ← 1 call
ToolSearch("TodoWrite")            ← 2nd call
ToolSearch("tabs_context_mcp")     ← 3rd call
ToolSearch("javascript_tool")      ← 4th call
```

**Always do this (from tools-manifest.md):**
```
ToolSearch("chrome navigate screenshot find", 20)  ← loads ALL Chrome tools at once
```

### Screenshot discipline

Maximum 3 screenshots per task:
1. After navigation — assess the page state
2. After your batch of actions — verify changes landed
3. Final state — confirm completion

Use `find()` + refs to click elements. Never screenshot just to find coordinates.

### Batch browser actions

Use `computer_batch` (available in computer-use MCP) to combine sequences:
```
[click field] + [ctrl+a] + [type new value] + [click save]  = 1 round trip
```

---

## Step 4: STORE — Write Back to Graph

At the END of every task, update the graph. This is what makes the system compound —
each task makes the next one cheaper.

### What to store

- **New entities discovered**: new repos, new people, new URLs, new credentials paths
- **Task completion**: what was done, date, outcome
- **Corrections**: if user said "actually my username is X not Y" — update immediately
- **Patterns that worked**: if you found a faster way, add it to the relevant workflow file

### How to update

```python
import json, datetime

with open('memory/graph.json', 'r') as f:
    graph = json.load(f)

# Add a node
graph['nodes']['repo:new-project'] = {
    "type": "repo",
    "name": "new-project",
    "owner": "dixitvision",
    "visibility": "public",
    "lang": "Python",
    "description": "...",
    "topics": []
}

# Log the task
graph['session_log'].append({
    "date": datetime.date.today().isoformat(),
    "task": "created repo new-project",
    "status": "done"
})

graph['updated'] = datetime.date.today().isoformat()

with open('memory/graph.json', 'w') as f:
    json.dump(graph, f, indent=2)
```

Then update CLAUDE.md / hot-cache.md if the new node is likely to be needed frequently.

---

## Graph Node Types

| Type | Key fields | Example id |
|------|-----------|------------|
| `user` | name, email, github, linkedin | `user:primary` |
| `repo` | name, owner, lang, topics, description | `repo:omnimind` |
| `task` | date, summary, status | `task:2026-04-09-github` |
| `service` | name, url, auth_method | `service:github` |
| `workflow` | name, steps, token_cost | `workflow:github-api` |
| `tool` | name, load_query, category | `tool:chrome-mcp` |
| `project` | name, status, stack | `project:marketing-capstone` |

---

## Initializing the Graph for a New User/Project

When no graph.json exists yet, create it by interviewing the user:

1. Ask: "What's your name, and what are the main services/tools you use?"
2. Ask: "What are the 2-3 projects you're currently working on?"
3. Ask: "What should I remember from our last session?" (if applicable)
4. Write graph.json with the answers as nodes
5. Write CLAUDE.md hot-cache from the same data

Do NOT ask more than 5 questions. Build from what the user says, not an exhaustive intake form.

---

## Token Budget (Target)

| Task type | Old cost | With skill | Savings |
|-----------|---------|------------|---------|
| Recall user context | 5,000 | 150 | 97% |
| GitHub bio/repo update | 12,000 | 500 | 96% |
| Browser form fill | 15,000 | 3,000 | 80% |
| Create a document | 8,000 | 2,000 | 75% |
| Load tools | 3,000 | 300 | 90% |

---

## Edge Cases

**Graph is wrong / outdated**: If the user corrects you ("that repo was renamed"), 
update the node immediately and add a `corrected_at` timestamp. Never argue — just fix.

**Graph doesn't have it**: Say "I don't have [X] in memory yet. What is it?" 
Store the answer before continuing.

**Multiple users / projects**: Namespace node ids: `user:alice`, `user:bob`, 
`project:client-a`, `project:client-b`.

**Large graphs**: Keep hot-cache.md under 100 lines. Anything older than 30 days
with no access gets moved to an `archive/` section of graph.json.
