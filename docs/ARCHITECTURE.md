# Architecture. M365 LangChain Agent

C4 model diagrams from system context down to request flow.

## Level 1. System Context

```
       ┌───────────┐
       │ Employee  │
       └─────┬─────┘
             │  Asks questions through Teams, WebChat, or M365 Copilot
             ▼
  ┌─────────────────────────────────────────────┐
  │          Azure Bot Service                  │
  │          Microsoft managed                  │
  │  Routes messages and handles channel auth   │
  └──────────────────┬──────────────────────────┘
                     │  HTTPS POST /api/messages
                     ▼
  ┌─────────────────────────────────────────────┐
  │        M365 LangChain Agent                 │
  │   Search → Deduplicate → Generate → Cite    │
  └───┬──────────────┬──────────────┬───────────┘
      │              │              │
      ▼              ▼              ▼
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ Azure AI │  │  Azure   │  │  Azure   │
  │  Search  │  │  OpenAI  │  │ CosmosDB │
  └──────────┘  └──────────┘  └──────────┘

  ┌──────────┐
  │ LangSmith│  Optional tracing
  └──────────┘
```

| From | To | Protocol | Purpose |
|---|---|---|---|
| Employee | Bot Service | Teams SDK or WebChat | User asks a question |
| Bot Service | Agent | HTTPS POST /api/messages | Sends activity payload |
| Agent | Azure AI Search | HTTPS | Runs hybrid search |
| Agent | Azure OpenAI | HTTPS | Embeddings and answer generation |
| Agent | CosmosDB | HTTPS | Reads and writes conversation history |
| Agent | LangSmith | HTTPS | Optional tracing |

## Level 2. Container Diagram

| Container | Technology | Purpose |
|---|---|---|
| M365 LangChain Agent | Python 3.10, FastAPI, uvicorn | Handles requests and runs the RAG pipeline |
| Container Apps Ingress | Azure managed | TLS termination and routing |
| Azure Bot Service | Microsoft managed | Channel and activity routing |
| Azure AI Search | Microsoft managed | Document index and retrieval |
| Azure OpenAI | Microsoft managed | LLM and embeddings |
| Azure CosmosDB | Microsoft managed | Conversation storage |
| LangSmith | SaaS | Observability |

## Level 3. Component Diagram

```
  app.py
     │
     ▼
  bot.py
     │
     ├──► agent.py
     │        │
     │        └──► utils/search.py
     │
     └──► cosmos_store.py

  foundry_register.py
```

**Responsibilities:**

- **app.py.** FastAPI app, adapter setup, health endpoints, managed identity adapter support
- **bot.py.** Handles messages and coordinates request flow
- **agent.py.** Runs retrieval, deduplication, prompt building, answer generation, and citation formatting
- **cosmos_store.py.** Stores and retrieves conversation history
- **search.py.** Azure AI Search client for hybrid retrieval
- **chainlit_app.py.** Chainlit interface with runtime settings and debug output
- **foundry_register.py.** Script for Azure AI Foundry registration

## Level 4. Request Flow

| Step | Component | Action |
|---|---|---|
| 1 | Employee | Sends a question |
| 2 | Bot Service | Forwards request to the app |
| 3 | Container Apps | Routes traffic to port 8080 |
| 4 | app.py | Deserializes the activity |
| 5 | bot.py | Extracts the user message |
| 6 | cosmos_store.py | Loads recent conversation history |
| 7 | agent.py | Creates embedding for the query |
| 8 | search.py | Runs hybrid retrieval |
| 9 | agent.py | Builds prompt from instructions, history, and sources |
| 10 | agent.py | Calls Azure OpenAI for answer generation |
| 11 | agent.py | Attaches citations |
| 12 | cosmos_store.py | Saves the turn |
| 13 | bot.py | Sends the response |
| 14 | Employee | Receives cited answer |

## Authentication and Security

### Inbound

Azure Bot Service sends a JWT in the Authorization header. The adapter validates signature, issuer, audience, expiry, and channel claims.

### Outbound

| Mode | Condition | Method |
|---|---|---|
| Managed Identity | BOT_APP_ID set and BOT_APP_PASSWORD empty | Custom MsiBotFrameworkAdapter using ManagedIdentityCredential |
| Client Secret | Both values set | Standard Bot Framework credential flow |

### Backend Connections

| Connection | Auth Method |
|---|---|
| Agent → Azure OpenAI | Managed Identity |
| Agent → Azure AI Search | Managed Identity |
| Agent → Azure CosmosDB | Managed Identity |
| Agent → LangSmith | API key |

### Network Security

| Service | Access Mode |
|---|---|
| CosmosDB | Private endpoint preferred |
| Azure OpenAI | Public endpoint or private endpoint |
| Azure AI Search | Public endpoint or private endpoint |
| LangSmith | Outbound HTTPS |

### Hardening Checklist

| Item | Current | Recommendation |
|---|---|---|
| Bot auth | Managed identity supported | Move to newer SDK path when stable |
| Backend auth | Managed identity | Keep RBAC strict |
| CosmosDB network | Private endpoint preferred | Use private access in production |
| OpenAI and Search | Public or private | Lock down where possible |
| Secret handling | Environment based | Move secrets into Key Vault |
| Logging | LangSmith | Add Azure Monitor and Log Analytics |

## Data Flow

The application is read only against the search index. Ingestion is handled outside this project.

### Ingestion pipeline

```
Document Source → Parse → Chunk → Redact → Embed → Index
                                                   │
                                             Azure AI Search
                                                   │ Read only
                                                   ▼
```

### Query pipeline

```
User → Bot Service → Agent → AI Search + OpenAI + CosmosDB → Cited answer
```
