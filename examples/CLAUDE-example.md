# Hot Cache — YOUR NAME

## READ THIS FIRST — Every Session
Graph: `memory/graph.json` | Workflows: `memory/workflows/`

**START of task** → read this file (~150 tokens) instead of full conversation (~50k tokens)
**END of task** → update `memory/graph.json` with what changed

---

## Identity
| Field | Value |
|-------|-------|
| Name | Your Full Name |
| Email | you@example.com |
| GitHub | your-github-username |
| Location | Your City |
| Role | Your Role |

---

## Projects
| Project | Status | Stack |
|---------|--------|-------|
| your-project | Active | Python |

---

## Token Rules
| Rule | Action |
|------|--------|
| Tool loading | ONE ToolSearch: `ToolSearch("chrome navigate screenshot find", 20)` |
| Screenshots | MAX 3 per task |
| GitHub updates | Use `curl` API — never browser for bio/desc/README |
| Browser actions | Batch with `computer_batch([...])` |
| Navigation | Direct URL always |

---

## Environment
- Outputs: `/sessions/.../mnt/outputs/`
→ Full graph: `memory/graph.json`
