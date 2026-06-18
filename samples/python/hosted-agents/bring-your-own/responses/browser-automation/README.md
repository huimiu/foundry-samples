# Browser Automation Agent (Responses Protocol)

A Bring Your Own hosted agent built with [azure-ai-agentserver-responses](https://pypi.org/project/azure-ai-agentserver-responses/) that automates remote browsers through Playwright CLI and an Azure AI Foundry Toolbox MCP endpoint. It uses the **Responses protocol** and streams browser session events, optional tool logs, and final answers back to the caller.

## How It Works

1. [main.py](main.py) creates a `ResponsesAgentServerHost`, which exposes `POST /responses`; the handler reads the request body `input` field and retrieves prior turns with `context.get_history()`.
2. The agent calls a Foundry model through `AIProjectClient.get_openai_client().responses.create(...)`, using the tool definitions and system prompt in [utils/constants.py](utils/constants.py).
3. The first `run_browser` tool call lazily creates a default browser session: [utils/toolbox.py](utils/toolbox.py) calls the toolbox `create_session` tool, then [utils/browser.py](utils/browser.py) attaches `playwright-cli` to the returned CDP URL.
4. Tool calls such as `run_browser`, `create_session`, `end_session`, `run_parallel`, `list_sessions`, and `load_skill` are executed and fed back to the model until it returns a final answer.
5. The handler streams `ResponseEventStream` lifecycle events. It always emits live-view session events, and emits tool action logs when the user starts the prompt with `/verbose`.

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `FOUNDRY_PROJECT_ENDPOINT` | Yes, for local runs | Azure AI Foundry project endpoint URL. Auto-injected when hosted — only needed locally |
| `AZURE_AI_MODEL_DEPLOYMENT_NAME` | Yes | Model deployment name (for example, `gpt-4.1`). Declared in `agent.yaml` and `agent.manifest.yaml` |
| `TOOLBOX_BROWSER_AUTOMATION_TOOLS_MCP_ENDPOINT` | Yes, when hosted with the toolbox resource | Browser automation toolbox MCP endpoint generated from the `browser-automation-tools` toolbox declared in `agent.yaml` |
| `PLAYWRIGHT_SERVICE_URL` | Yes, for provisioning the Playwright connection | Browser WebSocket endpoint for the Azure Playwright workspace |
| `PLAYWRIGHT_SERVICE_RESOURCE_ID` | Yes, for provisioning the Playwright connection | Azure resource ID of the Playwright workspace |
| `PLAYWRIGHT_SERVICE_ACCESS_TOKEN` | Yes, secret | Access token for the Azure Playwright workspace connection |
| `BROWSER_TIMEOUT_SECONDS` | No | Timeout for each `playwright-cli` command. Defaults to `180` seconds |

When deployed as a hosted agent, `FOUNDRY_PROJECT_ENDPOINT` is auto-injected by the platform. Authentication uses Managed Identity via `DefaultAzureCredential`; locally, use `az login` or another supported credential source.

## Running Locally

### Prerequisites

- Python 3.10+
- Azure CLI installed and authenticated (`az login`) or another `DefaultAzureCredential` source
- An Azure AI Foundry project with a model deployment (or let `azd provision` create one)
- An Azure Playwright workspace, its browser WebSocket URL, resource ID, and access token
- Playwright CLI available locally (`npm install -g @playwright/cli@latest`)

### Using `azd` (Recommended)

Create a local `.env` file from the sample template and fill in the required values:

```bash
cp .env.example .env  # skip if .env already exists
# Edit .env — see Environment Variables above
```

If you plan to deploy with `azd`, also add any secrets to your azd environment so they can be injected into the hosted agent:

```bash
azd env set PLAYWRIGHT_SERVICE_ACCESS_TOKEN="..."
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
azd ai agent invoke --local "Go to https://example.com and tell me the page title"
```

**PowerShell:**
```powershell
azd ai agent invoke --local "Go to https://example.com and tell me the page title"
```

**Test with curl:**

```bash
curl -sS -X POST http://localhost:8088/responses \
  -H "Content-Type: application/json" \
  -d '{"input": "Go to https://example.com and tell me the page title", "stream": false}' | jq .
```

<details>
<summary><h3>Using Foundry Toolkit VS Code Extension</h3></summary>

Open the **Agent Inspector** directly from the Foundry Toolkit extension to invoke the agent — no `curl` or CLI commands needed.

1. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Open Agent Inspector`.
2. The Inspector auto-connects to your running agent at `http://localhost:8088/`.
3. Type a message and send it. The Agent Inspector handles the protocol and displays the response inline.

> Multi-turn conversation is supported — the Inspector maintains session context across messages.

</details>

## Playwright Toolbox Setup

The [agent.manifest.yaml](agent.manifest.yaml) creates a `browserautomation` project connection and a `browser-automation-tools` toolbox that exposes the `browser_automation_preview` tool. Set the non-secret Playwright values and the access-token secret before provisioning or deploying:

```bash
azd env set PLAYWRIGHT_SERVICE_URL "wss://<region>.api.playwright.microsoft.com/playwrightworkspaces/<workspace-id>/browsers"
azd env set PLAYWRIGHT_SERVICE_RESOURCE_ID "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.LoadTestService/playwrightWorkspaces/<workspace-name>"
azd env set PLAYWRIGHT_SERVICE_ACCESS_TOKEN "<playwright-workspace-access-token>"
```

For local development, install Playwright CLI and keep the same values in `.env`:

```bash
npm install -g @playwright/cli@latest
```

The agent never returns CDP URLs or access tokens. It may return the toolbox `live_view_url` so you can watch a browser session while the model works.

## Browser Tools and Skills

| Tool | Description |
|------|-------------|
| `run_browser` | Runs a `playwright-cli` command. A default session is created automatically on first use |
| `create_session` | Creates an additional named browser session for parallel work |
| `end_session` | Ends one session, or `all`, immediately |
| `run_parallel` | Runs browser commands across multiple sessions concurrently |
| `list_sessions` | Lists active sessions and live-view URLs |
| `load_skill` | Loads a markdown skill from the `skills/` directory |

The included skills are:

- `form-filler` — step-by-step form filling with date picker handling and multi-page navigation
- `web-scraper` — data extraction using JavaScript `eval`, pagination, and reporting guidance

## Streaming and Verbose Mode

The canonical invoke uses a non-streaming curl request. To observe live session events and optional tool logs, send `stream: true`; prefix the prompt with `/verbose` to emit each tool action:

```bash
curl -N -X POST http://localhost:8088/responses \
  -H "Content-Type: application/json" \
  -d '{"input": "/verbose Go to https://example.com and tell me the page title", "stream": true}'
```

## Multi-Session Examples

Parallel research can create multiple browser sessions and run commands across them:

```text
User: Compare pricing on aws.amazon.com and azure.microsoft.com side by side

Agent:
  → create_session("aws")
  → create_session("azure")
  → run_parallel([
      {session: "aws", command: "goto", args: ["https://aws.amazon.com/pricing/"]},
      {session: "azure", command: "goto", args: ["https://azure.microsoft.com/pricing/"]}
    ])
  → run_parallel([
      {session: "aws", command: "snapshot"},
      {session: "azure", command: "snapshot"}
    ])
  → Responds with a comparison
```

Human-in-the-loop flows can keep a session open while the user provides missing values, such as one-time passcodes, and `end_session` can close a named session or all sessions when the task is done.

## Customization

- Edit [utils/constants.py](utils/constants.py) to adjust the system prompt or tool definitions.
- Add new markdown workflows under `skills/` and load them with the `load_skill` tool.
- Update [utils/browser.py](utils/browser.py) to add or restrict allowed `playwright-cli` commands.
- Adjust `agent.yaml` CPU and memory for heavier browser workloads.

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
azd ai agent invoke "Go to https://example.com and tell me the page title"
```

**PowerShell:**
```powershell
azd ai agent invoke "Go to https://example.com and tell me the page title"
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

### `playwright-cli` is not found locally

Install Playwright CLI with `npm install -g @playwright/cli@latest`, then restart the agent so [utils/browser.py](utils/browser.py) can find `playwright-cli` on `PATH`.
