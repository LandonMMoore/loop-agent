# SETUP-STATE.md — ADOM-LOOP Local Installation Status

*Update this file as you complete each step.*

## Repo

- **Status:** NOT CLONED
- **Repo:** AutoBridgeSystems/ADOM-LOOP (private — request access from AutoBridge)
- **Path:** (fill in after cloning)

## Prerequisites

| Requirement | Version Needed | Status | Notes |
|-------------|---------------|--------|-------|
| Python | 3.12+ | ? | Run: `python3 --version` |
| Node.js | 18+ | ? | Run: `node --version` |
| Docker Desktop | Latest | ? | Run: `docker info` |
| Claude Code CLI | Latest | ? | Run: `claude --version` |
| Azure CLI | Latest | ? | Only needed for Azure provisioning |

## Tokens & Credentials

| Token | Status | Notes |
|-------|--------|-------|
| CLAUDE_CODE_OAUTH_TOKEN | NOT CONFIGURED | Run `claude setup-token` to get it |
| ANTHROPIC_API_KEY | NOT CONFIGURED | Get from console.anthropic.com |
| AZURE_SUBSCRIPTION_ID | NOT CONFIGURED | Run `az account show --query id -o tsv` |

## .env Location

`(ADOM-LOOP path)/dashboard/backend/.env` — NOT YET CREATED

## Dashboard Status

- **Frontend:** NOT RUNNING
- **Backend:** NOT RUNNING

## Generated Apps

None yet. Registry in `context/APPS.md`.

---

*Update this file whenever install state changes.*
