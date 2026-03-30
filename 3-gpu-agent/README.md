# Part 3: Deploying an ADK Agent with GPU-Accelerated Gemma 

In this part, I deployed a **production-ready AI agent** backed by a GPU-accelerated **Gemma 3 270M** model running via Ollama on Cloud Run. I then ran an **elasticity test** with Locust to observe how both services handle load independently.

This part is **standalone** — it does not require parts 1 or 2.

---

## What I built

Two Cloud Run services that work together:

```
┌──────────────────────────────┐      inference      ┌──────────────────────────────┐
│  production-adk-agent        │ ──────────────────► │  ollama-gemma3-270m-gpu      │
│  Cloud Run (CPU only)        │                     │  Cloud Run (NVIDIA L4 GPU)   │
│  ADK + FastAPI               │                     │  Ollama + Gemma 3 270M       │
│  1 instance, 50 concurrency  │                     │  1–3 instances, 7 concurrency│
└──────────────────────────────┘                     └──────────────────────────────┘
        ▲
        │
   Locust load test
   (elasticity_test.py)
```

The agent is a conversational zoo guide named **Gem**, powered by Gemma. Unlike Parts 1 and 2, this agent answers from general knowledge — it has no MCP tools or Wikipedia access. The emphasis is on the deployment architecture and production patterns.

---

## What I learnt

- How to deploy a GPU-accelerated model (Gemma 3 via Ollama) on Cloud Run
- How to connect an ADK agent to an external LLM backend using `LiteLlm`
- How to serve an ADK agent with FastAPI
- How to configure environment variables in Cloud Run
- How to run load tests with Locust and observe independent autoscaling

---

## Prerequisites

- A Google Cloud project with billing enabled (GPU quota must be available in `europe-west1`)
- `gcloud` CLI authenticated
- Python 3.13+ and `uv` installed

> **Cost warning:** The Ollama backend uses an NVIDIA L4 GPU, which costs approximately **$2–4/hr** while running. Delete the service when you're done (see Clean up below).

---

## Files in this folder

```
3-gpu-agent/
├── ollama-backend/
│   └── Dockerfile               # Ollama + Gemma 3 270M, baked into the image
└── adk-agent/
    ├── production_agent/
    │   ├── __init__.py
    │   └── agent.py             # Gemma-powered "Gem" zoo guide agent
    ├── server.py                # FastAPI server wrapping the ADK agent
    ├── Dockerfile               # Container config for the ADK agent service
    ├── elasticity_test.py       # Locust load test script
    ├── pyproject.toml           # Python dependencies (uv-managed)
    └── .env.example             # Template — copy to .env and fill in
```

---

## Setup

### 1. Set your project and region

```bash
gcloud config set project YOUR_PROJECT_ID
gcloud config set run/region europe-west1
```

### 2. Enable required APIs

```bash
gcloud services enable \
  run.googleapis.com \
  artifactregistry.googleapis.com \
  cloudbuild.googleapis.com \
  aiplatform.googleapis.com
```

### 3. Clone the starter code

The starter structure for this lab is sourced from the Google Cloud DevRel demos repo:

```bash
cd ~
git clone --depth 1 --filter=blob:none --sparse \
  https://github.com/GoogleCloudPlatform/devrel-demos.git temp-repo \
  && cd temp-repo \
  && git sparse-checkout set agents/accelerate-ai-with-cloud-run/accelerate-ai-part3-starter \
  && cd .. \
  && mv temp-repo/agents/accelerate-ai-with-cloud-run/accelerate-ai-part3-starter . \
  && rm -rf temp-repo

cd accelerate-ai-part3-starter
```

---

## Step 1 — Deploy the Gemma backend

The Ollama backend bakes the Gemma 3 270M weights into the container image at build time so the model is ready immediately on startup.

```bash
cd ollama-backend

gcloud run deploy ollama-gemma3-270m-gpu \
  --source . \
  --region europe-west1 \
  --concurrency 7 \
  --cpu 8 \
  --set-env-vars OLLAMA_NUM_PARALLEL=4 \
  --gpu 1 \
  --gpu-type nvidia-l4 \
  --max-instances 1 \
  --memory 16Gi \
  --allow-unauthenticated \
  --no-cpu-throttling \
  --no-gpu-zonal-redundancy \
  --timeout 600 \
  --labels part=part3-gpu-agent
```

> This takes ~5 minutes. The model weights are pulled during the Docker build.

Save the backend URL:

```bash
export OLLAMA_URL=$(gcloud run services describe ollama-gemma3-270m-gpu \
    --region=europe-west1 \
    --format='value(status.url)')

echo "Gemma backend: $OLLAMA_URL"
```

### Why these settings?

| Setting | Value | Reason |
|---------|-------|--------|
| `--gpu-type` | `nvidia-l4` | Best price/performance for inference workloads; 24GB GPU memory |
| `--memory` | 16Gi | Covers model loading + CUDA ops + Ollama memory management |
| `--cpu` | 8 | Handles I/O and preprocessing alongside GPU inference |
| `--concurrency` | 7 | Balances throughput with GPU memory usage |
| `--max-instances` | 1 | Prevents runaway GPU costs during testing |
| `--timeout` | 600 | Accommodates model loading on first startup |

---

## Step 2 — Configure and deploy the ADK agent

```bash
cd ../adk-agent
```

Create your `.env` file:

```bash
cat << EOF > .env
GOOGLE_CLOUD_PROJECT=$(gcloud config get-value project)
GOOGLE_CLOUD_LOCATION=europe-west1
GEMMA_MODEL_NAME=gemma3:270m
OLLAMA_API_BASE=$OLLAMA_URL
EOF
```

Deploy the ADK agent service:

```bash
export PROJECT_ID=$(gcloud config get-value project)

gcloud run deploy production-adk-agent \
   --source . \
   --region europe-west1 \
   --allow-unauthenticated \
   --memory 4Gi \
   --cpu 2 \
   --max-instances 1 \
   --concurrency 50 \
   --timeout 300 \
   --set-env-vars GOOGLE_CLOUD_PROJECT=$PROJECT_ID \
   --set-env-vars GOOGLE_CLOUD_LOCATION=europe-west1 \
   --set-env-vars GEMMA_MODEL_NAME=gemma3:270m \
   --set-env-vars OLLAMA_API_BASE=$OLLAMA_URL \
   --labels part=part3-gpu-agent
```

Get the agent URL:

```bash
export AGENT_URL=$(gcloud run services describe production-adk-agent \
    --region=europe-west1 \
    --format='value(status.url)')

echo "ADK Agent: $AGENT_URL"
```

---

## Step 3 — Test the agent

Check the health endpoint:

```bash
curl $AGENT_URL/health
# Expected: {"status": "healthy", "service": "production-adk-agent"}
```

Then open `$AGENT_URL` in a browser to access the ADK web UI and chat with **Gem**:

- *"What do red pandas eat in the wild?"*
- *"Tell me an interesting fact about snow leopards."*
- *"Why are poison dart frogs so brightly colored?"*

> Note: Gem answers from general knowledge only — she has no access to zoo-specific data (no MCP tools). If you ask where to find a specific animal in the zoo, she'll say she doesn't know.

---

## Step 4 — Run an elasticity test

Install dependencies:

```bash
uv sync
```

Run Locust in headless mode:

```bash
uv run locust -f elasticity_test.py \
   -H $AGENT_URL \
   --headless \
   -t 60s \
   -u 20 \
   -r 5
```

**Parameters:**
- `-t 60s` — run for 60 seconds
- `-u 20` — 20 concurrent simulated users
- `-r 5` — spawn 5 users per second

While the test runs, watch the Cloud Run metrics in the console:

- **Cloud Run → production-adk-agent → Metrics** — CPU/memory usage, request count
- **Cloud Run → ollama-gemma3-270m-gpu → Metrics** — GPU utilization becomes the bottleneck

Both services stay at 1 instance (`max-instances 1`), but you'll see the GPU backend become the bottleneck as the model handles inference requests.

---

## Clean up

```bash
gcloud run services delete production-adk-agent --region=europe-west1
gcloud run services delete ollama-gemma3-270m-gpu --region=europe-west1
```

---

## Security note

This part uses `--allow-unauthenticated` for ease of testing. In production, use:

- Cloud Run service-to-service auth with service accounts
- IAM policy bindings with `roles/run.invoker`
- API keys or OAuth for external-facing endpoints
