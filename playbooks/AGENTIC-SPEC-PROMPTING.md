# AGENTIC-SPEC-PROMPTING.md — How to Prompt for Agent-Friendly Apps

*The ADOM-LOOP interview agent builds SPEC.md. The spec drives everything the build agent creates. This playbook shows how to steer the interview so the generated app is operable by an agent, not just a human clicking through a UI.*

---

## The Core Principle

Every feature should have a corresponding API endpoint. If a human can do it in the UI, an agent should be able to do it via a curl command. When answering interview questions, think in terms of operations, not screens.

---

## During the Interview: What to Push For

### 1. Explicit REST Endpoints for Everything
When the interview agent asks about features, frame your answers as API actions:

**Instead of:** "Users can go to settings and change the notification frequency."  
**Say:** "There should be a `PATCH /api/settings/notifications` endpoint that accepts `{frequency: 'daily' | 'weekly' | 'never'}`. The UI should call this."

**Instead of:** "Admin can approve pending items."  
**Say:** "There should be a `POST /api/items/{id}/approve` endpoint. Admins can trigger this from the UI, and it should also be callable with an API key for automated workflows."

### 2. API Key Auth for Administrative/Operational Endpoints
Push for a simple API key mechanism on any endpoint that an agent might call:

"Can we include a lightweight API key auth option for admin/operational endpoints? The key would live in the `.env` file. This allows automated tools to call management endpoints without going through the user session."

### 3. Admin/Management Namespace
Ask for a `/api/admin/` namespace:

"Can we include an `/api/admin/` namespace for operational tasks? Things like resetting state, triggering batch jobs, updating configuration, checking system health. This makes it easy to wire up automation later."

### 4. Health and Status Endpoints
Always ask for:
- `GET /health` — simple up/down check
- `GET /api/status` — richer system state (queue depth, last run, etc.)

### 5. Config as Data, Not Hard-Coded
"Rather than hard-coding configuration values, can they be stored in a config collection in MongoDB and exposed via `GET /api/config/` and `PATCH /api/config/`? This lets an agent update settings without redeploying."

### 6. Webhooks or Event Hooks (if relevant)
"Can the app support outbound webhooks when key events happen? For example, when a record is created or a status changes, POST to a configurable webhook URL. This allows external agents to react to state changes."

### 7. Structured Logging
"Can the backend use structured JSON logging? This makes it easier for agents to parse logs and detect errors without screen-scraping."

---

## Interview Answer Templates

**For any feature that has a UI action:**
> "The [feature] should be exposed as `[METHOD] /api/[resource]/[action]`. The frontend calls this endpoint. Document the request/response shape in the spec."

**For any admin/operational task:**
> "Add `[METHOD] /api/admin/[task]` that requires an `X-API-Key` header. The key is set in `.env` as `ADMIN_API_KEY`."

**For any configurable setting:**
> "Store this in a `config` collection. Expose `GET /api/config/` to read current values and `PATCH /api/config/` to update them. No restart needed."

---

## Example: Task Tracker with Agentic Framing

**Bad spec framing:**
"Users can create tasks, mark them complete, and filter by status on the dashboard."

**Agent-friendly spec framing:**
"The app manages tasks with the following API surface:
- `POST /api/tasks/` — create a task (title, description, priority, status)
- `GET /api/tasks/` — list tasks with optional `?status=` filter and pagination
- `GET /api/tasks/{id}` — get a single task
- `PATCH /api/tasks/{id}` — update any field including status
- `DELETE /api/tasks/{id}` — delete a task
- `POST /api/tasks/{id}/complete` — shorthand to mark complete
- `GET /api/admin/stats` — task counts by status (requires X-API-Key)
- `PATCH /api/config/` — update app settings like default priority

The UI wraps all of these. An agent can also call any endpoint directly using the `ADMIN_API_KEY` from `.env`."

---

## After the App is Built

Once the build agent finishes, review the generated `workspace/<app>/backend/src/routes/` directory. Make sure every intended operation has a corresponding route.

If something is missing:
1. Note it in `context/APPS.md` under that app's entry
2. You can ask Loop to plan a spec amendment and re-run the build agent on a patch prompt
