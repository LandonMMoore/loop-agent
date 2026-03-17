# GOTCHAS.md — Known Issues & Workarounds

*From the README, architecture docs, and real sessions.*

---

## Token Issues

### "Invalid or missing agent token"
**Cause:** `CLAUDE_CODE_OAUTH_TOKEN` is missing or expired.  
**Fix:**
```bash
claude setup-token
```
Copy the new `sk-ant-oat01-...` token and update `dashboard/backend/.env`.

### OAuth token vs API key — don't confuse them
- `CLAUDE_CODE_OAUTH_TOKEN` (starts `sk-ant-oat01-...`) — used by the build agent subprocess
- `ANTHROPIC_API_KEY` (starts `sk-ant-api03-...`) — used by interview and architecture agents
- Both are required. Both go in `.env`. They are different.

---

## Docker Issues

### Docker containers won't start / template placeholders
**Cause:** Build agent overwrote `docker-compose.dev.yml` with unresolved `{{PORT}}` placeholders.  
**Fix:** Dashboard self-heals on next start. Or manually:
```bash
grep -r '{{PORT}}' workspace/<app>/docker-compose.dev.yml
# Replace any {{PORT}} with the actual port from workspace/<app>/.env
```

### Port conflicts
**Cause:** Two generated apps assigned same port (rare), or another service on that port.  
**Fix:** Each app gets unique ports based on app name hash. Stop conflicting app:
```bash
docker compose -f workspace/<other-app>/docker-compose.dev.yml down
```

### Docker Desktop not running
**Fix:** Open Docker Desktop app. Wait for whale icon to be stable. Then retry.

---

## Windows-Specific

### Claude subprocess fails on Windows
**Fix:** Run from Git Bash, not PowerShell. Always.

---

## Build Agent Issues

### Agent stops early (all phases not complete)
**This shouldn't happen** — stop hooks prevent it. If it does:
- Check `.claude/phases/` for any `- [ ]` unchecked items
- Manually restart: re-run `python dashboard/start.py` and resume the build

### Build loops forever
**Cause:** Stop hook is blocking because of unchecked boxes that the agent can't satisfy.  
**Fix:** Review phase files. If a phase item is impossible (e.g., references a missing dependency), it may need manual intervention. Check logs.

---

## Setup Issues

### Python venv activation fails
On macOS/Linux use: `source .venv/bin/activate`  
On Windows (Git Bash): `source .venv/Scripts/activate`

### `pip install -e azure-provisioner/` fails
**Cause:** Missing build tools or Python version mismatch.  
**Fix:** Ensure Python 3.12+. Run: `pip install --upgrade pip setuptools wheel` first.

---

---

## Spec Size Issues (Discovered 2026-03-11)

### Issue 1: Interview Agent Produces Multi-Part Spec — Only Part 1 Writes to Disk
**What actually happened:**
The interview agent successfully produced a complete spec, but because the master spec was so large and detailed, the spec it generated had to be split across 4 parts (the AI's response was too long to emit in one message). When Landon clicked "next" to advance to the architecture step, ADOM-LOOP only captured Part 1 of 4 and wrote it to SPEC.md. Parts 2-4 were lost.

**Why it matters:** The architect and build agent then worked from an incomplete spec — Part 1 of 4 only. Major features described in Parts 2-4 (including PDF preview and other key capabilities) were never in SPEC.md and therefore never built.

**Root cause in ADOM-LOOP:** The `<!-- SPEC_READY -->` marker detection or the "next step" transition only captures whatever is in the last response, not all accumulated parts of a multi-message spec.

**Fix / Workaround:**
- Before clicking "next" from the interview step, verify SPEC.md on disk is complete:
  ```bash
  wc -c workspace/<slug>/SPEC.md
  cat workspace/<slug>/SPEC.md | grep -c "^##"  # count sections
  ```
- If only Part 1 is there, manually append the remaining parts from the interview chat before advancing
- Better: for large specs, bypass the interview entirely and inject the master spec directly (see workflow in LEARNINGS.md)

### Issue 2: Interview-Generated Spec Is Still Preferable to Raw Master Spec
**Context:** Even though the interview agent missed some things initially, its generated spec is structurally correct for ADOM-LOOP's build agent. The master spec (432KB) is a reference document — dense, formal, and not structured the way ADOM-LOOP's build pipeline expects.

**The real issue was not that the architect overwrote an injected spec** — the architect was doing its job correctly. The issue was that it only had Part 1 of the interview-generated spec to work from.

**Ideal flow for enterprise specs:**
1. Run the interview with the master spec as input
2. Let the interview agent produce its own structured spec (it will likely be multi-part for large inputs)
3. **Manually verify all parts are captured in SPEC.md before clicking "next"**
4. The interview-generated spec + any corrections from Landon = the canonical SPEC.md for the build

**What to check before advancing from interview:**
```bash
# The interview-generated spec should be substantial (not 77KB AI stub, not 432KB raw master)
# Expect something in the 80-200KB range for a complete enterprise spec
wc -c workspace/<slug>/SPEC.md
head -5 workspace/<slug>/SPEC.md  # should start with app title, not "# Part 1 of 4"
```

---

### Architect Resets In-Memory Session on Every Reconnect
**Cause:** `POST /api/agents/architect/start` calls `session["messages"] = []` — it resets the conversation every time it's called. The UI calls this endpoint every time it reconnects to the SSE stream, wiping in-progress work.

**Symptom:** Architect starts, UI reconnects, architect resets, infinite loop of partial runs.

**Fix:** Drive the architect via a single uninterrupted curl session, not through the UI:
```bash
curl -s -N --max-time 300 -X POST http://localhost:3001/api/agents/architect/start \
  -H "Content-Type: application/json" \
  -d '{"project":"<slug>"}'
```
Then immediately approve when done:
```bash
curl -X POST http://localhost:3001/api/agents/architect/approve \
  -H "Content-Type: application/json" \
  -d '{"project":"<slug>"}'
```

---

### Architect Catalog Search Returns Empty — Wrong Keywords
**Cause:** The architect uses `search_catalog` to find Azure resources. It searches for human-readable names like "AKS", "OpenAI", "Kubernetes" — but catalog IDs are `azure-kubernetes-service`, `azure-cosmos-db`, etc.

**Symptom:** `search_catalog` returns `{'count': 0, 'resources': []}` for every search, architect only adds 1-2 resources before giving up.

**Fix:** Don't let the architect search — tell it exactly what to deploy by sending a direct message via `/api/agents/architect/message` with the explicit resource ID list from the catalog. Available catalog IDs:
- `azure-kubernetes-service`
- `azure-cosmos-db`
- `storage-accounts`
- `azure-service-bus`
- `key-vault`
- `application-insights`
- `virtual-networks`
- `azure-front-door`
- `azure-container-apps`
- `azure-signalr`
- `azure-app-service`
- `azure-functions`
- `azure-redis-cache`
- `azure-app-configuration`

**NOT in catalog (requires manual Bicep or env-var config):**
- Azure OpenAI Service
- Azure AI Document Intelligence
- Azure Container Registry

---

### Azure Provisioner 15-Minute Timeout Kills Long Deployments
**Cause:** ADOM-LOOP's provisioner has a hardcoded 15-minute timeout. AKS alone takes 10-15 minutes. With 16 resources, total deployment takes 30-45 minutes.

**Symptom:** Deployment starts, some resources succeed, then ADOM-LOOP marks it as failed and triggers auto-teardown (deleting everything including successfully provisioned resources).

**Fix — bypass ADOM-LOOP's provisioner entirely for enterprise specs:**
```bash
# Save the Bicep from the temp dir before ADOM-LOOP cleans up:
cp /var/folders/.../azprov-*/main.bicep ~/saved-main.bicep

# Run directly with no timeout:
az deployment group create \
  --resource-group <rg-name> \
  --template-file ~/saved-main.bicep \
  --mode Incremental \
  --parameters location=eastus2 appNameSlug=<slug>
```

---

### Generated Bicep Missing Enterprise Services
**Cause:** The provisioner generates Bicep from its catalog. The catalog doesn't include AKS, Azure OpenAI, or Azure AI Document Intelligence. Enterprise specs that require these get incomplete infrastructure.

**Symptom:** Generated `main.bicep` has Cosmos DB, Redis, Storage, Functions — but no AKS or AI services.

**Fix:** Skip provisioning in ADOM-LOOP entirely (`POST /api/agents/architect/approve` to advance). Provision the full enterprise stack separately using the `508-INFRA-AZURE.md` spec from the abs-specs repo.

---

### UI Gets Stuck on Provision Step After Failure
**Cause:** `wizardPhase` is persisted in `localStorage` under key `adom_session`. When provisioning fails, the phase stays `"provision"` and project can go null.

**Fix — update localStorage directly via browser console or devtools:**
```javascript
const data = JSON.parse(localStorage.getItem('adom_session'));
data.wizardPhase = 'build';
data.project = 'app-<slug>';
localStorage.setItem('adom_session', JSON.stringify(data));
location.reload();
```

---

### macOS: `start.py` Exits Immediately
**Cause:** On macOS, the frontend/backend startup script exits without keeping processes alive.

**Fix — run backend and frontend manually:**
```bash
# Backend
cd ~/clawd/repos/ADOM-LOOP/dashboard/backend
~/clawd/repos/ADOM-LOOP/.venv/bin/python -m uvicorn main:app --host 0.0.0.0 --port 3001 &

# Frontend
cd ~/clawd/repos/ADOM-LOOP/dashboard/frontend
npx vite --port 3000 &
```

---

### pdfjs-dist v4/v5 + Vite + React StrictMode: Blank/White Canvas (3 Bugs)

**Date discovered:** 2026-03-14

**Symptom:** PDF viewer is scrollable and sized correctly (canvas has real dimensions) but renders a completely blank white page. The backend blob endpoint returns 200 and the PDF is valid.

**Root cause — THREE bugs, all present:**

**Bug 1 — Double-prefixed blob URL (404):**
- `blob_path` from the API is already a full relative URL: `/api/v1/blobs/org_id/doc_id/filename`
- Frontend was building: `` `/api/v1/blobs/${doc.blob_path}` `` → garbage double-prefixed URL → 404
- Fix: use `doc.blob_path` directly: `const blobUrl = doc.blob_path ?? null`

**Bug 2 — React StrictMode double-invoke races the canvas clear:**
- `useEffect(() => { renderPage(); }, [renderPage])` had no cleanup function
- React StrictMode fires effects twice in dev. Both async `renderPage()` calls ran concurrently
- The second call reset `canvas.width` (which clears the canvas) while the first render was still writing pixels → blank white result
- Fix: add a `cancelled` flag + cleanup that sets it. Second invocation sees `cancelled = true` and exits immediately

**Bug 3 — Vite pre-bundles pdfjs-dist v5, breaking its internal ESM structure:**
- Vite's esbuild pre-bundler transforms pdfjs-dist at startup, which can break its pure-ESM module structure
- Fix: add `optimizeDeps: { exclude: ['pdfjs-dist'] }` in `vite.config.ts`

**Bonus — static `import * as pdfjsLib` crashes jsdom tests:**
- `import * as pdfjsLib from 'pdfjs-dist'` at module level runs pdfjs initialization, which calls `new DOMMatrix()` — not available in jsdom
- Fix: keep only the `?url` import at module level (safe, just a string). Use `import('pdfjs-dist')` inside the effect with a `cancelled` guard
- The `?url` suffix import IS safe at module level because it's just a URL string, no pdfjs code runs

**Final correct pattern for pdfjs-dist v5 + Vite + React StrictMode:**
```ts
// Top of file — ?url import is just a string, safe at module level
import pdfjsWorkerUrl from 'pdfjs-dist/build/pdf.worker.min.mjs?url';

// Inside useEffect:
useEffect(() => {
  if (!blobUrl) return;
  let cancelled = false;
  import('pdfjs-dist').then(async (pdfjsLib) => {
    pdfjsLib.GlobalWorkerOptions.workerSrc = pdfjsWorkerUrl;
    if (cancelled) return;
    const pdf = await pdfjsLib.getDocument(blobUrl).promise;
    if (cancelled) return;
    setPdfDoc(pdf);
  });
  return () => { cancelled = true; };
}, [blobUrl]);

// In renderPage useEffect:
useEffect(() => {
  let cancelled = false;
  renderPage(cancelled);               // pass flag into async fn
  return () => { cancelled = true; renderTaskRef.current?.cancel(); };
}, [renderPage]);
```

**vite.config.ts:**
```ts
optimizeDeps: {
  exclude: ['pdfjs-dist'],
}
```

**Applies to:** Any ADOM-generated app using pdfjs-dist v4+ with Vite + React StrictMode.

*Add new gotchas here as they are discovered.*
