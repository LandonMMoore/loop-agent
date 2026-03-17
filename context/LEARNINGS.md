# LEARNINGS.md — ADOM-LOOP Knowledge Base

*Accumulated from docs, architecture study, and real sessions.*

---

## How ADOM-LOOP Works (The Big Picture)

ADOM-LOOP is a 6-step pipeline that turns a description into a running full-stack app:

1. **Create** — Name + describe the app. Templates scaffold, Docker starts in background.
2. **Interview** — AI agent asks clarifying questions → produces a detailed SPEC.md.
3. **Architecture** — AI agent designs the Azure infrastructure.
4. **Provision** — Azure resources are created via Bicep (or skipped for local-only).
5. **Build** — Claude Code agent reads SPEC.md + CLAUDE.md, writes the entire app, runs tests, enforces completion via hooks.
6. **Complete** — Working app in `workspace/<app-name>/` with Docker Compose, tests, deployment config.

## The Agent Execution Protocol (How Claude Code Builds)

The build agent follows a strict protocol enforced by hooks:
- **EXPLORE** — parallel subagents read the codebase
- **PLAN** — writes all phase files (`.claude/phases/*.md`) before writing code
- **EXECUTE → VALIDATE → VERIFY** loop per phase
- **Integration Testing** — Docker rebuild, curl all endpoints, write Playwright E2E tests
- **Completeness Audit** — cross-references every SPEC.md requirement against actual code
- **Stop hooks** — the agent literally cannot stop until all checkboxes are checked and an audit phase exists

## The Stop Hook System

This is key: the build agent is prevented from stopping early by `validate-stop.py`.
- Gate 1: Any `- [ ]` unchecked in any phase file → agent is blocked from stopping
- Gate 2: No completeness audit phase → agent is blocked from stopping
- This means the agent self-drives to completion. It doesn't need babysitting.

## Context Survival (Compaction)

When Claude Code's context fills up (long builds), compaction is triggered:
- `pre-compact.py` snapshots phase state to `compact-checkpoint.json`
- After compaction, `post-compact-context.py` injects the checkpoint back
- The agent re-reads phase files and picks up exactly where it left off
- **No work is lost** even across context resets

## Generated App Stack

Every app ADOM-LOOP builds uses:
- **Backend:** FastAPI + Beanie ODM + MongoDB (port auto-assigned 10000-60000)
- **Frontend:** React 19 + Vite + TypeScript + Tailwind + shadcn/ui
- **Database:** MongoDB 7 (Docker) or Azure Cosmos DB
- **Testing:** Pytest (backend), Vitest (frontend), Playwright (E2E)
- **DevOps:** Docker Compose, Makefiles, GitHub Actions, Helm charts

## Key Ports

- Dashboard UI: `http://localhost:3000`
- Dashboard Backend API: `http://localhost:3001`
- Generated app ports: Auto-assigned (10000-60000 range), check `workspace/<app>/.env`

## Agentic-First: What It Means for Specs

The interview agent builds SPEC.md. The spec drives everything. To get agent-friendly apps:
- Request explicit REST API endpoints for every action (even config/admin actions)
- Ask for API key auth on admin endpoints
- Ask for management endpoints (e.g., `PATCH /api/config/`) not just CRUD
- Avoid specs that say "user goes to settings page" — say "API endpoint updates setting X"
- Ask for a `/api/admin/` namespace for operational tasks

---

*Add learnings here as they are discovered during real sessions.*

---

## Enterprise Spec Workflow (Discovered 2026-03-11)

### The Right Way to Use a Large Master Spec with ADOM-LOOP

ADOM-LOOP's interview agent is **not designed for pre-written specs**. The interview accumulates messages and blows the 200K token limit with a large spec. The correct flow for an enterprise spec like the 508 module:

**Step 1 — Create the app via API, write spec directly to disk:**
```bash
# Create app (skip spec field — too large for the API call)
curl -X POST http://localhost:3001/api/apps/create \
  -H "Content-Type: application/json" -d '{"name":"My App Name"}'

# Write master spec directly to disk
cp /path/to/MASTER-SPEC.md workspace/<slug>/SPEC.md
```

**Step 2 — Drive architect manually via single curl (not UI — UI resets session):**
```bash
# Skip interview
curl -X POST http://localhost:3001/api/agents/interview/approve \
  -H "Content-Type: application/json" -d '{"project":"<slug>"}'

# Start architect in one long curl session
curl -s -N --max-time 300 -X POST http://localhost:3001/api/agents/architect/start \
  -H "Content-Type: application/json" -d '{"project":"<slug>"}'

# If catalog search returns empty, send resource list explicitly:
curl -X POST http://localhost:3001/api/agents/architect/message \
  -H "Content-Type: application/json" \
  -d '{"project":"<slug>","message":"Call save_architecture with: azure-kubernetes-service, azure-cosmos-db, storage-accounts, azure-service-bus, key-vault, application-insights, virtual-networks..."}'

# Approve
curl -X POST http://localhost:3001/api/agents/architect/approve \
  -H "Content-Type: application/json" -d '{"project":"<slug>"}'
```

**Step 3 — Skip provisioning, start build:**
```bash
curl -X POST http://localhost:3001/api/projects/<slug>/provision/approve \
  -H "Content-Type: application/json" -d '{}'

curl -X POST http://localhost:3001/api/run \
  -H "Content-Type: application/json" -d '{"project":"<slug>","step_index":0}'
```

**Step 4 — If UI stuck on wrong phase, fix localStorage:**
```javascript
const d = JSON.parse(localStorage.getItem('adom_session'));
Object.assign(d, {project:'<slug>', wizardPhase:'build'});
localStorage.setItem('adom_session', JSON.stringify(d));
location.reload();
```

### Why Features Get Missed in Generated Apps

The build agent reads SPEC.md using chunked `Read` tool calls — it CAN handle large specs. The issue is not spec size at the build stage. The issue is **what is in SPEC.md when the build starts.**

**The real failure mode (confirmed 2026-03-11):**
When a large master spec is fed to the interview agent, the agent produces a good structured spec — but it has to split its output across multiple messages (e.g. 4 parts) because the response is too long. When Landon clicked "next" to advance to the architecture step, ADOM-LOOP only wrote Part 1 of 4 to SPEC.md. Parts 2-4 were silently dropped.

The architect and build agent then worked from a partial spec. Features described only in Parts 2-4 (PDF preview, etc.) were never built.

**The interview-generated spec is the right format.** It is NOT better to inject the raw master spec directly. The interview agent structures the spec the way ADOM-LOOP's build pipeline expects. The fix is making sure all parts make it to disk.

**Critical check before clicking "next" from the interview step:**
```bash
# Check SPEC.md was fully written — for a full enterprise spec expect 80-200KB
wc -c workspace/<slug>/SPEC.md

# Check it doesn't end mid-spec (look for a proper conclusion section)
tail -20 workspace/<slug>/SPEC.md

# Count major sections — a complete spec should have many
grep -c "^## " workspace/<slug>/SPEC.md
```

If SPEC.md is truncated (only Part 1), manually copy the remaining parts from the interview chat into the file before advancing.

---

## PDF.js (pdfjs-dist v5) + Vite + React StrictMode: Full Fix Pattern (2026-03-14)

ADOM generates PDF viewers using `pdfjs-dist`. Three separate bugs all produced the same symptom: blank white canvas. ADOM took 2 failed attempts before identifying all three. See GOTCHAS.md for the full breakdown. Summary:

1. `blob_path` from the API is already the full URL -- never prepend `/api/v1/blobs/` to it
2. React StrictMode fires render effects twice concurrently -- always use a `cancelled` flag + cleanup to prevent the second invocation from clearing the canvas while the first is still painting
3. Vite pre-bundles pdfjs-dist and breaks its ESM structure -- add `optimizeDeps: { exclude: ['pdfjs-dist'] }` in vite.config.ts
4. Static `import * as pdfjsLib from 'pdfjs-dist'` crashes jsdom tests (DOMMatrix) -- use dynamic `import('pdfjs-dist')` inside the effect, with only `import workerUrl from '...?url'` at module level

**ADOM limitation identified:** ADOM's default pdfjs scaffolding has all three bugs. Until the ADOM-LOOP scaffold is updated, any generated app with a PDF viewer will need these patches applied manually. Worth raising with the ADOM-LOOP lead dev as a scaffold-level fix.

*Add learnings here as they are discovered.*

---

## Architect Catalog — Manually Added Missing Azure Services (2026-03-12)

Three Azure services required by the 508 module were missing from the architect's catalog and have been added **locally only** to:
`~/clawd/repos/ADOM-LOOP/dashboard/backend/agents/catalogs/azure_resources.json`

Services added:
- `azure-openai-service` — LLM enrichment, AI fix suggestions
- `azure-ai-document-intelligence` — PDF OCR and structure extraction
- `azure-container-registry` — required for AKS image pulls

**This is a local-only change. Not committed to the ADOM-LOOP repo.**

Action needed for lead dev: These three resources should be added to the upstream catalog in the ADOM-LOOP repo so they are available to all users building AI/document-processing apps. The full JSON entries are in the local catalog file above and can be copied directly. They follow the existing catalog schema exactly.

