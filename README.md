# AI Agent from Production to Deployment

A 3-part project built with Google Cloud, showing how to go from a simple MCP server all the way to a GPU-accelerated AI agent, which is all deployed on **Google Cloud Run**.

Each part is self-contained with its own README, but they build on each other. 

---

## What is Built

| Part | Folder | What is built | Key tech |
|-----|--------|----------------|----------|
| **1** | [`1-mcp-server/`](./1-mcp-server/) | A secure, production-ready MCP server that exposes zoo animal data as tools for LLMs | FastMCP, Cloud Run, Gemini CLI |
| **2** | [`2-adk-agent/`](./2-adk-agent/) | A multi-agent zoo tour guide that uses the MCP server + Wikipedia | ADK, SequentialAgent, LangChain, Cloud Run |
| **3** | [`3-gpu-agent/`](./3-gpu-agent/) | A GPU-accelerated Gemma agent with elasticity testing | Ollama, Gemma 3 270M, NVIDIA L4, Cloud Run |

<img width="2048" height="1117" alt="image" src="https://github.com/user-attachments/assets/5e751753-873f-4110-94cb-036496a0831b" />

---

## Architecture Overview

<img width="816" height="527" alt="image" src="https://github.com/user-attachments/assets/6b021879-2214-4a16-b1b3-b517db690118" />




---

## Prerequisites

Before starting, make sure you have:

- A **Google Cloud account** with billing enabled ([free trial available](https://cloud.google.com/free))
- The **gcloud CLI** installed and authenticated
- **Python 3.13+**
- **uv** — the Python package manager used throughout ([install](https://docs.astral.sh/uv/getting-started/installation/))

> All parts can run in **Google Cloud Shell**, which comes with all of the above pre-installed. Click [here](https://shell.cloud.google.com/) to open it.

---

## Getting Started

Clone the repo and navigate into it:

```bash
git clone https://github.com/YOUR_USERNAME/zoo-mcp-on-cloudrun.git
cd zoo-mcp-on-cloudrun
```

Then start with Part 1:

```bash
cd 1-mcp-server
cat README.md
```

---

## Cost Estimate

| Part | Approximate cost |
|-----|-----------------|
| Part 1 | < $1 USD |
| Part 2 | < $1 USD |
| Part 3 | ~$2–4/hr while GPU is running (NVIDIA L4) |

Each part README includes a **clean up** section to delete resources and avoid ongoing charges.

---

## Project at a Glance

### Part 1 — Deploying a secure MCP server on Cloud Run

Building a [Model Context Protocol](https://modelcontextprotocol.io/) server using **FastMCP** that exposes zoo animal data as tools. Deploying it to Cloud Run with authentication required, then connecting to it using **Gemini CLI**.

**Concepts involved:** MCP concepts, FastMCP, deploying from source on Cloud Run, IAM-based auth, service accounts.

→ [Go to Part 1](./1-mcp-server/README.md)

---

### Part 2 — Building and Deploying an ADK Agent that uses an MCP Server

Building a multi-agent zoo tour guide using Google's **Agent Development Kit (ADK)**. The agent uses the MCP server from Part 1 as its toolset, augmented with the Wikipedia API for general knowledge. Deploying the agent to Cloud Run.

**Concepts involved:** ADK agents, `SequentialAgent`, `MCPToolset`, `LangchainTool`, state management, `adk deploy`.

→ [Go to Part 2](./2-adk-agent/README.md)

---

### Part 3 — Deploying an ADK agent to Cloud Run with GPU

Deploying a **GPU-accelerated Gemma 3** model via Ollama on Cloud Run, then wiring it up to an ADK agent. Running elasticity tests with **Locust** to observe how both services handle load independently.

**Concepts involved:** GPU on Cloud Run, Ollama, LiteLlm, FastAPI + ADK, Locust load testing, autoscaling behavior.

→ [Go to Part 3](./lab3-gpu-agent/README.md)

---

## Project Structure

```
ai-agent-prod-deployment/
├── README.md                        # You are here
├── .gitignore
├── 1-mcp-server/
│   ├── README.md
│   ├── server.py                    # FastMCP zoo server with 2 tools
│   ├── Dockerfile
│   └── pyproject.toml
├── 2-adk-agent/
│   ├── README.md
│   ├── zoo_guide_agent/
│   │   ├── __init__.py
│   │   └── agent.py                 # Multi-agent workflow (greeter → researcher → formatter)
│   ├── requirements.txt
│   └── .env.example
└── 3-gpu-agent/
    ├── README.md
    ├── ollama-backend/
    │   └── Dockerfile               # Gemma 3 270M via Ollama
    └── adk-agent/
        ├── production_agent/
        │   ├── __init__.py
        │   └── agent.py             # Gemma-powered conversational agent
        ├── server.py                # FastAPI server with ADK integration
        ├── Dockerfile
        ├── elasticity_test.py       # Locust load test
        ├── pyproject.toml
        └── .env.example
```

---


