# Part 2: Building and Deploying an ADK Agent with an MCP Server 

In this part, I built a **multi-agent zoo tour guide** using Google's Agent Development Kit (ADK). The agent connects to the MCP server from Part 1 for zoo-specific data, and to Wikipedia for general animal knowledge. The full system is deployed as a Cloud Run service with the ADK web UI.

---

## What I built

A Cloud Run service called `zoo-tour-guide` — a multi-agent pipeline made up of three agents:

```
User prompt
    │
    ▼
┌─────────────────────────────────────────────┐
│  greeter (root_agent)                       │
│  Welcomes the visitor, saves their prompt   │
│  to shared state, then hands off            │
└──────────────────────┬──────────────────────┘
                       │ transfers to
                       ▼
┌─────────────────────────────────────────────┐
│  tour_guide_workflow (SequentialAgent)      │
│                                             │
│  Step 1: comprehensive_researcher           │
│    → uses MCP tools (zoo data)              │
│    → uses Wikipedia (general facts)         │
│                                             │
│  Step 2: response_formatter                 │
│    → takes raw research data                │
│    → outputs a friendly tour guide response │
└─────────────────────────────────────────────┘
```

---

## What I learnt

- How to structure a Python project for ADK deployment
- How to use `MCPToolset` to connect an agent to a remote MCP server
- How to use `LangchainTool` to wrap third-party tools (Wikipedia)
- How `SequentialAgent` orchestrates a multi-step workflow
- How agents share data using `tool_context.state` and `output_key`
- How to deploy an ADK agent with `adk deploy cloud_run`

---

## Prerequisites

- **Part 1 completed** — the `1-mcp-server` must be running on Cloud Run
- A Google Cloud project with billing enabled
- `gcloud` CLI authenticated
- Python 3.13+ and `uv` installed

---

## Files in this folder

```
2-adk-agent/
├── zoo_guide_agent/
│   ├── __init__.py    # Marks this directory as a Python package
│   └── agent.py       # All agent logic — tools, agents, workflow
├── requirements.txt   # Python dependencies
└── .env.example       # Template — copy to .env and fill in your values
```

---

## Setup

### 1. Set your project

```bash
gcloud config set project YOUR_PROJECT_ID
```

### 2. Enable required APIs

```bash
gcloud services enable \
    run.googleapis.com \
    artifactregistry.googleapis.com \
    cloudbuild.googleapis.com \
    aiplatform.googleapis.com
```

### 3. Navigate into the folder

```bash
cd 2-adk-agent
```

### 4. Create a service account

```bash
export PROJECT_ID=$(gcloud config get-value project)
export PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID \
  --format="value(projectNumber)")
export SA_NAME=part2-cr-service
export SERVICE_ACCOUNT="${SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"

gcloud iam service-accounts create ${SA_NAME} \
    --display-name="Service Account for Part 2"
```

### 5. Grant permissions

```bash
# Permission to use Gemini on Vertex AI
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:$SERVICE_ACCOUNT" \
  --role="roles/aiplatform.user"

# Permission to call the Part 1 MCP server
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:$SERVICE_ACCOUNT" \
  --role="roles/run.invoker"
```

### 6. Create your `.env` file

```bash
cp .env.example .env
```

Then open `.env` and fill in your values. At minimum:

```ini
MODEL="gemini-2.5-flash"
MCP_SERVER_URL=https://1-mcp-server-PROJECT_NUMBER.europe-west1.run.app/mcp
```

Replace `PROJECT_NUMBER` with your actual project number (from step 4 above).

---

## How the agent works

### Tools

| Tool | Type | Purpose |
|------|------|---------|
| `add_prompt_to_state` | Python function | Saves the user's question to shared agent state |
| `MCPToolset` | MCP | Connects to the zoo MCP server for animal data |
| `wikipedia_tool` | LangchainTool | Searches Wikipedia for general animal facts |

### Agent authentication to MCP server

The agent uses `google.oauth2.id_token.fetch_id_token` to automatically get a short-lived identity token, allowing it to call the authenticated MCP server without any manual token management.

### State flow

```
greeter
  → saves user prompt to state["PROMPT"]
  → comprehensive_researcher reads state["PROMPT"]
  → comprehensive_researcher saves results to state["RESEARCH_DATA"]  (via output_key)
  → response_formatter reads state["RESEARCH_DATA"]
  → final response delivered to user
```

---

## Deploy

```bash
uvx --from google-adk==1.14.0 \
adk deploy cloud_run \
  --project=$PROJECT_ID \
  --region=europe-west1 \
  --service_name=zoo-tour-guide \
  --with_ui \
  . \
  -- \
  --labels=part=2-adk-agent \
  --service-account=$SERVICE_ACCOUNT
```

When prompted:
- **Artifact Registry repository**: type `Y`
- **Allow unauthenticated invocations**: type `y` (for testing purposes)

Copy the service URL from the output.

---

## Test it

Open the service URL in a browser. You'll see the ADK developer web UI. Toggle on **Token Streaming** in the top right.

Start a conversation:

```
hello
```

Then ask something like:

```
Where can I find the polar bears and what do they eat in the wild?
```

Watch the agent use both the MCP server (for zoo-specific info) and Wikipedia (for diet facts), then combine the results into a friendly response.

---

## Clean up

```bash
gcloud run services delete zoo-tour-guide --region=europe-west1 --quiet
gcloud artifacts repositories delete cloud-run-source-deploy \
  --location=europe-west1 --quiet
```

---

## Next step

→ [Part 3 — Deploy an ADK agent with GPU-accelerated Gemma](../3-gpu-agent/README.md)
