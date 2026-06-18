# Browser Automation Agent (Responses Protocol)

An [Agent Framework](https://github.com/microsoft/agent-framework) hosted browser automation agent using Foundry Toolbox, Azure Playwright Service, Playwright CLI, and the **Responses protocol**. The agent creates remote browser sessions through a Toolbox MCP endpoint, runs Playwright CLI commands against the browser, and can return a live view link for human-in-the-loop steps.

## How It Works

1. `main.py` loads `.env`, builds `AgentSettings`, and starts a `ResponsesHostServer`
2. `utils/agent_factory.py` creates a `FoundryChatClient` with a scoped Azure credential for `https://ai.azure.com/.default`
3. The agent reads browser lifecycle and safety instructions from [prompts/base.md](prompts/base.md) and loads skills from [skills/](skills/) plus Playwright CLI-installed skills when present
4. `make_toolbox_mcp_tool()` connects to `<FOUNDRY_PROJECT_ENDPOINT>/toolboxes/<TOOLBOX_NAME>/mcp?api-version=v1` and exposes the Toolbox `create_session` capability
5. When `create_session` returns a CDP URL and live view URL, `utils/tools.py` stores them server-side so the model never has to echo secrets or long URLs
6. `run_playwright_cli`, `get_live_view_url`, and `close_browser_session` run Playwright CLI commands, inject live view links, and clean up browser sessions
7. Middleware logs tool usage with sensitive values redacted and injects the live browser URL into Responses output
8. The agent starts on `http://localhost:8088/`

See [main.py](main.py), [utils/agent_factory.py](utils/agent_factory.py), [utils/tools.py](utils/tools.py), and [docs/sample-structure.md](docs/sample-structure.md) for the full implementation.

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `FOUNDRY_PROJECT_ENDPOINT` | Yes | Azure AI Foundry project endpoint URL. Auto-injected when hosted — only needed locally |
| `AZURE_AI_MODEL_DEPLOYMENT_NAME` | Yes | Model deployment name (e.g. `gpt-4.1`). Declared in `agent.yaml` |
| `TOOLBOX_NAME` | No | Foundry Toolbox name. Defaults to `browser-automation-tools` |
| `BROWSER_AGENT_PLAYWRIGHT_CLI_TIMEOUT_SECONDS` | No | Timeout for each Playwright CLI command. Defaults to `180` |
| `BROWSER_AGENT_MCP_TIMEOUT_SECONDS` | No | Timeout for Toolbox MCP calls. Defaults to `120` |

When deployed as a hosted agent, `FOUNDRY_PROJECT_ENDPOINT` is auto-injected by the platform. The default `TOOLBOX_NAME` is declared in `agent.yaml`. Authentication uses Managed Identity via `DefaultAzureCredential`.

## Running Locally

### Prerequisites

- Python 3.11+
- An Azure AI Foundry project with a model deployment (or let `azd provision` create one)
- A compatible Foundry Toolbox containing the `browser_automation_preview` tool, or an Azure Playwright workspace to provision one from this sample
- Node.js/npm with `@playwright/cli` installed for local browser commands; the Dockerfile installs this automatically for hosted/container runs
- Access to an Azure Playwright workspace. If using sample provisioning, you also need its WebSocket URL, Azure resource ID, and access token

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
azd ai agent invoke --local "Open https://example.com and report the page title."
```

**PowerShell:**
```powershell
azd ai agent invoke --local "Open https://example.com and report the page title."
```

**Test with curl:**

```bash
curl -sS -X POST http://localhost:8088/responses \
  -H "Content-Type: application/json" \
  -d '{"input": "Open https://example.com and report the page title.", "stream": false}' | jq .
```

<details>
<summary><h3>Using Foundry Toolkit VS Code Extension</h3></summary>

Open the **Agent Inspector** directly from the Foundry Toolkit extension to invoke the agent — no `curl` or CLI commands needed.

1. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Open Agent Inspector`.
2. The Inspector auto-connects to your running agent at `http://localhost:8088/`.
3. Type a message and send it. The Agent Inspector handles the protocol and displays the response inline.

> Multi-turn conversation is supported — the Inspector maintains session context across messages.

</details>

## Browser Automation Configuration

The runtime Toolbox endpoint is derived from the Foundry project endpoint and toolbox name:

```text
<FOUNDRY_PROJECT_ENDPOINT>/toolboxes/<TOOLBOX_NAME>/mcp?api-version=v1
```

The default toolbox name is `browser-automation-tools`. If your Foundry project already has a compatible toolbox, set:

```bash
azd env set TOOLBOX_NAME "<your-toolbox-name>"
```

If you want this sample to provision the Playwright connection and default toolbox, set the Playwright workspace values in your `azd` environment before `azd provision`:

```bash
azd env set PLAYWRIGHT_SERVICE_URL "wss://<region>.api.playwright.microsoft.com/playwrightworkspaces/<workspace-id>/browsers"
azd env set PLAYWRIGHT_SERVICE_RESOURCE_ID "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.LoadTestService/playwrightWorkspaces/<workspace-name>"
azd env set PLAYWRIGHT_SERVICE_ACCESS_TOKEN "<playwright-workspace-access-token>"
```

`PLAYWRIGHT_SERVICE_ACCESS_TOKEN` is a secret provisioning parameter in [agent.manifest.yaml](agent.manifest.yaml). It is stored in the Foundry project connection and is not read by the Python agent process at runtime.

## Adding Tools and Skills

- Update [prompts/base.md](prompts/base.md) for browser lifecycle, safety, extraction, and form-filling behavior
- Add repeatable procedural guidance under [skills/](skills/) as `SKILL.md` files
- Add new Agent Framework tools in [utils/tools.py](utils/tools.py), then register them in [utils/agent_factory.py](utils/agent_factory.py)
- Review [docs/sample-structure.md](docs/sample-structure.md) for the sample's layering and design rationale

The default hosted resources (`cpu: "0.25"`, `memory: "0.5Gi"`) are intentionally small. Increase them in [agent.yaml](agent.yaml) for long scraping sessions or heavier browser workflows.

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
azd ai agent invoke "Open https://example.com and report the page title."
```

**PowerShell:**
```powershell
azd ai agent invoke "Open https://example.com and report the page title."
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

### Toolbox MCP calls fail

Confirm `TOOLBOX_NAME` points to a Foundry Toolbox in the same project and that it contains the `browser_automation_preview` tool with a valid Playwright workspace connection.

### Playwright CLI commands fail locally

Install the CLI before invoking browser tasks locally:

```bash
npm install -g @playwright/cli@latest
playwright-cli install --skills
```

The Dockerfile performs these steps for the hosted container.

### Live view or CDP URLs appear in logs or responses

The sample redacts common token and URL patterns before logging. If you add tools or prompts, do not ask the model to repeat CDP URLs or access tokens; the code stores the CDP URL server-side and injects live view links directly.
