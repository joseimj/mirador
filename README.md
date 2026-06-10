# Mirador 🔭

**A Looker analytics agent for Gemini Enterprise — with native inline rendering (A2UI) and Agent2Agent (A2A) interoperability.**

Ask in plain language, get answers from your Looker instance:

```
"show me the sales dashboard"        →  rendered inline as an image
"how many orders are complete?"      →  answered with live numbers
"give me the link to look 9"         →  a signed, interactive Looker URL
```

Mirador is a single-script deployment: it provisions the cloud resources, builds the agent, deploys it to Vertex AI Agent Engine, and registers it in Gemini Enterprise. It is **idempotent** — run it again and it reuses what already exists.

---

## Architecture

```
                       ┌──────────────────────────────┐
   Gemini Enterprise ──┤                              │
   (chat UI, A2UI)     │   Vertex AI Agent Engine     │
                       │   (managed runtime)          ├──► Looker
   Other agents ───────┤                              │    (dashboards, Looks,
   (A2A protocol)      │   ADK agent + Looker SDK     │     explores, data)
                       └──────────────────────────────┘
```

Design decisions worth knowing:

1. **No middle tier.** The agent calls Looker directly through its SDK. No Cloud Run, no broker, no cache. Fewer moving parts, fewer ways to fail, cheap and fast.
2. **Images travel as artifacts, not text.** Rendered PNGs are saved as ADK artifacts and displayed natively by the runtime. Image bytes never pass through the model's text channel.
3. **Links are signed by Looker.** Embed URLs are minted server-side via `create_sso_embed_url` — the HMAC is never hand-rolled.
4. **Two front doors, one agent.** The same `LlmAgent` is exposed (a) to Gemini Enterprise via the assistant registration (the A2UI path), and (b) to any A2A-compliant client via Agent Engine's A2A endpoints. Deployments are independent and additive — adding A2A never breaks the A2UI experience.

## What the agent can do

| Capability | Tools | Output |
|---|---|---|
| Render dashboards (with filters) | `show_dashboard_inline`, `list_dashboard_filters` | Inline PNG |
| Render Looks and ad-hoc charts | `show_look_inline`, `show_query_inline` | Inline PNG |
| Answer data questions | `query_looker_data` | Text / markdown table |
| Discover the schema by itself | `list_looker_models`, `list_looker_explores`, `list_looker_fields` | Text |
| Signed interactive links | `get_dashboard_link`, `get_look_link`, `get_explore_link` | Looker SSO URL |

## Repo layout

```
.
├── gemini_looker.sh      # End-to-end provisioning + A2UI deploy + Gemini Enterprise registration
├── gemini_looker_a2a.sh  # Additive companion: deploys the SAME agent as an A2A server
└── README.md
```

Both scripts follow the same pattern: they generate their Python (`looker_app/`, `deploy.py`, `deploy_a2a.py`) at run time via heredocs, so the repo stays small and the deployed code always matches the script version. `gemini_looker_a2a.sh` does **not** regenerate the agent — `gemini_looker.sh` remains the single source of truth for `agent.py`.

---

## Quickstart

### 0. Prerequisites

- A Google Cloud project with billing enabled, and `gcloud` authenticated.
- A Looker instance with API credentials (client ID + secret).
- A Gemini Enterprise app already created (you need its app ID).
- If using Claude as the reasoning model: the model enabled in **Vertex AI Model Garden** (one-time Console action — the script preflights this and fails fast before the long deploy).

### 1. Configure

Edit the configuration block at the top of `gemini_looker.sh`:

```bash
PROJECT_ID="..."            PROJECT_NUMBER="..."
BUCKET_NAME="..."           # one bucket: deploy staging + image artifacts
LOOKER_URL="https://your-instance.looker.com"
LOOKER_CLIENT_ID="..."      LOOKER_CLIENT_SECRET="..."
LOOKER_MODELS='["thelook"]'
AS_APP="..."                # your Gemini Enterprise app ID

AGENT_MODEL_PROVIDER="claude"   # claude | claude_native | anthropic | gemini
CLAUDE_MODEL="claude-sonnet-4-6"
CLAUDE_LOCATION="us-east5"      # Vertex region that serves the model
```

The script refuses to run while any value still starts with `YOUR_`.

### 2. Deploy the A2UI path (Gemini Enterprise)

```bash
./gemini_looker.sh
```

~15–20 minutes on a first run (the Agent Engine deploy is the long part). At the end the agent is live inside your Gemini Enterprise app. Try:

- *"Show me dashboard 1"*
- *"Chart orders by status as a bar chart"*
- *"How many orders are there per status?"*
- *"Give me the interactive link to dashboard 1"*

### 3. Deploy the A2A path (other agents)

After step 2 has run at least through STEP 5 (the agent app must exist under `my-agents/`), copy the same configuration values into the config block of `gemini_looker_a2a.sh` and run:

```bash
./gemini_looker_a2a.sh
```

The script preflights that the agent app, venv, service account and bucket exist (it never recreates them), installs `a2a-sdk` into the existing venv, generates `deploy_a2a.py` via heredoc, and deploys (~15–20 min, with the same background-heartbeat pattern as the main script).

This creates a **second, independent** Reasoning Engine that exposes the agent through A2A endpoints, with an Agent Card describing its skills (`render_dashboard`, `render_chart`, `query_data`, `signed_links`, `schema_discovery`). The original A2UI registration is untouched.

> **Preview note:** the A2A integration on Agent Engine is in preview (`v1beta1`). Pin your `google-cloud-aiplatform` and `a2a-sdk` versions once you have a working combination — `deploy_a2a.py` marks the spot.

---

## Consuming Mirador from another agent (A2A)

Any A2A-compliant client can discover the agent through its Agent Card and send it tasks. The consuming side authenticates with a standard Google Cloud bearer token; its service account needs `roles/aiplatform.user` on this project.

```python
import vertexai
from google.genai import types

client = vertexai.Client(
    project="PROJECT_ID", location="REGION",
    http_options=types.HttpOptions(api_version="v1beta1"),
)
remote_agent = client.agent_engines.get(name="projects/.../reasoningEngines/...")
# remote_agent wraps the A2A endpoints (agent card, send message, get task)
```

From an ADK agent, point a `RemoteA2aAgent` at the agent card URL and use it like any sub-agent.

**What A2A clients receive:** task responses with text parts (answers, signed links) and file parts (the rendered PNGs). Clients that don't render images still get the signed Looker URL as a usable fallback — that behavior is built into the agent's instructions.

## Looker-side setup (required for interactive links)

In **Looker Admin → Embed**:

1. Enable **Embed SSO Authentication** (and Reset Secret).
2. Add the Gemini Enterprise domain to the embed allowlist.
3. Make sure the API user's role can access the model(s) in `LOOKER_MODELS` — otherwise data queries return an access error.

## Reasoning model options

| `AGENT_MODEL_PROVIDER` | Route | Needs |
|---|---|---|
| `claude` | Claude on Vertex AI via LiteLlm | Model enabled in Model Garden; no API key (billed via GCP) |
| `claude_native` | Claude on Vertex AI via ADK's native wrapper (different code path than LiteLlm) | Same as above; recent `google-adk` |
| `anthropic` | Claude via the public Anthropic API | `export ANTHROPIC_API_KEY=...` before running |
| `gemini` | Gemini on Vertex AI | Nothing extra |

Swapping the model never affects image rendering — that lives in the tools and the artifact pipeline, not in the LLM.

## Troubleshooting

- **Preflight 404 on the Claude model** → not enabled in this project, not served in `CLAUDE_LOCATION`, or the model id needs a version suffix. The script prints the exact Model Garden URL to fix it.
- **Gemini Enterprise returns "size 0" but `stream_query` works** → try `AGENT_MODEL_PROVIDER="claude_native"` (a different code path than LiteLlm).
- **Data queries return an access error** → almost always Looker-side: the API credentials' role lacks `access_data`/`explore` or the model set doesn't include your model. Ask the agent to run `list_looker_models` to see what it can actually reach.
- **Runtime errors after deploy** → Logs Explorer → resource type *Vertex AI Reasoning Engine*.

## Roadmap

- [ ] Stream A2A task status updates during long dashboard renders (the protocol supports SSE; the render loop already polls).
- [ ] Per-user embed identity (map the Gemini Enterprise user to `external_user_id` instead of the shared `gemini-user`).
- [ ] Optional row-level filtering via Looker user attributes on the embed session.

---

**Author:** Jose Maldonado ([@joseimj](https://github.com/joseimj))
