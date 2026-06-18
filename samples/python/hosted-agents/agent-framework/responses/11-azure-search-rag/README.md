# Azure AI Search RAG Agent (Responses Protocol)

An [Agent Framework](https://github.com/microsoft/agent-framework) agent with Retrieval Augmented Generation (RAG) backed by Azure AI Search, hosted on Microsoft Foundry using the **Responses protocol**. It uses `AzureAISearchContextProvider` to retrieve Contoso Outdoors product-policy content before each model invocation and asks the model to cite retrieved sources.

## How It Works

1. `main.py` creates a `DefaultAzureCredential` and a `FoundryChatClient` pointed at your Foundry project endpoint and model deployment
2. `_resolved_env()` treats missing or unsubstituted search placeholders as empty so the agent can still start without RAG in smoke-test environments
3. When `AZURE_SEARCH_ENDPOINT` and `AZURE_SEARCH_INDEX_NAME` are set, `AzureAISearchContextProvider` connects to the index in semantic mode with `top_k=3`
4. The search context provider runs before each model invocation and injects matching documents into the model context
5. The agent answers as a Contoso Outdoors support specialist and cites source documents when retrieved context is available
6. `ResponsesHostServer` exposes the agent through an OpenAI-compatible `POST /responses` endpoint and starts on `http://localhost:8088/`

See [main.py](main.py) and [provision_index.py](provision_index.py) for the full implementation.

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `FOUNDRY_PROJECT_ENDPOINT` | Yes | Azure AI Foundry project endpoint URL. Auto-injected when hosted — only needed locally |
| `AZURE_AI_MODEL_DEPLOYMENT_NAME` | Yes | Model deployment name (e.g. `gpt-4.1-mini`). Declared in `agent.yaml` |
| `AZURE_SEARCH_ENDPOINT` | Yes, for RAG | Azure AI Search service endpoint, such as `https://<search-name>.search.windows.net` |
| `AZURE_SEARCH_INDEX_NAME` | Yes, for RAG | Azure AI Search index name. The included provisioning script uses `contoso-outdoors` |

When deployed as a hosted agent, `FOUNDRY_PROJECT_ENDPOINT` is auto-injected by the platform. Search values are declared in `agent.yaml` and should be set in your `azd` environment before deployment. Authentication uses Managed Identity via `DefaultAzureCredential`.

## Running Locally

### Prerequisites

- Python 3.10+
- An Azure AI Foundry project with a model deployment (or let `azd provision` create one)
- An Azure AI Search service
- A search index with the schema shown below, or permission to create and seed it with [provision_index.py](provision_index.py)
- **Search Index Data Reader** on the Azure AI Search service for the identity running the agent

### Using `azd` (Recommended)

Create a local `.env` file from the sample template and fill in the required values:

```bash
cp .env.example .env  # skip if .env already exists
# Edit .env — see Environment Variables above
```

The sample loads `.env` automatically when running locally. Next, start the agent locally with the `run` command:

```bash
azd ai agent run
```

The agent starts on `http://localhost:8088/`.

<details>
<summary><h3>Using the Foundry Toolkit VS Code Extension</h3></summary>

The [Foundry Toolkit VS Code extension](https://learn.microsoft.com/en-us/azure/foundry/agents/quickstarts/quickstart-hosted-agent?view=foundry&pivots=vscode) has a built-in sample gallery that scaffolds this project directly into a new workspace — no manual cloning needed.

1. It's recommended to scaffold the project using the Foundry Toolkit extension. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Create new Hosted Agent`. The extension automatically creates the VS Code debug configuration files and `.env`.
2. Edit `.env` and fill in the required environment variables (see [Environment Variables](#environment-variables) above for the full list).
3. Set up a Python virtual environment:

   **Windows (PowerShell):**
   ```powershell
   python -m venv .venv
   .\.venv\Scripts\Activate.ps1
   ```

   **macOS/Linux:**
   ```bash
   python -m venv .venv
   source .venv/bin/activate
   ```

4. Install dependencies:
   ```bash
   pip install -r requirements.txt
   pip install debugpy
   ```

5. Press **F5** to start the agent in debug mode. The agent starts on `http://localhost:8088/`.

</details>
<details>
<summary><h3>Manual setup</h3></summary>

```bash
pip install -r requirements.txt
cp .env.example .env  # skip if .env already exists
# Edit .env — see Environment Variables above
python main.py
```

The agent starts on `http://localhost:8088/`.

</details>

## Invoke

### Using azd

**Local:**

**Bash:**
```bash
azd ai agent invoke --local "What is the Contoso Outdoors return policy?"
```

**PowerShell:**
```powershell
azd ai agent invoke --local "What is the Contoso Outdoors return policy?"
```

**Test with curl:**

```bash
curl -sS -X POST http://localhost:8088/responses \
  -H "Content-Type: application/json" \
  -d '{"input": "What is the Contoso Outdoors return policy?", "stream": false}' | jq .
```

<details>
<summary><h3>Using Foundry Toolkit VS Code Extension</h3></summary>

Open the **Agent Inspector** directly from the Foundry Toolkit extension to invoke the agent — no `curl` or CLI commands needed.

1. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Open Agent Inspector`.
2. The Inspector auto-connects to your running agent at `http://localhost:8088/`.
3. Type a message and send it. The Agent Inspector handles the protocol and displays the response inline.

> Multi-turn conversation is supported — the Inspector maintains session context across messages.

</details>

## Provisioning the Search Index

[`provision_index.py`](provision_index.py) creates the index if it does not exist and seeds three Contoso Outdoors documents. Your identity needs these roles on the Azure AI Search service:

- **Search Service Contributor** to create the index
- **Search Index Data Contributor** to upload documents

Set the search values, then run the script:

```bash
export AZURE_SEARCH_ENDPOINT="https://<your-search>.search.windows.net"
export AZURE_SEARCH_INDEX_NAME="contoso-outdoors"
python provision_index.py
```

PowerShell:

```powershell
$env:AZURE_SEARCH_ENDPOINT="https://<your-search>.search.windows.net"
$env:AZURE_SEARCH_INDEX_NAME="contoso-outdoors"
python provision_index.py
```

The index schema is:

| Field | Type | Attributes |
|---|---|---|
| `id` | `Edm.String` | key, filterable |
| `content` | `Edm.String` | searchable full text |
| `sourceName` | `Edm.String` | retrievable, filterable |
| `sourceLink` | `Edm.String` | retrievable |

The script is safe to re-run. If you need to change the schema, delete the index first and run the script again.

## How RAG Works in This Sample

When the index is seeded with the included documents, these prompts should retrieve distinct source content:

| User query mentions | Search result injected |
|---|---|
| `return` or `refund` | Contoso Outdoors Return Policy, including canary token `TR-CANARY-7821` |
| `shipping` or `promo` | Contoso Outdoors Shipping Guide, including canary token `SHIP-CANARY-4493` |
| `tent` or `fabric` | TrailRunner Tent Care Instructions, including canary token `TENT-CANARY-9067` |

The canary tokens do not exist in model training data, so seeing one in a response helps confirm the answer was grounded in retrieved content.

## Preparing Hosted Deployment Values

Set the Azure AI Search values in your `azd` environment before deploying:

```bash
azd env set AZURE_SEARCH_ENDPOINT "https://<your-search>.search.windows.net"
azd env set AZURE_SEARCH_INDEX_NAME "contoso-outdoors"
```

After deployment, assign the hosted agent's managed identity **Search Index Data Reader** on the Azure AI Search service.

## Deploying the Agent to Microsoft Foundry

### Using azd

Once you've tested locally, deploy to Microsoft Foundry:

```bash
# Provision Azure resources (skip if already done during local setup)
azd provision

# Build, push, and deploy the agent to Foundry
azd deploy
```

After deploying, invoke the agent running in Foundry:

**Bash:**
```bash
azd ai agent invoke "What is the Contoso Outdoors return policy?"
```

**PowerShell:**
```powershell
azd ai agent invoke "What is the Contoso Outdoors return policy?"
```

To stream logs from the running agent:

```bash
azd ai agent monitor
```

For the full deployment guide, see [Azure AI Foundry hosted agents](https://aka.ms/azdaiagent/docs).

<details>
<summary><h3>Using the Foundry Toolkit VS Code Extension</h3></summary>

1. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Deploy Hosted Agent`. The extension opens a tab-based **Deploy Hosted Agent** wizard and reads `agent.yaml` to auto-populate what it can.
2. If prompted, complete **Foundry Project Setup** to pick the subscription and Foundry project (or create a new one) to deploy to.
3. On the **Basics** tab, configure the core deployment settings:
   - **Deployment Method**: **Code** (upload as a ZIP) or **Container** (Docker image via ACR).
   - For **Code**, pick a packaging option: **Remote** or **Local**.
   - For **Container**, pick a registry option: default ACR, your own ACR, or a prebuilt ACR image.
   - **Hosted Agent Name**: confirm the name to register with the hosting service.
4. On the **Review + Deploy** tab, finalize the runtime and resources:
   - Confirm the auto-detected runtime details (language, entry point, or Dockerfile).
   - Pick a **CPU and Memory** size.
   - Click **Deploy**. Fields are validated inline, and the extension handles the build/upload, agent version creation, and RBAC role assignment.
5. After deployment, invoke the agent in the Agent Playground and stream live logs from the **Logs** tab.

</details>

## Troubleshooting

### Images built on Apple Silicon or other ARM64 machines do not work on our service

We **recommend deploying with `azd deploy`**, which uses ACR remote build and always produces images with the correct architecture.

If you choose to **build locally**, and your machine is **not `linux/amd64`** (for example, an Apple Silicon Mac), the image will **not be compatible with our service**, causing runtime failures.

**Fix for local builds:**

```bash
docker build --platform=linux/amd64 -t image .
```

This forces the image to be built for the required `amd64` architecture.

### Azure AI Search returns `403 Forbidden`

Confirm the calling identity has **Search Index Data Reader** for runtime queries. If you are provisioning the index, your identity also needs **Search Service Contributor** and **Search Index Data Contributor**. Also confirm the search service API access control allows RBAC, not API-key-only access.

### The agent starts but answers without retrieved context

Check that `AZURE_SEARCH_ENDPOINT` and `AZURE_SEARCH_INDEX_NAME` are set to real values, not unresolved placeholders. The code intentionally starts without RAG when those values are empty.
