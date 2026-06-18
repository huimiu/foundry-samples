**IMPORTANT!** All samples and other resources made available in this GitHub repository ("samples") are designed to assist in accelerating development of agents, solutions, and agent workflows for various scenarios. Review all provided resources and carefully test output behavior in the context of your use case. AI responses may be inaccurate and AI actions should be monitored with human oversight.

# Browser Automation Agent (Responses Protocol)

A browser automation agent hosted on Microsoft Foundry using the [Agent Framework](https://github.com/microsoft/agent-framework), Foundry Toolbox, Azure Playwright Service, and the **Responses protocol**. It provisions a remote Chromium browser, drives it with Playwright CLI, and streams a live-view link back to the user.

## How It Works

1. `Program.cs` loads local `.env` values via `DotNetEnv`, reads the Foundry project endpoint, model deployment, toolbox name, and Playwright CLI timeout
2. If `FOUNDRY_AGENT_TOOLSET_ENDPOINT` is missing, the sample derives it from the project endpoint so toolbox discovery can reach `<project>/toolboxes`
3. The agent loads base instructions from [prompts/base.md](prompts/base.md) and a browser-operation skill from [skills/azure-playwright-browser-automation/SKILL.md](skills/azure-playwright-browser-automation/SKILL.md)
4. Local tools from [utils/Tools.cs](utils/Tools.cs) expose `run_playwright_cli`, `close_browser_session`, and `get_live_view_url`; Foundry Toolbox supplies remote browser `create_session`
5. Middleware in [utils/Middlewares.cs](utils/Middlewares.cs) intercepts `create_session`, stores CDP and live-view URLs server-side, and injects the live-view URL into normal and streaming responses
6. `AddFoundryToolboxes` connects to the configured toolbox with a scoped `https://ai.azure.com/.default` credential, and `AddFoundryResponses` hosts `POST /responses`
7. The agent starts on `http://localhost:8088/`

See [Program.cs](Program.cs) and the [utils/](utils/) folder for the full implementation.

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `FOUNDRY_PROJECT_ENDPOINT` | Yes | Azure AI Foundry project endpoint URL. Auto-injected when hosted — only needed locally |
| `AZURE_AI_MODEL_DEPLOYMENT_NAME` | Yes | Model deployment name. Declared in `agent.yaml` and `agent.manifest.yaml` |
| `TOOLBOX_NAME` | No | Foundry Toolbox name. Defaults to `browser-automation-tools`; override when using a compatible pre-existing toolbox |
| `BROWSER_AGENT_PLAYWRIGHT_CLI_TIMEOUT_SECONDS` | No | Timeout in seconds for each Playwright CLI command. Defaults to `180` |

When deployed as a hosted agent, `FOUNDRY_PROJECT_ENDPOINT` is auto-injected by the platform. Authentication uses Managed Identity via `DefaultAzureCredential`. Locally, the sample loads a `.env` file via `DotNetEnv` if present.

## Running Locally

### Prerequisites

- .NET 10.0 SDK or later (`dotnet --version`)
- An Azure AI Foundry project with a model deployment (or let `azd provision` create one)
- An Azure Playwright workspace plus access token, or an existing Foundry Toolbox with `browser_automation_preview`
- Node.js/npm with Playwright CLI installed for local runs: `npm install -g @playwright/cli@latest && playwright-cli install --skills`

### Using `azd` (Recommended)

Start the agent locally with the `run` command — `azd` restores dependencies, sets environment variables, builds, and starts the agent:

```bash
azd ai agent run
```

The agent starts on `http://localhost:8088/`.

<details>
<summary><h3>Using the Foundry Toolkit VS Code Extension</h3></summary>

The [Foundry Toolkit VS Code extension](https://learn.microsoft.com/en-us/azure/foundry/agents/quickstarts/quickstart-hosted-agent?view=foundry&pivots=vscode) has a built-in sample gallery that scaffolds this project directly into a new workspace — no manual cloning needed.

1. It's recommended to scaffold the project using the Foundry Toolkit extension. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Create new Hosted Agent`. The extension automatically creates the VS Code debug configuration files and `.env`.
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
3. Type a message and send it. The Agent Inspector handles the Responses protocol and displays the reply inline.

> Multi-turn conversation is supported — the Inspector maintains session context across messages.

</details>

## MCP Configuration

`agent.manifest.yaml` can provision the default `browser-automation-tools` toolbox. It creates a `PlaywrightWorkspace` project connection from these `azd` parameters:

| Parameter | Secret | Description |
|-----------|--------|-------------|
| `PLAYWRIGHT_SERVICE_URL` | No | Browser WebSocket endpoint for the Azure Playwright workspace |
| `PLAYWRIGHT_SERVICE_RESOURCE_ID` | No | Azure resource ID of the Playwright workspace |
| `PLAYWRIGHT_SERVICE_ACCESS_TOKEN` | Yes | Access token for the Azure Playwright workspace |

Set them before provisioning if you want this sample to create the connection and toolbox:

```bash
azd env set PLAYWRIGHT_SERVICE_URL "wss://<region>.api.playwright.microsoft.com/playwrightworkspaces/<workspace-id>/browsers"
azd env set PLAYWRIGHT_SERVICE_RESOURCE_ID "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.LoadTestService/playwrightWorkspaces/<workspace-name>"
azd env set PLAYWRIGHT_SERVICE_ACCESS_TOKEN "<playwright-workspace-access-token>"
```

If your Foundry project already has a compatible toolbox, skip those provisioning parameters and set `TOOLBOX_NAME` to that toolbox name. The toolbox must include the `browser_automation_preview` tool and its Playwright workspace connection.

## Browser Automation Components

| Path | Purpose |
|------|---------|
| [prompts/base.md](prompts/base.md) | Browser lifecycle, safety, cleanup, extraction, and form-filling rules |
| [skills/azure-playwright-browser-automation/SKILL.md](skills/azure-playwright-browser-automation/SKILL.md) | Playwright CLI command patterns for remote Azure Playwright Service sessions |
| [utils/Tools.cs](utils/Tools.cs) | Local tool factories and secure URL storage |
| [utils/Middlewares.cs](utils/Middlewares.cs) | Tool logging, `create_session` interception, and live-view URL injection |
| [utils/BrowserSession.cs](utils/BrowserSession.cs) | Playwright CLI subprocess runner with timeout and redaction |
| [utils/ToolboxScopedCredential.cs](utils/ToolboxScopedCredential.cs) | Credential wrapper that forces the toolbox auth scope to `https://ai.azure.com/.default` |

The `run_playwright_cli` tool accepts Playwright CLI arguments only; it does not expose arbitrary shell execution. Review authentication, network access, data handling, logging, browser permissions, and approval flows before adapting this sample for production.

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
