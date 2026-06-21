# Azure AI Search RAG Agent (Responses Protocol)

A Contoso Outdoors support agent hosted on Microsoft Foundry using the [Agent Framework](https://github.com/microsoft/agent-framework) and the **Responses protocol**. It grounds answers in a keyword-indexed Azure AI Search knowledge base through `TextSearchProvider` and `Azure.Search.Documents`.

## How It Works

1. `Program.cs` loads local `.env` values via `DotNetEnv`, then reads the Foundry project endpoint, chat deployment, Azure AI Search endpoint, and index name
2. A `SearchClient` authenticates to Azure AI Search with `DefaultAzureCredential`; no search API key is used by the agent runtime
3. `TextSearchProvider` is configured to search before each AI invocation and include up to three top results from the recent conversation context
4. The search adapter runs full-text search, then maps each hit's `content`, `sourceName`, and `sourceLink` into `TextSearchResult` entries
5. `.AsAIAgent(new ChatClientAgentOptions { ... AIContextProviders = [...] })` creates a support assistant instructed to cite source documents when available
6. `AgentHost.CreateBuilder(args)` plus `AddFoundryResponses(agent)` and `MapFoundryResponses()` host the agent behind `POST /responses`
7. The agent starts on `http://localhost:8088/`

See [Program.cs](Program.cs) for the full implementation.

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `FOUNDRY_PROJECT_ENDPOINT` | Yes | Azure AI Foundry project endpoint URL. Auto-injected when hosted — only needed locally |
| `AZURE_AI_MODEL_DEPLOYMENT_NAME` | No | Chat model deployment name. Defaults to `gpt-4.1-mini`. Declared in `agent.yaml` and `agent.manifest.yaml` |
| `AZURE_SEARCH_ENDPOINT` | Yes | Azure AI Search service endpoint. `agent.yaml` derives it from `AZURE_AI_SEARCH_SERVICE_NAME` when provisioned by `azd` |
| `AZURE_SEARCH_INDEX_NAME` | No | Search index name. Defaults to `contoso-outdoors`; the index must already exist before the agent starts |
| `APPLICATIONINSIGHTS_CONNECTION_STRING` | No | Enables telemetry. Auto-injected in hosted containers; set manually for local development |

When deployed as a hosted agent, `FOUNDRY_PROJECT_ENDPOINT` and Application Insights are auto-injected by the platform. Authentication uses Managed Identity via `DefaultAzureCredential`. Locally, the sample loads a `.env` file via `DotNetEnv` if present.

## Running Locally

### Prerequisites

- .NET 10.0 SDK or later (`dotnet --version`)
- An Azure AI Foundry project with a model deployment (or let `azd provision` create one)
- An Azure AI Search service with an existing `contoso-outdoors` index, Entra ID authentication enabled, and data-plane RBAC for the user or managed identity that queries it

### Using `azd`

Start the agent locally with the `run` command — `azd` restores dependencies, sets environment variables, builds, and starts the agent:

```bash
azd ai agent run
```

The agent starts on `http://localhost:8088/`.

<details>
<summary><h3>Using the Foundry Toolkit VS Code Extension</h3></summary>

The [Foundry Toolkit VS Code extension](https://learn.microsoft.com/en-us/azure/foundry/agents/quickstarts/quickstart-hosted-agent?view=foundry&pivots=vscode) sets up a one-click **F5** debug experience for this sample.

1. Scaffold the project using the Foundry Toolkit extension: open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Create new Hosted Agent`. The extension automatically creates the VS Code debug configuration files and `.env`.
2. Edit `.env` and fill in the required environment variables (see [Environment Variables](#environment-variables) above for the full list).
3. Restore dependencies:
   ```bash
   dotnet restore
   ```
4. Press **F5** to start the agent in debug mode. The agent starts on `http://localhost:8088/`.

</details>
<details>
<summary><h3>Manual setup</h3></summary>

Set the environment variables from [Environment Variables](#environment-variables) (or place them in a `.env` file, which the sample loads via `DotNetEnv`), then:

```bash
dotnet run
```

The agent starts on `http://localhost:8088/`.

</details>

## Invoke

### Using azd

**Local:**

**Bash:**
```bash
azd ai agent invoke --local "What is your return policy?"
```

**PowerShell:**
```powershell
azd ai agent invoke --local "What is your return policy?"
```

**Test with curl:**

```bash
curl -sS -X POST http://localhost:8088/responses \
  -H "Content-Type: application/json" \
  -d '{"input": "What is your return policy?", "stream": false}' | jq .
```

<details>
<summary><h3>Using Foundry Toolkit VS Code Extension</h3></summary>

Open the **Agent Inspector** directly from the Foundry Toolkit extension to invoke the agent — no `curl` or CLI commands needed.

1. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Open Agent Inspector`.
2. The Inspector auto-connects to your running agent at `http://localhost:8088/`.
3. Type a message and send it. The Agent Inspector handles the Responses protocol and displays the reply inline.

> Multi-turn conversation is supported — the Inspector maintains session context across messages.

</details>

## Provisioning the Search Index

The agent expects the search index to exist and be populated before startup. `Program.cs` does not create or seed the index at runtime.

Required fields for the default `contoso-outdoors` index:

| Field | Type | Attributes |
|-------|------|------------|
| `id` | `Edm.String` | key, filterable |
| `content` | `Edm.String` | searchable |
| `sourceName` | `Edm.String` | filterable |
| `sourceLink` | `Edm.String` | none |

The sample content used by the previous README included three Contoso Outdoors documents: return policy, shipping guide, and TrailRunner tent care instructions. Seed equivalent documents before invoking prompts such as "What is your return policy?" or "How do I clean my tent?".

The script or user that creates and seeds the index needs `Search Index Data Contributor` on the search service. The hosted agent's managed identity needs `Search Index Data Reader` on the same scope. Subscription `Owner` or `Contributor` alone is not sufficient because Azure AI Search document APIs require data-plane roles.

Azure AI Search must accept Entra ID tokens. If an older service is API-key-only, enable AAD/API-key auth before using `DefaultAzureCredential`:

```bash
az search service update -g <resource-group> -n <search-service> \
  --auth-options aadOrApiKey --aad-auth-failure-mode http403
```

The `azd-ai-starter-basic` template provisions the Foundry project, chat model, Azure AI Search service, and project-scoped Search connection from the model/tool resources in `agent.manifest.yaml`. If your generated `azure.yaml` requires the starter's storage dependency, add a `storage` resource alongside `azure_ai_search` before `azd provision`.

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
azd ai agent invoke "What is your return policy?"
```

**PowerShell:**
```powershell
azd ai agent invoke "What is your return policy?"
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

**Deploy with `azd deploy`**, which uses ACR remote build and always produces images with the correct architecture.

If you choose to **build locally**, and your machine is **not `linux/amd64`** (for example, an Apple Silicon Mac), the image will **not be compatible with our service**, causing runtime failures.

**Fix for local builds:**

```bash
docker build --platform=linux/amd64 -t image .
```

This forces the image to be built for the required `amd64` architecture.
