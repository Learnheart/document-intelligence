# Agent Registry

> Each agent is a standalone App under `src/agents/`.
> How orchestrator connect with sub agent
> Each agent owns its own `AGENT.md` with full details (routes, env vars, architecture, deps).

---

## 🛑 IMPORTANT RULES

> **These rules are non-negotiable. AI agents and human contributors MUST follow them. Do NOT skip any step.**

### 1. Documentation-First: Plans BEFORE Code

All plans, design documents, and implementation specs **MUST** be stored in the `docs/` folder **BEFORE any implementation begins**. No exceptions.

```
docs/
└── YYYY-MM-DD/
    └── <feature-name>/
        ├── plan.md           # Implementation plan (REQUIRED — write FIRST)
        ├── design.md         # Technical design (if needed)
        └── notes.md          # Research notes, decisions (if needed)
```

Every doc **MUST** start with this metadata block:

```markdown
---
author: <name or email>
date: YYYY-MM-DD
status: draft | in-progress | done | abandoned
agents: <comma-separated agent IDs affected>
summary: <one-line description of what this is about>
---
```

- **ALWAYS** create `docs/YYYY-MM-DD/<feature-name>/plan.md` BEFORE writing any code
- Use the date the work started (ISO format: `YYYY-MM-DD`)
- Use kebab-case for feature names
- Every feature, enhancement, or bugfix that involves a plan MUST have a corresponding `docs/` entry
- The plan must include: metadata block, problem statement, requirements, decisions made, and implementation approach
- If you are an AI agent: **stop and create the plan document first**, then proceed with implementation

### 2. Shared Code Coordination

- Changes to `src/shared/` affect **ALL** agents — coordinate with other teams before modifying
- When adding a new shared module, update the Shared Module Reference table in this file
- When modifying a shared module's interface, check all agents that depend on it

### 3. Agent Details Live in Each Agent's `AGENT.md`

- Each agent's full details (API routes, env vars, architecture, deps) live in `src/agents/<name>/AGENT.md`
- When changing an agent, update **its own** `AGENT.md` — that is the source of truth
- Do NOT duplicate agent details in this file — keep only the index table below

### 4. Deployment Safety

- Always run `./deploy.sh --env dev --dry-run <service>` before actual deployment
- Deploy to `dev` environment first, verify, then deploy to `prod`
- When modifying `app.yaml.tpl`, verify the generated `app.yaml` has correct values via dry run

### 5. Changelog — Track Every Change

- **ALWAYS** update `CHANGELOG.md` when making any code change
- Follow [Semantic Versioning](https://semver.org/): `MAJOR.MINOR.PATCH`
  - **MAJOR** — breaking changes (API contract changes, data model changes, removed features)
  - **MINOR** — new features, new agents, new shared components
  - **PATCH** — bug fixes, prompt tuning, styling tweaks, doc updates
- Group entries under: `Added`, `Changed`, `Fixed`, `Removed`
- Tag each entry with the affected agent(s) in brackets, e.g. `[summarizer]`, `[shared]`, `[all]`
- If you are an AI agent: **update CHANGELOG.md as part of every commit**
- **Enforced by Kiro hook**: `.kiro/hooks/check-changelog.sh` blocks `git commit` if `CHANGELOG.md` is not staged

---

## Agent Index

For full details on any agent, see its `AGENT.md` file.

| #  | Name                  | ID                    | Databricks App              | Path                                  | Frontend |
|----|-----------------------|-----------------------|-----------------------------|---------------------------------------|----------|
| 1  | TÀI Studio            | `super-agent`         | `super-agent`               | `src/super-agent/`                    | Yes      |
| 2  | The Translator        | `the-translator`      | `agent-the-translator`      | `src/agents/the-translator/`          | Yes      |
| 3  | The Summarizer        | `the-summarizer`      | `agent-the-summarizer`      | `src/agents/the-summarizer/`          | Yes      |
| 4  | The Powerpoint-er     | `the-powerpoint-er`   | `agent-the-powerpoint-er`   | `src/agents/the-powerpoint-er/`       | Yes      |
| 5  | The Vaulter           | `the-vaulter`         | `agent-the-vaulter`         | `src/agents/the-vaulter/`             | Yes      |
| 6  | The Brainstormer      | `the-brainstormer`    | `agent-the-brainstormer`    | `src/agents/the-brainstormer/`        | Yes      |
| 7  | TÀI                   | `tai`                 | `agent-tai`                 | `src/agents/tai/`                     | No       |
| 8  | The AI Visionary      | `the-ai-visionary`    | `agent-the-ai-visionary`    | `src/agents/the-ai-visionary/`        | Yes      |
| 9  | The Graphics Designer | `the-canvas-designer` | `agent-the-canvas-designer` | `src/agents/the-canvas-designer/`     | Yes      |
| 10 | The Resume Evaluator  | `the-resume-evaluator`| `agent-the-resume-evaluator`| `src/agents/the-resume-evaluator/`    | Yes      |

---

## Shared Module Reference

Modules in `src/shared/backend/services/`:

| Module | Purpose |
|--------|---------|
| `lakebase.py` | Shared Lakebase connection pool with OAuth token rotation + `lakebase_checkpointer()` for LangGraph |
| `llm.py` | ChatDatabricks client (Sonnet / Haiku) with lazy MLflow init |
| `compact_messages.py` | Message compaction (summarize-then-trim) to prevent context overflow |
| `file_store.py` | Volume-based file storage (save / load) |
| `file_parser.py` | Document parsing via `ai_parse_document` |
| `file_validator.py` | Upload validation (size, type, magic bytes) |
| `chunker.py` | Semantic document chunking with contextual enrichment for RAG |
| `user_context.py` | Extract user email from request headers |
| `schemas.py` | Shared Pydantic response models |
| `user_preferences.py` | Lakebase-backed cross-session user preferences (language, tone, topics) |
| `logging_config.py` | Shared logging setup — standard format, naming convention, noisy logger suppression |

Components in `src/shared/frontend/components/`:

| Component | Purpose |
|-----------|---------|
| `Markdown.tsx` | `react-markdown`-based renderer for LLM responses with brand-consistent styling (used by all agents with text output) |
| `AgentIcons.tsx` | SVG icon components for each agent |
| `FileDropzone.tsx` | Drag-and-drop file upload component |
| `GovernanceBanner.tsx` | Two inline governance notices: `AdvisoryNotice` (above chat inputs) and `PiiNotice` (below file upload components) |

---

## Lakebase Connection

Agents that use Lakebase (TÀI, Brainstormer, Vaulter, TÀI Studio) share a single connection module: `lakebase.py`.

**How it works** (follows the [official Databricks guide](https://docs.databricks.com/aws/en/oltp/projects/tutorial-databricks-apps-autoscaling)):
- `OAuthConnection` generates a fresh token per new pool connection
- `ConnectionPool` reuses connections — most requests don't trigger token generation
- `DATABRICKS_CLIENT_ID` (auto-set by Databricks Apps runtime) is used as the Postgres username

**Config lives in 3 files:**

| File | Role | Example |
|------|------|---------|
| `environments/dev.env` | Defines values per environment | `LAKEBASE_PGHOST="ep-xxx.database.region.cloud.databricks.com"` |
| `src/agents/<name>/app.yaml.tpl` | Template with `${VAR}` placeholders | `value: "${LAKEBASE_PGHOST}"` |
| `/tmp/deploy-<name>/app.yaml` | Generated at deploy time (never committed) | `value: "ep-xxx.database.region.cloud.databricks.com"` |

**Required env vars in `app.yaml.tpl` for Lakebase agents:**

```yaml
env:
  - name: PGHOST
    value: "${LAKEBASE_PGHOST}"
  - name: PGDATABASE
    value: "databricks_postgres"
  - name: PGPORT
    value: "5432"
  - name: PGSSLMODE
    value: "require"
  - name: ENDPOINT_NAME
    value: "${LAKEBASE_ENDPOINT_NAME}"
  - name: LAKEBASE_AUTOSCALING_PROJECT
    value: "${LAKEBASE_PROJECT}"
  - name: LAKEBASE_AUTOSCALING_BRANCH
    value: "${LAKEBASE_BRANCH}"
resources:
  - name: lakebase
    postgres:
      project: ${LAKEBASE_PROJECT}
      branch: projects/${LAKEBASE_PROJECT}/branches/${LAKEBASE_BRANCH}
      database: ${LAKEBASE_DATABASE}
      permission: CAN_CONNECT_AND_CREATE
```

**Required variables in `environments/<env>.env`:**

```bash
LAKEBASE_PROJECT="super-agent"
LAKEBASE_BRANCH="production"
LAKEBASE_PGHOST="<from Lakebase Connect dialog>"
LAKEBASE_ENDPOINT_NAME="<from branch → Computes → Get ID → Copy resource name>"
```

**`PGUSER` is NOT set in `app.yaml`** — each app has a different service principal. `DATABRICKS_CLIENT_ID` is auto-injected by the Databricks Apps runtime and `lakebase.py` reads it as the Postgres username. For local testing, set `PGUSER=your.email@company.com`.

**⚠ New environment setup:** Before the first deploy, run the Lakebase SQL grants for each service principal. See [Deployment Runbook — Step 6](docs/deployment-runbook.md#6-lakebase-project--branch).

---

## MLflow Tracing

All agents use MLflow for LLM tracing via `mlflow.langchain.autolog()`.

**Setup pattern** (in `app.yaml`):

```yaml
env:
  - name: MLFLOW_TRACKING_URI
    value: "databricks"
  - name: MLFLOW_EXPERIMENT_ID
    value: "<experiment_id>"
resources:
  - name: mlflow-experiment
    experiment:
      experiment_id: "<experiment_id>"
      permission: CAN_MANAGE
```

**Key details:**
- MLflow init is **lazy** — runs on first `get_llm()` call, not at import time. This avoids startup crashes.
- `MLFLOW_TRACKING_URI=databricks` must NOT be set at import time (crashes the app). The shared `llm.py` handles this.
- The `experiment` resource grants the app's service principal `CAN_MANAGE` on the experiment.
- Traces include: input/output messages, token counts, latency, model endpoint.
- Known issue: S3 artifact uploads may fail with `ConnectionResetError` — traces still appear in Databricks, just without full artifact payload.

---

## Adding a New Agent

1. Create `src/agents/<name>/` with `backend/`, `frontend/` (optional), and `app.yaml.tpl`
2. Create `src/agents/<name>/AGENT.md` with full agent details (use any existing agent as template)
3. Backend pattern: standalone FastAPI + router + CORS + file upload (if needed) + SPA serving + health
4. Frontend pattern: Vite project, render page directly (no AgentLayout), copy shared components from `src/shared/frontend/`
5. Add governance notices: `AdvisoryNotice` above chat/input areas, `PiiNotice` below file upload components (import from `GovernanceBanner.tsx`)
6. Create MLflow experiment: `databricks workspace mkdirs "/super-agent" && databricks experiments create-experiment "/super-agent/<name>"`
7. Add `MLFLOW_TRACKING_URI`, `MLFLOW_EXPERIMENT_ID` env vars and `mlflow-experiment` resource to `app.yaml.tpl`
8. Add a row to the Agent Index table in this file
9. Add `<NAME>_URL` env var to `src/super-agent/app.yaml.tpl`
10. Add agent card to super-agent's `App.tsx` (`AGENTS` array)
11. Deploy: `./deploy.sh --env dev <name>` then `./deploy.sh --env dev super-agent`
12. Update `CHANGELOG.md`