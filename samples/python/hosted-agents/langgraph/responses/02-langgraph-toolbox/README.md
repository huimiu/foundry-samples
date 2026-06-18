# LangGraph Toolbox Agent (Responses Protocol)

A [LangGraph](https://langchain-ai.github.io/langgraph/) ReAct agent hosted on Microsoft Foundry using the **Responses protocol**. It loads tools from a Foundry Toolbox defined by `toolbox.yaml`, including `web_search` and the public Microsoft Learn MCP server.

## How It Works

1. `main.py` loads local settings from `.env`, then builds a Foundry-backed `ChatOpenAI` model from `FOUNDRY_PROJECT_ENDPOINT` and `AZURE_AI_MODEL_DEPLOYMENT_NAME` using `DefaultAzureCredential`.
2. `_LazyToolboxHostServer` initially starts with an empty LangGraph agent so readiness can succeed quickly, then lazily loads real tools on the first request.
3. `AzureAIProjectToolbox(toolbox_name=os.environ["TOOLBOX_NAME"])` opens the Foundry Toolbox MCP endpoint, authenticates with `DefaultAzureCredential`, and returns LangChain `BaseTool` instances.
4. Loaded tool schemas are sanitized before use, and each tool receives a `handle_tool_error` callback that turns OAuth consent errors into friendly tool messages.
5. `langchain.agents.create_agent(...)` compiles a LangGraph ReAct agent with the toolbox tools and a system prompt that asks the model to cite tool-provided sources.
6. `ResponsesHostServer` exposes the OpenAI-compatible `POST /responses` endpoint on `http://localhost:8088/`.

See [main.py](main.py) and [toolbox.yaml](toolbox.yaml) for the full implementation.

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `FOUNDRY_PROJECT_ENDPOINT` | Yes | Azure AI Foundry project endpoint URL. Auto-injected when hosted — only needed locally |
| `AZURE_AI_MODEL_DEPLOYMENT_NAME` | Yes | Model deployment name (e.g. `gpt-4o`). Declared in `agent.yaml` |
| `TOOLBOX_NAME` | Yes | Name of the Foundry Toolbox to load. Defaults to `my-toolbox` in `.env.example` and `agent.yaml` |

When deployed as a hosted agent, `FOUNDRY_PROJECT_ENDPOINT` is auto-injected by the platform — you only need to set `AZURE_AI_MODEL_DEPLOYMENT_NAME` and ensure `TOOLBOX_NAME` points at an existing toolbox. Authentication uses Managed Identity via `DefaultAzureCredential`.

## Running Locally

### Prerequisites

- Python 3.12+
- An Azure AI Foundry project with a model deployment (or let `azd provision` create one)
- A Foundry Toolbox named `my-toolbox` created from [toolbox.yaml](toolbox.yaml), or `azd provision` against [agent.manifest.yaml](agent.manifest.yaml) to create it automatically

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
azd ai agent invoke --local "How do I create a Foundry project with the Azure CLI?"
```

**PowerShell:**
```powershell
azd ai agent invoke --local "How do I create a Foundry project with the Azure CLI?"
```

**Test with curl:**

```bash
curl -sS -X POST http://localhost:8088/responses \
  -H "Content-Type: application/json" \
  -d '{"input": "How do I create a Foundry project with the Azure CLI?", "stream": false}' | jq .
```

<details>
<summary><h3>Using Foundry Toolkit VS Code Extension</h3></summary>

Open the **Agent Inspector** directly from the Foundry Toolkit extension to invoke the agent — no `curl` or CLI commands needed.

1. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Open Agent Inspector`.
2. The Inspector auto-connects to your running agent at `http://localhost:8088/`.
3. Type a message and send it. The Agent Inspector handles the protocol and displays the response inline.

> Multi-turn conversation is supported — the Inspector maintains session context across messages.

</details>

## Foundry Toolbox Configuration

The bundled [toolbox.yaml](toolbox.yaml) defines two tools behind a single Foundry Toolbox endpoint:

| Tool | Type | Purpose |
|------|------|---------|
| `web_search` | `web_search` | Searches the public web for current information |
| `mslearn` | `mcp` | Connects to the public Microsoft Learn MCP server at `https://learn.microsoft.com/api/mcp` |

Create the toolbox manually with:

```bash
azd ai toolbox create my-toolbox --from-file ./toolbox.yaml
```

If you provision from this sample's [agent.manifest.yaml](agent.manifest.yaml), the `my-toolbox` toolbox is created automatically. `AzureAIProjectToolbox` identifies the toolbox by name and uses its current default version, so publishing a new default toolbox version makes the agent pick it up automatically.

## OAuth Consent and Tool Schema Handling

If a toolbox connection needs additional consent, the Foundry MCP gateway raises MCP error code `-32006` with a URL on `consent.azure-apim.net`. The sample detects that error and returns a tool message like:

```
OAuth consent required. Open this URL in a browser to authorize the toolbox connection, then retry the request: https://consent.azure-apim.net/...
```

The sample also patches malformed MCP schemas that are missing `properties` on `object`-type schemas before LangGraph passes them to the model.

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
azd ai agent invoke "How do I create a Foundry project with the Azure CLI?"
```

**PowerShell:**
```powershell
azd ai agent invoke "How do I create a Foundry project with the Azure CLI?"
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

### Agent starts but returns no tools

Check that the toolbox exists in your Foundry project and that `TOOLBOX_NAME` matches its name. `my-toolbox` is the default name provisioned from [agent.manifest.yaml](agent.manifest.yaml).

### OAuth consent required

Open the consent URL surfaced by the agent, complete the consent flow, then retry the original request.

### Tool schemas rejected by OpenAI

The sample sanitizes missing `properties` on MCP `object` schemas at load time. If `400 Invalid tool schema` errors continue, inspect the raw schema returned by your MCP server for additional shape issues.
