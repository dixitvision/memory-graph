# Workflow Templates

Pre-built patterns for common task types. Load only the section you need.

---

## GitHub — API Pattern (300 tokens, no browser)

**When to use:** Updating bio, repo descriptions, topics, README files, creating repos.

```bash
# STEP 1: Check if user has a GitHub token stored
# If not, ask once and save to graph.json as service:github.token

# Update profile bio
curl -s -X PATCH https://api.github.com/user \
  -H "Authorization: token $GITHUB_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"bio\": \"$BIO_TEXT\"}"

# Update repo description + topics
curl -s -X PATCH https://api.github.com/repos/$OWNER/$REPO \
  -H "Authorization: token $GITHUB_TOKEN" \
  -d "{\"description\": \"$DESC\"}"

curl -s -X PUT https://api.github.com/repos/$OWNER/$REPO/topics \
  -H "Authorization: token $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github.mercy-preview+json" \
  -d "{\"names\": [\"python\", \"ai\"]}"

# Create or update a file (e.g., README.md)
CONTENT=$(python3 -c "import base64,sys; print(base64.b64encode(open('$FILE','rb').read()).decode())")
SHA=$(curl -s https://api.github.com/repos/$OWNER/$REPO/contents/$FILE_PATH \
  -H "Authorization: token $GITHUB_TOKEN" | python3 -c "import json,sys; d=json.load(sys.stdin); print(d.get('sha',''))")

curl -s -X PUT https://api.github.com/repos/$OWNER/$REPO/contents/$FILE_PATH \
  -H "Authorization: token $GITHUB_TOKEN" \
  -d "{\"message\": \"Update $FILE_PATH\", \"content\": \"$CONTENT\", \"sha\": \"$SHA\"}"
```

**If no token available:** Fall back to browser workflow below, but flag to user that
providing a GitHub token would reduce future token costs by ~95%.

---

## Browser Automation — Batched Pattern (2,000–3,000 tokens)

**When to use:** OAuth flows, visual verification, sites with no API, form fills.

```
SETUP (once per session):
  ToolSearch("chrome navigate screenshot find", 20)  → loads all Chrome MCP tools
  tabs_context_mcp(createIfEmpty: true)               → get tab ID

NAVIGATE (1 tool call):
  navigate(tabId, "https://exact.url/path")           → go directly, no menu clicking

ASSESS (1 screenshot):
  computer(screenshot)                                → see page state, plan actions

EXECUTE (1 batch call):
  Use find() to get element refs
  computer_batch([
    {action: "left_click", ref: "ref_123"},
    {action: "key", text: "ctrl+a"},
    {action: "type", text: "new value"},
    {action: "left_click", ref: "ref_save_button"}
  ])

VERIFY (1 screenshot):
  computer(screenshot)                                → confirm changes saved

TOTAL: 5 tool calls max for a form update
```

---

## Document Creation — Skill Pattern (2,000 tokens)

**When to use:** Creating .docx, .pptx, .pdf, .xlsx files.

```
1. Read relevant skill SKILL.md FIRST (docx/pptx/pdf/xlsx)
2. Write content to /sessions/.../mnt/outputs/ (skip temp files)
3. Single Bash execution for the file creation script
4. Link the output file
```

Never write file content twice (once to temp, once to output). Write once, to outputs.

---

## Memory Read Pattern (150 tokens)

**When to use:** Start of ANY task.

```bash
# Hot cache (covers 90% of cases)
cat /sessions/peaceful-great-cori/CLAUDE.md

# Specific node from graph (when hot cache isn't enough)
python3 -c "
import json
g = json.load(open('/sessions/peaceful-great-cori/memory/graph.json'))
# Get specific node
print(json.dumps(g['nodes'].get('user:primary', {}), indent=2))
"

# Search graph by type
python3 -c "
import json
g = json.load(open('/sessions/peaceful-great-cori/memory/graph.json'))
repos = {k:v for k,v in g['nodes'].items() if v.get('type')=='repo'}
print(json.dumps(repos, indent=2))
"
```

---

## Memory Write Pattern (100 tokens)

**When to use:** End of ANY task, or when user says "remember this".

```python
import json, datetime

with open('/sessions/peaceful-great-cori/memory/graph.json', 'r') as f:
    g = json.load(f)

# Add/update a node
g['nodes']['KEY'] = {
    "type": "TYPE",
    "field": "value",
    # ...
}

# Add an edge
g['edges'].append({"from": "user:primary", "to": "KEY", "rel": "RELATIONSHIP"})

# Log the session task
g['session_log'].append({
    "date": datetime.date.today().isoformat(),
    "task": "description of what was done",
    "status": "done"
})

g['updated'] = datetime.date.today().isoformat()

with open('/sessions/peaceful-great-cori/memory/graph.json', 'w') as f:
    json.dump(g, f, indent=2)

# Also update CLAUDE.md hot cache if this node will be frequently needed
```
