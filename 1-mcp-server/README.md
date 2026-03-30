# Part 1: Deploying a Secure MCP Server on Cloud Run

In this part, I built and deployed a **Model Context Protocol (MCP) server** using FastMCP. It exposes zoo animal data as tools that any MCP-compatible LLM client can use. It was then secured with IAM authentication and tested live using **Gemini CLI**.

---

## What I built

A Cloud Run service called `1-mcp-server` that provides two tools:

| Tool | Description |
|------|-------------|
| `get_animals_by_species` | Returns all animals of a given species (e.g. all penguins) |
| `get_animal_details` | Returns full details for a specific animal by name |

The server is secured with `--no-allow-unauthenticated`, meaning only callers with a valid identity token can reach it.

---

## Concepts involved

- What MCP is and how it connects LLMs to external tools
- How to build an MCP server with FastMCP
- How to deploy a Python service to Cloud Run from source
- How to require authentication on a Cloud Run service
- How to connect Gemini CLI to a remote, authenticated MCP server

---

## Prerequisites

- A Google Cloud project with billing enabled
- `gcloud` CLI authenticated with your account
- Python 3.13+ and `uv` installed (both available in Cloud Shell)

> **Tip:** Open [Cloud Shell Editor](https://shell.cloud.google.com/) to follow along — everything is pre-installed.

---

## Files in this folder

```
1-mcp-server/
├── server.py        # FastMCP server — the zoo MCP tools and data
├── Dockerfile       # Container config for Cloud Run deployment
└── pyproject.toml   # Python project config (uv-managed)
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
  cloudbuild.googleapis.com
```

### 3. Navigate into the folder

```bash
cd 1-mcp-server
```

### 4. Install dependencies

```bash
uv add fastmcp==2.12.4 --no-sync
```

---

## Deploy to Cloud Run

### 1. Create a service account for the MCP server

```bash
gcloud iam service-accounts create mcp-server-sa \
  --display-name="MCP Server Service Account"
```

### 2. Deploy

```bash
gcloud run deploy 1-mcp-server \
    --service-account=mcp-server-sa@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com \
    --no-allow-unauthenticated \
    --region=europe-west1 \
    --source=. \
    --labels=part=1-mcp-server
```

When prompted to create an Artifact Registry repository, type `Y` and press Enter.

After a few minutes you should see:

```
Service [1-mcp-server] revision [...] has been deployed and is serving 100 percent of traffic.
```

> Visiting the URL directly returns **403 Forbidden** — that's expected. Authentication is required.

---

## Connect Gemini CLI

### 1. Grant your user account permission to call the MCP server

```bash
gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT \
    --member=user:$(gcloud config get-value account) \
    --role='roles/run.invoker'
```

### 2. Set environment variables

```bash
export PROJECT_NUMBER=$(gcloud projects describe $GOOGLE_CLOUD_PROJECT \
  --format="value(projectNumber)")
export ID_TOKEN=$(gcloud auth print-identity-token)
```

### 3. Configure Gemini CLI

```bash
mkdir -p ~/.gemini
```

Open `~/.gemini/settings.json` and add:

```json
{
  "mcpServers": {
    "zoo-remote": {
      "httpUrl": "https://1-mcp-server-PROJECT_NUMBER.europe-west1.run.app/mcp",
      "headers": {
        "Authorization": "Bearer ID_TOKEN"
      }
    }
  }
}
```

Replace `PROJECT_NUMBER` and `ID_TOKEN` with your actual values.

### 4. Start Gemini CLI and test

```bash
gemini
```

```
/mcp
```

You should see `zoo-remote` listed. Then ask:

```
Where can I find penguins?
```

Select **Yes, always allow all tools from server "zoo-remote"** when prompted.

---

## Verify tool calls in logs (optional)

```bash
gcloud run services logs read 1-mcp-server \
  --region europe-west1 \
  --limit=10
```

You should see lines like:

```
[INFO]: >>> 🛠️ Tool: 'get_animals_by_species' called for 'penguin'
```

---

## Troubleshooting

**Error: MCP server requires authentication / OAuth not found**

Your `ID_TOKEN` has expired (they last ~1 hour). Refresh it:

```bash
export ID_TOKEN=$(gcloud auth print-identity-token)
```

Then restart Gemini CLI.

---

## Clean up

```bash
gcloud run services delete 1-mcp-server --region=europe-west1
```

> **Note:** If you plan to continue to Part 2, keep this service running — Part 2 connects to it.

---

## Next step

→ [Part 2 — Build and deploy an ADK agent that uses this MCP server](../2-adk-agent/README.md)
