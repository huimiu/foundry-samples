**IMPORTANT!** All samples and other resources made available in this GitHub repository ("samples") are designed to assist in accelerating development of agents, solutions, and agent workflows for various scenarios. Review all provided resources and carefully test output behavior in the context of your use case. AI responses may be inaccurate and AI actions should be monitored with human oversight.

# Toolbox Agent (Invocations Protocol)

A Bring Your Own hosted agent using the Microsoft Foundry **Invocations protocol** with Azure AI Foundry Toolbox MCP integration. It discovers toolbox tools over MCP, lets the Foundry model call them in an agentic loop, and streams the final answer as SSE.

## How It Works

1. [main.py](main.py) loads `.env`, reads `FOUNDRY_PROJECT_ENDPOINT`, `AZURE_AI_MODEL_DEPLOYMENT_NAME`, and resolves the toolbox MCP URL from `TOOLBOX_ENDPOINT` or `TOOLBOX_NAME`
2. `_McpToolboxClient` authenticates with `DefaultAzureCredential`, sends MCP `initialize`, and calls `tools/list`; this happens lazily on the first request so the container can pass health checks before the toolbox is reachable
3. MCP tool schemas are converted to Responses API function definitions
4. `InvocationAgentServerHost` exposes `POST /invocations`
5. The handler accepts a JSON object with `message`, `input`, or `query`, or a plain-text body, and stores conversation history in memory by `agent_session_id`
6. The agentic loop calls the model, executes requested tools through MCP `tools/call`, feeds results back to the model for up to 10 rounds, and streams a final `token` event followed by a `done` event

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `FOUNDRY_PROJECT_ENDPOINT` | Yes | Azure AI Foundry project endpoint URL. Auto-injected when hosted — only needed locally |
| `AZURE_AI_MODEL_DEPLOYMENT_NAME` | Yes | Model deployment name. `agent.yaml` defaults to `gpt-4.1` |
| `TOOLBOX_ENDPOINT` | Yes, unless `TOOLBOX_NAME` is set | Full toolbox MCP endpoint URL. Declared in `agent.yaml` and `.env.example` |
| `TOOLBOX_NAME` | No | Alternative to `TOOLBOX_ENDPOINT`; the agent reads `TOOLBOX_<NAME>_MCP_ENDPOINT` if injected, or constructs a latest-version endpoint from `FOUNDRY_PROJECT_ENDPOINT` |
| `FOUNDRY_AGENT_TOOLBOX_FEATURES` | No | Feature flag header value for toolbox calls. Defaults to `Toolboxes=V1Preview` |
| `APPLICATIONINSIGHTS_CONNECTION_STRING` | No | Application Insights connection string. Auto-injected when hosted — only needed locally for telemetry |

Set `TOOLBOX_ENDPOINT` to either the latest-version endpoint or a version-pinned endpoint:

```
https://<account>.services.ai.azure.com/api/projects/<project>/toolboxes/<toolbox-name>/mcp?api-version=v1
https://<account>.services.ai.azure.com/api/projects/<project>/toolboxes/<toolbox-name>/versions/<version>/mcp?api-version=v1
```

## Running Locally

### Prerequisites

- Python 3.10+
- An Azure AI Foundry project with a model deployment (or let `azd provision` create one)
- A Foundry toolbox created from [toolbox.yaml](toolbox.yaml), or another toolbox MCP endpoint you can access
- Azure credentials with access to the Foundry project and toolbox (for example, `az login`)

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
azd ai agent invoke --local '{"message": "Search the web for Azure AI Foundry news"}'
```

**PowerShell:**
```powershell
azd ai agent invoke --local '{\"message\": \"Search the web for Azure AI Foundry news\"}'
```

**Test with curl:**

```bash
# Turn 1
curl -sS -N -X POST "http://localhost:8088/invocations?agent_session_id=chat-001" \
  -H "Content-Type: application/json" \
  -d '{"message": "Search the web for Azure AI Foundry news"}'

# Turn 2 (same session)
curl -sS -N -X POST "http://localhost:8088/invocations?agent_session_id=chat-001" \
  -H "Content-Type: application/json" \
  -d '{"message": "Tell me more about the first result"}'
```

**SSE Event Format:**

The agent streams the final tool-grounded answer and a completion event:

```
data: {"type": "token", "content": "..."}

data: {"type": "done", "invocation_id": "...", "session_id": "...", "full_text": "..."}
```

<details>
<summary><h3>Using Foundry Toolkit VS Code Extension</h3></summary>

Open the **Agent Inspector** directly from the Foundry Toolkit extension to invoke the agent — no `curl` or CLI commands needed.

1. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Open Agent Inspector`.
2. The Inspector auto-connects to your running agent at `http://localhost:8088/`.
3. Type a message and send it. The Agent Inspector handles the protocol and displays the response inline.

> Multi-turn conversation is supported — the Inspector maintains session context across messages.

</details>

## Creating the Toolbox

The bundled [toolbox.yaml](toolbox.yaml) defines a `web_search` tool plus the public Microsoft Learn MCP server:

```bash
azd ai toolbox create my-toolbox --from-file ./toolbox.yaml
```

The command prints the toolbox MCP endpoint. Copy it into your local `.env` and, for deployment, store it in the `azd` environment:

```bash
azd env set TOOLBOX_ENDPOINT "<endpoint-from-output>"
```

Manage toolboxes with `azd ai toolbox list`, `azd ai toolbox show my-toolbox`, `azd ai toolbox version list my-toolbox`, and `azd ai toolbox delete my-toolbox --force`. To stage incremental changes, use `azd ai toolbox connection add/remove` and `azd ai toolbox skill add/list/remove`; promote a version with `azd ai toolbox publish my-toolbox <version>`.

To attach tools that need credentials (MCP servers with API keys or OAuth, Azure AI Search, Bing Custom Search, and more), create a project connection with `azd ai connection create` and reference it from `toolbox.yaml`.

## Supported Scenarios

The broader toolbox sample set documents these scenarios:

<details>
<summary><strong>View supported toolbox scenarios</strong></summary>

1. **Web Search** — Bing web search
2. **File Search** — Vector store RAG search
3. **Code Interpreter** — Python code execution
4. **MCP Key-Auth (GitHub)** — GitHub MCP with PAT
5. **MCP No-Auth** — Public MCP servers
6. **MCP OAuth (Managed)** — Foundry-managed OAuth
7. **MCP OAuth (Custom)** — Bring-your-own OAuth app
8. **MCP Agent Identity** — Entra ID agent identity
9. **Azure AI Search** — Search index queries
10. **A2A (Agent-to-Agent)** — Remote agent delegation
11. **Bing Custom Search** — Scoped web search
12. **OpenAPI Key-Auth** — REST API integration
13. **MCP OAuth (Entra Passthrough)** — User identity delegation
14. **Multi-Tool Toolbox** — Web search plus GitHub MCP combined

See [`samples/python/toolbox/azd/README.md`](../../../../toolbox/azd/README.md#supported-scenarios) for full scenario details and manifest examples.

</details>

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
azd ai agent invoke '{"message": "Search the web for Azure AI Foundry news"}'
```

**PowerShell:**
```powershell
azd ai agent invoke '{\"message\": \"Search the web for Azure AI Foundry news\"}'
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