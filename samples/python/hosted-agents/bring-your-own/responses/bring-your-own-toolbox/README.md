<!-- Begin standard disclaimer — do not modify -->
**IMPORTANT!** All samples and other resources made available in this GitHub repository ("samples") are designed to assist in accelerating development of agents, solutions, and agent workflows for various scenarios. Review all provided resources and carefully test output behavior in the context of your use case. AI responses may be inaccurate and AI actions should be monitored with human oversight. Learn more in the transparency documents for [Agent Service](https://learn.microsoft.com/en-us/azure/ai-foundry/responsible-ai/agents/transparency-note) and [Agent Framework](https://github.com/microsoft/agent-framework/blob/main/TRANSPARENCY_FAQ.md).

Agents, solutions, or other output you create may be subject to legal and regulatory requirements, may require licenses, or may not be suitable for all industries, scenarios, or use cases. By using any sample, you are acknowledging that any output created using those samples are solely your responsibility, and that you will comply with all applicable laws, regulations, and relevant safety standards, terms of service, and codes of conduct.

Third-party samples contained in this folder are subject to their own designated terms, and they have not been tested or verified by Microsoft or its affiliates.

Microsoft has no responsibility to you or others with respect to any of these samples or any resulting output.
<!-- End standard disclaimer -->

# Foundry Toolbox Agent (Responses Protocol)

A Bring Your Own hosted agent built with [azure-ai-agentserver-responses](https://pypi.org/project/azure-ai-agentserver-responses/) and the Foundry SDK. It uses the **Responses protocol** and an Azure AI Foundry Toolbox MCP endpoint to discover tools, let the model call them, and return the final answer.

## How It Works

1. [main.py](main.py) loads local configuration, creates an `AIProjectClient` with `DefaultAzureCredential`, and gets a Foundry OpenAI Responses client for `AZURE_AI_MODEL_DEPLOYMENT_NAME`.
2. The toolbox MCP endpoint is resolved from `TOOLBOX_ENDPOINT`; as a fallback, the code can construct a latest-version endpoint from `TOOLBOX_NAME` and `FOUNDRY_PROJECT_ENDPOINT`.
3. On the first request, `_McpToolboxClient` initializes the MCP session, sends the `notifications/initialized` message, calls `tools/list`, and converts discovered MCP tools into Responses API function definitions.
4. The `POST /responses` handler extracts the request body `input` field, retrieves recent conversation history with `context.get_history()`, and sends both to the model with the discovered tools.
5. If the model returns `function_call` items, the agent calls the toolbox with MCP `tools/call`, appends each `function_call_output`, and loops until the model produces final text or the max tool rounds limit is reached.
6. The final answer is streamed through `ResponseEventStream` using Responses protocol lifecycle events.

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `FOUNDRY_PROJECT_ENDPOINT` | Yes, for local runs | Azure AI Foundry project endpoint URL. Auto-injected when hosted — only needed locally |
| `AZURE_AI_MODEL_DEPLOYMENT_NAME` | Yes | Model deployment name (for example, `gpt-4.1`). Declared in `agent.yaml` and `agent.manifest.yaml` |
| `TOOLBOX_ENDPOINT` | Yes, for toolbox use | Full toolbox MCP endpoint URL. Create the toolbox from [toolbox.yaml](toolbox.yaml), then copy the versioned endpoint from the command output |
| `TOOLBOX_NAME` | No | Optional fallback name. If `TOOLBOX_ENDPOINT` is unset, the agent constructs a latest-version toolbox endpoint from this name and `FOUNDRY_PROJECT_ENDPOINT` |
| `FOUNDRY_AGENT_TOOLBOX_FEATURES` | No | Toolbox feature header value. Defaults to `Toolboxes=V1Preview` |
| `APPLICATIONINSIGHTS_CONNECTION_STRING` | No | Enables local telemetry. Auto-injected in hosted containers when monitoring is configured |

When deployed as a hosted agent, `FOUNDRY_PROJECT_ENDPOINT` is auto-injected by the platform. Authentication uses Managed Identity via `DefaultAzureCredential`; locally, use `az login` or another supported credential source.

## Running Locally

### Prerequisites

- Python 3.10+
- Azure CLI installed and authenticated (`az login`) or another `DefaultAzureCredential` source
- An Azure AI Foundry project with a model deployment (or let `azd provision` create one)
- A Foundry Toolbox MCP endpoint. The bundled [toolbox.yaml](toolbox.yaml) creates `web_search` and public Microsoft Learn MCP tools

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
azd ai agent invoke --local "Search the web for Azure AI Foundry news"
```

**PowerShell:**
```powershell
azd ai agent invoke --local "Search the web for Azure AI Foundry news"
```

**Test with curl:**

```bash
curl -sS -X POST http://localhost:8088/responses \
  -H "Content-Type: application/json" \
  -d '{"input": "Search the web for Azure AI Foundry news", "stream": false}' | jq .
```

<details>
<summary><h3>Using Foundry Toolkit VS Code Extension</h3></summary>

Open the **Agent Inspector** directly from the Foundry Toolkit extension to invoke the agent — no `curl` or CLI commands needed.

1. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Open Agent Inspector`.
2. The Inspector auto-connects to your running agent at `http://localhost:8088/`.
3. Type a message and send it. The Agent Inspector handles the protocol and displays the response inline.

> Multi-turn conversation is supported — the Inspector maintains session context across messages.

</details>

## Create the Toolbox

The sample [toolbox.yaml](toolbox.yaml) defines two public tools behind one MCP endpoint:

| Tool | Description |
|------|-------------|
| `web_search` | Searches the public web for current information |
| `mslearn` | Connects to the public Microsoft Learn MCP server |

Create the toolbox once from this file:

```bash
azd ai toolbox create my-toolbox --from-file ./toolbox.yaml
```

`azd ai toolbox create` prints the toolbox MCP endpoint. Copy that endpoint into `.env` for local runs and into your azd environment for deployments:

```bash
azd env set TOOLBOX_ENDPOINT "<endpoint-from-output>"
```

The endpoint can be latest-version or pinned to a specific version:

```text
https://<account>.services.ai.azure.com/api/projects/<project>/toolboxes/<toolbox-name>/mcp?api-version=v1
https://<account>.services.ai.azure.com/api/projects/<project>/toolboxes/<toolbox-name>/versions/<version>/mcp?api-version=v1
```

To stage toolbox changes safely, use `azd ai toolbox connection add/remove` and `azd ai toolbox skill add/list/remove`. Each command creates a new toolbox version; promote one with `azd ai toolbox publish my-toolbox <version>` when you are ready.

## MCP Configuration

The MCP client sends `initialize`, `notifications/initialized`, `tools/list`, and `tools/call` over HTTP with a bearer token for `https://ai.azure.com/.default`. If `TOOLBOX_ENDPOINT` is set, it is used directly. If it is not set and `TOOLBOX_NAME` is set, the agent first checks the azd-injected `TOOLBOX_<NAME>_MCP_ENDPOINT` variable and then falls back to a latest-version endpoint derived from `FOUNDRY_PROJECT_ENDPOINT`.

To attach tools that need credentials, create a project connection with `azd ai connection create` and reference it from `toolbox.yaml`; do not hard-code secrets in the README or source files.

## Streaming Responses

The canonical invoke uses `stream: false`. To watch Responses protocol SSE events for a toolbox-enabled answer, send `stream: true`:

```bash
curl -sS -N -X POST http://localhost:8088/responses \
  -H "Content-Type: application/json" \
  -d '{"input": "What is Azure AI Foundry?", "stream": true}'
```

## Supported Scenarios

The toolbox pattern can be adapted to the scenarios documented in [`samples/python/toolbox/azd/README.md`](../../../../toolbox/azd/README.md#supported-scenarios).

<details>
<summary><strong>View supported toolbox scenario categories</strong></summary>

1. **Web Search** — Bing web search
2. **File Search** — vector store RAG search
3. **Code Interpreter** — Python code execution
4. **MCP Key-Auth** — MCP servers with API keys
5. **MCP No-Auth** — public MCP servers
6. **MCP OAuth (Managed)** — Foundry-managed OAuth
7. **MCP OAuth (Custom)** — bring-your-own OAuth app
8. **MCP Agent Identity** — Entra ID agent identity
9. **Azure AI Search** — search index queries
10. **A2A (Agent-to-Agent)** — remote agent delegation
11. **Bing Custom Search** — scoped web search
12. **OpenAPI Key-Auth** — REST API integration
13. **MCP OAuth (Entra Passthrough)** — user identity delegation
14. **Multi-Tool Toolbox** — combined tools behind one endpoint

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
azd ai agent invoke "Search the web for Azure AI Foundry news"
```

**PowerShell:**
```powershell
azd ai agent invoke "Search the web for Azure AI Foundry news"
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
