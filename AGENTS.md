# AGENTS.md — Loop Operating Instructions

## Every Session

1. Read `SOUL.md` — your standard and mindset
2. Read `USER.md` — who you're helping
3. Read `context/SETUP-STATE.md` — what's installed, what's configured, what's running
4. Read `context/LEARNINGS.md` — accumulated knowledge about ADOM-LOOP behavior
5. Read `context/GOTCHAS.md` — known failure modes and workarounds
6. Read `memory/YYYY-MM-DD.md` for today and yesterday (if they exist)
7. Proceed with the task

---

## Tasks You Do

### 1. Setup Walkthrough
Guide the user through the full local installation. See `playbooks/SETUP.md` for the step-by-step.

Check first:
```bash
python3 --version        # need 3.12+
node --version           # need 18+
docker info              # need Docker Desktop running
claude --version         # need Claude Code CLI
az --version             # need Azure CLI (for provisioning)
```

### 2. Start ADOM-LOOP

**macOS workaround (start.py exits immediately on macOS — run separately):**
```bash
# Backend
cd ~/path/to/ADOM-LOOP/dashboard/backend
~/path/to/ADOM-LOOP/.venv/bin/python -m uvicorn main:app --host 0.0.0.0 --port 3001

# Frontend (separate terminal)
cd ~/path/to/ADOM-LOOP/dashboard/frontend
npx vite --port 3000
```

Open http://localhost:3000

### 3. Create a New App
Walk the user through the dashboard flow:
1. Name the app and give a description
2. Go through the AI interview (spec refinement) — see `playbooks/AGENTIC-SPEC-PROMPTING.md`
3. Review the architecture plan
4. Provision Azure (or skip for local-only)
5. Watch the build agent run in the live preview

**Agentic-first prompting guide:** See `playbooks/AGENTIC-SPEC-PROMPTING.md`

### 4. Monitor a Running Build
```bash
# Check running containers
docker compose -f ~/path/to/ADOM-LOOP/workspace/<app-name>/docker-compose.dev.yml ps

# Watch backend logs live
docker compose -f ~/path/to/ADOM-LOOP/workspace/<app-name>/docker-compose.dev.yml logs -f backend

# Check phase progress
cat ~/path/to/ADOM-LOOP/workspace/<app-name>/.claude/phases/*.md
```

### 5. Troubleshoot a Failed Build
Common issues documented in `context/GOTCHAS.md`. Quick checks:
```bash
# Check for template placeholders (self-heal bug)
grep -r '{{PORT}}' workspace/<app>/docker-compose.dev.yml

# Verify Claude Code token
claude setup-token    # if expired, regenerate and update .env
```

### 6. Interact with a Generated App via API
Every generated app has a FastAPI backend at `http://localhost:<port>/docs` (Swagger UI).
Prefer curl/API calls over browser UI for all management tasks.
```bash
# Health check
curl http://localhost:<backend-port>/health

# List records
curl http://localhost:<backend-port>/api/<resource>/

# Create a record
curl -X POST http://localhost:<backend-port>/api/<resource>/ \
  -H "Content-Type: application/json" \
  -d '{"field": "value"}'
```

### 7. Update Learnings
After any significant discovery, update:
- `context/LEARNINGS.md` — what works, patterns, tips
- `context/GOTCHAS.md` — what breaks and how to fix it
- `memory/YYYY-MM-DD.md` — session-level notes

---

## Key Files & Locations

| File | Purpose |
|------|---------|
| `context/SETUP-STATE.md` | Current install status, tokens, paths |
| `context/LEARNINGS.md` | Accumulated knowledge about ADOM-LOOP |
| `context/GOTCHAS.md` | Known issues and workarounds |
| `context/APPS.md` | Registry of generated apps (name, port, status) |
| `playbooks/SETUP.md` | Full step-by-step local setup guide |
| `playbooks/AGENTIC-SPEC-PROMPTING.md` | How to prompt for agent-friendly app configs |
| `memory/YYYY-MM-DD.md` | Session notes |

---

## Reporting Back

Use this structure when summarizing work:

```
🔁 Loop — [task summary]

STATUS: [what's running / what's done]
NEXT STEP: [what the user needs to do physically, if anything]
NOTES: [anything worth capturing]
```

Always separate:
- **User must do this** (physical actions: run a command, click something, enter a token)
- **I can do this for you** (file reads, API calls, status checks, drafting prompts)
