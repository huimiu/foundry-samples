# Foundry Toolbox Agent (Responses Protocol)

An [Agent Framework](https://github.com/microsoft/agent-framework) agent hosted on Microsoft Foundry using the **Responses protocol**. It demonstrates how to connect an agent to a Microsoft Foundry Toolbox MCP endpoint so toolbox-managed tools can be discovered and invoked at runtime.

## How It Works

1. `main.py` creates a `DefaultAzureCredential` and token provider for `https://ai.azure.com/.default`
2. `resolve_toolbox_endpoint()` prefers `TOOLBOX_ENDPOINT`; if it is not set, it builds a toolbox MCP endpoint from `FOUNDRY_PROJECT_ENDPOINT` and `TOOLBOX_NAME`
3. The sample creates an authenticated `httpx.AsyncClient` with the `Foundry-Features: Toolboxes=V1Preview` header and passes it to `MCPStreamableHTTPTool`
4. `FoundryChatClient` connects to your Foundry model deployment, and the Agent Framework `Agent` receives the toolbox as its tool source
5. The agent is served by `ResponsesHostServer`, which exposes an OpenAI-compatible `POST /responses` endpoint on `http://localhost:8088/`

See [main.py](main.py) and [toolbox.yaml](toolbox.yaml) for the full implementation.

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `FOUNDRY_PROJECT_ENDPOINT` | Yes | Azure AI Foundry project endpoint URL. Auto-injected when hosted — only needed locally |
| `AZURE_AI_MODEL_DEPLOYMENT_NAME` | Yes | Model deployment name (e.g. `gpt-4o`). Declared in `agent.yaml` |
| `TOOLBOX_ENDPOINT` | Yes | Versioned MCP endpoint for the Foundry Toolbox this agent consumes. Declared in `agent.yaml` |

When deployed as a hosted agent, `FOUNDRY_PROJECT_ENDPOINT` is auto-injected by the platform — you only need to set `AZURE_AI_MODEL_DEPLOYMENT_NAME` and `TOOLBOX_ENDPOINT`. Authentication uses Managed Identity via `DefaultAzureCredential`.

## Running Locally

### Prerequisites

- Python 3.10+
- An Azure AI Foundry project with a model deployment (or let `azd provision` create one)
- A Foundry Toolbox created from [toolbox.yaml](toolbox.yaml), with its versioned MCP endpoint stored in `TOOLBOX_ENDPOINT`

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
azd ai agent invoke --local "What tools do you have available?"
```

**PowerShell:**
```powershell
azd ai agent invoke --local "What tools do you have available?"
```

**Test with curl:**

```bash
curl -sS -X POST http://localhost:8088/responses \
  -H "Content-Type: application/json" \
  -d '{"input": "What tools do you have available?", "stream": false}' | jq .
```

<details>
<summary><h3>Using Foundry Toolkit VS Code Extension</h3></summary>

Open the **Agent Inspector** directly from the Foundry Toolkit extension to invoke the agent — no `curl` or CLI commands needed.

1. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Open Agent Inspector`.
2. The Inspector auto-connects to your running agent at `http://localhost:8088/`.
3. Type a message and send it. The Agent Inspector handles the Responses protocol and displays the reply inline.

> Multi-turn conversation is supported — the Inspector maintains session context across messages.

</details>

## Foundry Toolbox Configuration

The bundled [toolbox.yaml](toolbox.yaml) defines two connectionless tools behind one endpoint:

- **Web search**, which grounds responses in real-time public web results
- **Microsoft Learn MCP server** (`https://learn.microsoft.com/api/mcp`), a public endpoint that grounds responses in official Microsoft documentation and requires no authentication

Create the toolbox once from the bundled file:

```bash
azd ai toolbox create my-toolbox --from-file ./toolbox.yaml
```

`azd ai toolbox create` prints the toolbox's versioned MCP endpoint. Copy that endpoint and store it in your azd environment so the agent connects to it:

```bash
azd env set TOOLBOX_ENDPOINT "https://<account>.services.ai.azure.com/api/projects/<project>/toolboxes/my-toolbox/versions/1/mcp?api-version=v1"
```

Use `azd ai toolbox list`, `azd ai toolbox show my-toolbox`, and `azd ai toolbox version list my-toolbox` to inspect the toolbox. To stage incremental changes safely, use `azd ai toolbox connection add/remove` and `azd ai toolbox skill add/list/remove`, then promote a version with `azd ai toolbox publish my-toolbox <version>` when ready.

You can also create a Foundry Toolbox in code using the [Foundry Toolbox CRUD sample](https://github.com/Azure/azure-sdk-for-python/blob/main/sdk/ai/azure-ai-projects/samples/hosted_agents/sample_toolboxes_crud.py) or in the Foundry portal. For connection-backed scenarios, see the [`langgraph-toolbox`](../../../bring-your-own/responses/langgraph-toolbox/README.md), [`langgraph-toolbox-user-identity`](../../../bring-your-own/responses/langgraph-toolbox-user-identity/README.md), and [supported toolbox scenarios](https://github.com/microsoft-foundry/foundry-samples/blob/main/samples/python/hosted-agents/SUPPORTED_TOOLBOX_SCENARIOS.md).

## Next Steps

- [Quickstart: Create a hosted agent](https://learn.microsoft.com/en-us/azure/foundry/agents/quickstarts/quickstart-hosted-agent) — end-to-end walkthrough using `azd`
- [Tool catalog](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/tool-catalog) — browse available tools to extend your agent
- [Manage hosted agents](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/manage-hosted-agent) — monitor and manage deployed agents
- [Basic agent](../01-basic/) — minimal agent with no tools
- [Add local tools](../02-tools/) — sample with locally defined Python tool functions
- [Build multi-agent workflows](../05-workflows/) — sample with chained agent pipelines

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
azd ai agent invoke "What tools do you have available?"
```

**PowerShell:**
```powershell
azd ai agent invoke "What tools do you have available?"
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
