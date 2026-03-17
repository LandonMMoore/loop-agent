# SETUP.md — ADOM-LOOP Local Setup Playbook

*Step-by-step. Landon does the physical actions. Loop guides each one.*

---

## Phase 1: Verify Prerequisites

Run each of these in your terminal. Tell Loop what you get back.

```bash
python3 --version   # Need: 3.12 or higher
node --version      # Need: v18 or higher
docker info         # Need: Docker Desktop running (no error)
claude --version    # Need: Claude Code CLI installed
az --version        # Need: Azure CLI (only needed for Azure provisioning)
```

**If anything is missing:**
- Python 3.12: https://www.python.org/downloads/
- Node 18+: https://nodejs.org/
- Docker Desktop: https://www.docker.com/products/docker-desktop/
- Claude Code: `npm install -g @anthropic-ai/claude-code`
- Azure CLI: https://learn.microsoft.com/en-us/cli/azure/install-azure-cli

---

## Phase 2: Clone the Repo

```bash
cd ~/path/to/repos   # or wherever you want it
git clone https://github.com/AutoBridgeSystems/ADOM-LOOP.git
cd ADOM-LOOP
```

Update `context/SETUP-STATE.md` with the cloned path.

---

## Phase 3: Python Environment

```bash
python3 -m venv .venv
source .venv/bin/activate   # macOS/Linux

pip install -r dashboard/backend/requirements.txt
pip install -e azure-provisioner/
```

You should see packages installing. No red errors = good.

---

## Phase 4: Frontend Dependencies

```bash
npm install --prefix dashboard/frontend
```

---

## Phase 5: Get Your Tokens

**Claude Code OAuth Token** (for the build agent):
```bash
claude setup-token
```
This opens a browser. Authenticate with Anthropic. Copy the token that starts with `sk-ant-oat01-...`.

**Anthropic API Key** (for interview + architecture agents):
Go to: https://console.anthropic.com/ → API Keys → Create key.
Starts with `sk-ant-api03-...`.

**Azure Subscription ID** (only needed for Azure provisioning):
```bash
az login
az account show --query id -o tsv
```

---

## Phase 6: Create .env File

Create the file at `dashboard/backend/.env`:

```env
CLAUDE_CODE_OAUTH_TOKEN=sk-ant-oat01-your-token-here
ANTHROPIC_API_KEY=sk-ant-api03-your-key-here
AZURE_PROVISIONER_AZURE_SUBSCRIPTION_ID=your-subscription-uuid
```

**Do not commit this file.** It's in `.gitignore`.

Update `context/SETUP-STATE.md` to mark tokens as configured (not the values — just the status).

---

## Phase 7: Run It

```bash
# Make sure venv is activated
source .venv/bin/activate

python dashboard/start.py
```

You should see something like:
```
Starting ADOM-LOOP dashboard...
Backend: http://localhost:3001
Frontend: http://localhost:3000
```

Open your browser to: **http://localhost:3000**

---

## Phase 8: First App Test

1. Click "New App"
2. Give it a simple name and description (start small — a task tracker or notes app)
3. Go through the interview — answer Loop's prompting guide in `playbooks/AGENTIC-SPEC-PROMPTING.md`
4. Let the architecture agent run
5. Skip Azure provisioning for now (local Docker only)
6. Watch the build agent work in the live preview
7. When complete, check `workspace/<your-app>/` for the generated code

---

## Setup Complete Checklist

- [ ] All prerequisites verified
- [ ] Repo cloned to known path
- [ ] Python venv created and packages installed
- [ ] Frontend deps installed
- [ ] .env file created with all 3 tokens/keys
- [ ] Dashboard starts cleanly
- [ ] http://localhost:3000 loads
- [ ] First test app created and running

Update `context/SETUP-STATE.md` as each item is completed.
