# M365 LangChain Agent

RAG agent for Microsoft 365. It answers employee questions from an internal knowledge base with source backed responses, and runs as a single container on Azure Container Apps.

**Stack:** Python 3.10 | LangChain | Azure Bot Service | CosmosDB | Azure AI Search | Azure OpenAI

**Architecture:** Single container, 1 replica, no Redis, no Postgres, no LangGraph. CosmosDB stores conversation state.

> Full architecture with C4 diagrams: **[ARCHITECTURE.md](docs/ARCHITECTURE.md)**

---

## Table of Contents

- [Quick Start](#quick-start)
- [User Interfaces](#user-interfaces)
- [RAG Pipeline](#rag-pipeline)
- [Deployment](#deployment)
- [Feature Flags](#feature-flags)
- [Environment Variables](#environment-variables)
- [Project Structure](#project-structure)
- [AI Foundry Integration](#ai-foundry-integration)
- [Operational Commands](#operational-commands)
- [Prerequisites](#prerequisites)

---

## Quick Start

```bash
# 1. Install
cd m365-langchain-agent
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

# 2. Configure
cp .env.example .env   # Fill in your Azure resource values

# 3. Run in Bot Service mode
python app.py

# 4. Run in Chainlit UI mode
USER_INTERFACE=CHAINLIT_UI python app.py
# Open http://localhost:8080/chat/

# 5. Verify
curl http://localhost:8080/health
# {"status":"healthy","service":"m365-langchain-agent"}
```

## User Interfaces

The agent supports two interface modes through the USER_INTERFACE environment variable. Both use the same FastAPI app and share /health, /readiness, and /test/query endpoints.

### Bot Service Mode

`USER_INTERFACE=BOT_SERVICE`

Exposes POST /api/messages for Azure Bot Service. Messages from Teams, WebChat, and DirectLine are passed into the RAG pipeline. Responses include inline citations rendered as markdown links.

**Flow:**

1. Azure Bot Service sends an Activity payload to /api/messages
2. BotFrameworkAdapter validates the request
3. DocAgentBot.on_message_activity() extracts the message text
4. Conversation history is loaded from CosmosDB
5. invoke_agent() runs search and answer generation
6. The response is sent back through Bot Service
7. The turn is saved to CosmosDB for multi turn context

**Managed identity support:**
When BOT_APP_ID is set and BOT_APP_PASSWORD is empty, the app uses MsiBotFrameworkAdapter, a custom adapter that uses ManagedIdentityCredential for token acquisition.

### Chainlit UI Mode

`USER_INTERFACE=CHAINLIT_UI`

Provides a browser based chat interface at /chat/, with / redirected to /chat/.

Features include:

- runtime settings panel
- model switching
- debug accordions
- clickable citations
- conversation history through CosmosDB

#### Chainlit Settings Panel

The settings panel lets you tune agent behavior without restarting the app.

| Setting | Type | Default | Range | Description |
|---|---|---|---|---|
| Model | Dropdown | gpt-4.1 | Configured models | Azure OpenAI deployment to use |
| Top K | Slider | 5 | 1 to 20 | Number of chunks retrieved from Azure AI Search |
| Temperature | Slider | 0.2 | 0.0 to 1.0 | Output randomness. Ignored for reasoning models |
| System Prompt | Text Input | Default RAG prompt | Free text | System instructions passed to the model |

**Model behavior:**

- Standard models such as GPT 4.1 respect temperature
- Reasoning models such as o1 and o3 use preview API versions and ignore temperature

#### Debug Panel

Each Chainlit response includes three expandable debug sections:

- **Retrieved Chunks.** Search results with scores, metadata, and content
- **Full LLM Prompt.** The prompt sent to Azure OpenAI
- **Settings Used.** The active settings for that response

## RAG Pipeline

The agent follows a direct flow:

```
User Question
     |
     v
[1] Azure AI Search
     | hybrid retrieval with keyword, vector, and semantic ranking
     v
[2] Deduplicate Sources
     | keep the highest scoring chunk per source
     v
[3] Build Context
     | format source chunks and recent conversation history
     v
[4] Generate Answer
     | Azure OpenAI produces the response with citation rules
     v
[5] Return Result
     | answer, sources, raw chunks, prompt
     v
[6] Save Turn
     | CosmosDB stores recent conversation history
```

### Search Details

Hybrid search combines:

- Keyword retrieval using BM25
- Vector retrieval using embeddings
- Semantic reranking for better final ordering

## Deployment

### Azure Container Apps

```bash
cp .env.example .env
chmod +x deploy-container-apps.sh
./deploy-container-apps.sh
```

### Manual Deployment

```bash
docker buildx build --platform linux/amd64 \
  -t <your-acr>.azurecr.io/m365-langchain-agent:latest .

az acr login --name <your-acr>
docker push <your-acr>.azurecr.io/m365-langchain-agent:latest

az containerapp update \
  --name m365-langchain-agent \
  --resource-group <your-rg> \
  --image <your-acr>.azurecr.io/m365-langchain-agent:latest
```

## Feature Flags

| Variable | Value | Behavior |
|---|---|---|
| USER_INTERFACE | BOT_SERVICE | FastAPI with Bot Framework adapter |
| USER_INTERFACE | CHAINLIT_UI | FastAPI with Chainlit at /chat/ |
| LANGCHAIN_TRACING_V2 | true | Sends traces to LangSmith |
| AZURE_OPENAI_AVAILABLE_MODELS | Comma separated list | Models shown in Chainlit |

## Environment Variables

All settings are externalized. See .env.example for the full list.

| Group | Variables | Required |
|---|---|---|
| Azure OpenAI | AZURE_OPENAI_ENDPOINT, AZURE_OPENAI_DEPLOYMENT_NAME, AZURE_OPENAI_API_VERSION, AZURE_OPENAI_EMBEDDING_DEPLOYMENT | Yes |
| Azure AI Search | AZURE_SEARCH_ENDPOINT, AZURE_SEARCH_INDEX_NAME, AZURE_SEARCH_SEMANTIC_CONFIG_NAME | Yes |
| CosmosDB | AZURE_COSMOS_ENDPOINT, AZURE_COSMOS_DATABASE, AZURE_COSMOS_CONTAINER | Yes |
| Bot Framework | BOT_APP_ID, BOT_APP_PASSWORD, BOT_AUTH_TENANT | Bot mode only |
| AI Foundry | AZURE_FOUNDRY_ENDPOINT, AZURE_FOUNDRY_SUBSCRIPTION_ID, AZURE_FOUNDRY_RESOURCE_GROUP | Optional |
| LangSmith | LANGSMITH_API_KEY, LANGCHAIN_TRACING_V2, LANGCHAIN_PROJECT | Optional |

## Project Structure

```
m365-langchain-agent/
├── app.py
├── m365_langchain_agent/
│   ├── bot.py
│   ├── agent.py
│   ├── chainlit_app.py
│   ├── cosmos_store.py
│   ├── foundry_register.py
│   └── utils/
│       └── search.py
├── scripts/
│   └── register_foundry_agent.py
├── deploy-container-apps.sh
├── Dockerfile
├── docs/
│   ├── ARCHITECTURE.md
│   └── images/
├── requirements.txt
├── pyproject.toml
└── .env.example
```

## AI Foundry Integration

Use the registration script to register the agent in Azure AI Foundry.

```bash
python scripts/register_foundry_agent.py
python scripts/register_foundry_agent.py --list
python scripts/register_foundry_agent.py --delete <agent-id>
```

## Operational Commands

```bash
curl -sk https://<your-fqdn>/health
curl -sk https://<your-fqdn>/readiness

curl -X POST https://<your-fqdn>/test/query \
  -H "Content-Type: application/json" \
  -d '{"query": "your question here"}'

az containerapp logs show --name m365-langchain-agent --resource-group <your-rg> --follow
```

## Prerequisites

| Resource | Purpose |
|---|---|
| Azure Container Apps | Container hosting |
| Azure Container Registry | Image storage |
| Azure OpenAI | LLM and embeddings |
| Azure AI Search | Hybrid search index |
| Azure Bot Service | Teams, WebChat, and DirectLine routing |
| Azure CosmosDB | Conversation state |
| User Assigned Managed Identity | Authentication without client secrets |
| AI Foundry Hub and Project | Optional agent registration |

**Local development:** Python 3.10+, Docker Desktop, Azure CLI
