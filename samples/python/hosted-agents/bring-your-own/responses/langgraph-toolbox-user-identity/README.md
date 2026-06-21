<!-- Begin standard disclaimer — do not modify -->
**IMPORTANT!** All samples and other resources made available in this GitHub repository ("samples") are designed to assist in accelerating development of agents, solutions, and agent workflows for various scenarios. Review all provided resources and carefully test output behavior in the context of your use case. AI responses may be inaccurate and AI actions should be monitored with human oversight. Learn more in the transparency documents for [Agent Service](https://learn.microsoft.com/en-us/azure/ai-foundry/responsible-ai/agents/transparency-note) and [Agent Framework](https://github.com/microsoft/agent-framework/blob/main/TRANSPARENCY_FAQ.md).

Agents, solutions, or other output you create may be subject to legal and regulatory requirements, may require licenses, or may not be suitable for all industries, scenarios, or use cases. By using any sample, you are acknowledging that any output created using those samples are solely your responsibility, and that you will comply with all applicable laws, regulations, and relevant safety standards, terms of service, and codes of conduct.

Third-party samples contained in this folder are subject to their own designated terms, and they have not been tested or verified by Microsoft or its affiliates.

Microsoft has no responsibility to you or others with respect to any of these samples or any resulting output.
<!-- End standard disclaimer -->

# LangGraph Toolbox User Identity Agent (Responses Protocol)

A LangGraph ReAct agent hosted on Microsoft Foundry using the **Responses protocol**. It consumes a Foundry toolbox whose manifest defines delegated-user WorkIQ Mail/Calendar MCP connections and a managed OAuth GitHub MCP connection.

## How It Works

1. [main.py](main.py) loads local environment values, reads the Foundry project endpoint and model deployment, and resolves a toolbox from `TOOLBOX_ENDPOINT` or `TOOLBOX_NAME`
2. A `ChatOpenAI` client targets the Foundry project `/openai/v1` endpoint and authenticates with `DefaultAzureCredential`
3. `AzureAIProjectToolbox` loads tools from the toolbox; the Foundry toolbox connections handle user-identity or OAuth authorization for each remote MCP server
4. On cold start, the agent retries toolbox loading up to five times if the toolbox is temporarily unavailable or returns no tools
5. Tool errors are returned as tool messages (`handle_tool_error = True`), and malformed object schemas are sanitized before tools are bound to a LangGraph ReAct agent
6. `ResponsesAgentServerHost` exposes an OpenAI-compatible `POST /responses` endpoint on `http://localhost:8088/`
7. The handler extracts the request `input`, invokes the LangGraph agent with a 240-second timeout, and emits a Responses output message

See [main.py](main.py) for the full implementation.

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `FOUNDRY_PROJECT_ENDPOINT` | Yes | Azure AI Foundry project endpoint URL. Auto-injected when hosted — only needed locally |
| `AZURE_AI_MODEL_DEPLOYMENT_NAME` | Yes | Model deployment name (e.g. `gpt-4.1`). Declared in `agent.yaml` |
| `TOOLBOX_NAME` | Yes for hosted manifest | Toolbox name. The manifest sets `langgraph-toolbox-user-identity-tools` |
| `TOOLBOX_ENDPOINT` | Yes for local endpoint override | Full toolbox MCP endpoint URL, including `?api-version=v1`. Used instead of `TOOLBOX_NAME` when set |
| `FOUNDRY_AGENT_TOOLBOX_FEATURES` | Optional | Feature-flag header value for toolbox requests. Defaults to `Toolboxes=V1Preview` |
| `APPLICATIONINSIGHTS_CONNECTION_STRING` | Optional | Application Insights connection string for local telemetry if tracing is enabled; hosted containers may inject it |

When deployed as a hosted agent, `FOUNDRY_PROJECT_ENDPOINT` is auto-injected by the platform. The manifest provisions `TOOLBOX_NAME`; for local testing, set `TOOLBOX_ENDPOINT` to the toolbox MCP URL.

## Running Locally

### Prerequisites

- Python 3.12+
- An Azure AI Foundry project with a model deployment
- A Foundry toolbox containing the WorkIQ Mail, WorkIQ Calendar, and GitHub MCP connections from [agent.manifest.yaml](agent.manifest.yaml), or an equivalent existing toolbox endpoint
- Azure credentials available to `DefaultAzureCredential` (for example, run `az login`)
- End-user consent for delegated WorkIQ or GitHub tools when those tools require it

### Using `azd`

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

The [Foundry Toolkit VS Code extension](https://learn.microsoft.com/en-us/azure/foundry/agents/quickstarts/quickstart-hosted-agent?view=foundry&pivots=vscode) sets up a one-click **F5** debug experience for this sample.

1. Scaffold the project using the Foundry Toolkit extension: open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Create new Hosted Agent`. The extension automatically creates the VS Code debug configuration files and `.env`.
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
# Create and activate a virtual environment
python -m venv .venv
source .venv/bin/activate  # Windows (PowerShell): .\.venv\Scripts\Activate.ps1

# Install dependencies
pip install -r requirements.txt

# Configure environment variables
cp .env.example .env  # skip if .env already exists
# Edit .env — see Environment Variables above

# Run the agent
python main.py
```

The agent starts on `http://localhost:8088/`.

</details>

## Invoke

### Using azd

**Local:**

**Bash:**
```bash
azd ai agent invoke --local "List the toolbox tools and explain when consent is required."
```

**PowerShell:**
```powershell
azd ai agent invoke --local "List the toolbox tools and explain when consent is required."
```

**Test with curl:**

```bash
curl -sS -X POST http://localhost:8088/responses \
  -H "Content-Type: application/json" \
  -d '{"input": "List the toolbox tools and explain when consent is required.", "stream": false}' | jq .
```

<details>
<summary><h3>Using Foundry Toolkit VS Code Extension</h3></summary>

Open the **Agent Inspector** directly from the Foundry Toolkit extension to invoke the agent — no `curl` or CLI commands needed.

1. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Open Agent Inspector`.
2. The Inspector auto-connects to your running agent at `http://localhost:8088/`.
3. Type a message and send it. The Agent Inspector handles the protocol and displays the response inline.

> Multi-turn conversation is supported — the Inspector maintains session context across messages.

</details>

## User Identity and OAuth Toolbox

The agent does not read or store user tokens directly. It authenticates to Foundry with `DefaultAzureCredential`; the Foundry toolbox layer and its project connections handle delegated user identity, OAuth consent, and remote MCP authorization.

[agent.manifest.yaml](agent.manifest.yaml) defines these resources:

| Connection | Auth type | Target |
|------------|-----------|--------|
| `workiq-mail-conn` | `UserEntraToken` | `https://agent365.svc.cloud.microsoft/agents/servers/mcp_MailTools` |
| `workiq-calendar-conn` | `UserEntraToken` | `https://agent365.svc.cloud.microsoft/agents/servers/mcp_CalendarTools` |
| `github-oauth-conn` | Managed `OAuth2` connector `foundrygithubmcp` | `https://api.githubcopilot.com/mcp` |

The manifest also creates a toolbox named `langgraph-toolbox-user-identity-tools` and sets `TOOLBOX_NAME` in the hosted agent. For local testing against an existing toolbox, set `TOOLBOX_ENDPOINT` in `.env` to the full MCP URL:

```text
https://<account>.services.ai.azure.com/api/projects/<project>/toolboxes/langgraph-toolbox-user-identity-tools/mcp?api-version=v1
```

If a user has not consented to a delegated tool, the MCP gateway may return a consent-required error or tool message. Complete the consent flow, then retry the same prompt.

## Toolbox Deployment Notes

When using `azd ai agent init` with this sample's manifest, the generated project carries the connection and toolbox definitions forward into `azure.yaml`. The first toolbox version becomes the default. Use `azd ai toolbox list`, `azd ai toolbox show langgraph-toolbox-user-identity-tools`, and `azd ai toolbox version list langgraph-toolbox-user-identity-tools` to inspect the deployed toolbox.

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
azd ai agent invoke "List the toolbox tools and explain when consent is required."
```

**PowerShell:**
```powershell
azd ai agent invoke "List the toolbox tools and explain when consent is required."
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

### Agent starts but returns no tools

Check that either `TOOLBOX_ENDPOINT` points to an existing toolbox with `?api-version=v1`, or `TOOLBOX_NAME` matches a toolbox in the Foundry project. The agent retries transient cold-start failures, but a missing toolbox or wrong name must be fixed in configuration.

### Consent required for WorkIQ or GitHub tools

Complete the delegated consent flow returned by the MCP gateway. WorkIQ tools use user Entra tokens, and the GitHub tool uses Foundry's managed OAuth connector, so consent is per user rather than a static secret in `.env`.

### Tool schemas rejected by the model

The agent sanitizes object schemas that are missing `properties`. If you still see `400 Invalid tool schema`, inspect the raw schema returned by the MCP server.
