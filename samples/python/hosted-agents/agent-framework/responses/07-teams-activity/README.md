**IMPORTANT!** All samples and other resources made available in this GitHub repository ("samples") are designed to assist in accelerating development of agents, solutions, and agent workflows for various scenarios. Review all provided resources and carefully test output behavior in the context of your use case. AI responses may be inaccurate and AI actions should be monitored with human oversight.

# Teams Activity Agent (Responses Protocol)

An [Agent Framework](https://github.com/microsoft/agent-framework) hosted agent for Microsoft Foundry using the **Responses protocol**. It can be deployed to Foundry, published to Teams and Microsoft 365, and optionally connected to Work IQ Teams and calendar tools through a Foundry Toolbox.

## How It Works

1. `main.py` creates a `DefaultAzureCredential`, a Foundry Toolbox bearer-token provider, and a `FoundryChatClient` for your Foundry project endpoint and model deployment
2. `resolve_toolbox_endpoint()` builds the toolbox MCP endpoint from `FOUNDRY_PROJECT_ENDPOINT` and `TOOLBOX_NAME`
3. `MCPStreamableHTTPTool` is configured with the `Foundry-Features: Toolboxes=V1Preview` header and the `teams-tools` toolbox declared in `agent.manifest.yaml`
4. The Agent Framework `Agent` uses the toolbox only when `ENABLE_WORK_IQ` is enabled; `agent.yaml` sets it to `false` by default, so the sample starts as a plain chat assistant unless you opt in
5. `_ResilientResponsesHostServer` extends `ResponsesHostServer` to tolerate transient hosted-history fetch failures and exposes an OpenAI-compatible `POST /responses` endpoint on `http://localhost:8088/`

See [main.py](main.py) for the full implementation.

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `FOUNDRY_PROJECT_ENDPOINT` | Yes | Azure AI Foundry project endpoint URL. Auto-injected when hosted — only needed locally |
| `AZURE_AI_MODEL_DEPLOYMENT_NAME` | Yes | Model deployment name (e.g. `gpt-4o`). Declared in `agent.yaml` |
| `TOOLBOX_NAME` | Yes | Foundry Toolbox name used to build the Work IQ toolbox MCP endpoint. Defaults to `teams-tools` in `agent.yaml` |
| `ENABLE_WORK_IQ` | No | Set to `true` to enable Teams and calendar Work IQ tools. Defaults to `false` in `agent.yaml` |

When deployed as a hosted agent, `FOUNDRY_PROJECT_ENDPOINT` is auto-injected by the platform — you only need to set `AZURE_AI_MODEL_DEPLOYMENT_NAME`, `TOOLBOX_NAME`, and optionally `ENABLE_WORK_IQ`. Authentication uses Managed Identity via `DefaultAzureCredential`.

## Running Locally

### Prerequisites

- Python 3.10+
- An Azure AI Foundry project with a model deployment (or let `azd provision` create one)
- A Microsoft 365/Teams environment if you plan to publish the deployed agent to Teams
- Work IQ Teams and calendar connections plus the `teams-tools` toolbox from `agent.manifest.yaml` if you set `ENABLE_WORK_IQ=true`

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
azd ai agent invoke --local "How many meetings do I have tomorrow?"
```

**PowerShell:**
```powershell
azd ai agent invoke --local "How many meetings do I have tomorrow?"
```

**Test with curl:**

```bash
curl -sS -X POST http://localhost:8088/responses \
  -H "Content-Type: application/json" \
  -d '{"input": "How many meetings do I have tomorrow?", "stream": false}' | jq .
```

<details>
<summary><h3>Using Foundry Toolkit VS Code Extension</h3></summary>

Open the **Agent Inspector** directly from the Foundry Toolkit extension to invoke the agent — no `curl` or CLI commands needed.

1. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Open Agent Inspector`.
2. The Inspector auto-connects to your running agent at `http://localhost:8088/`.
3. Type a message and send it. The Agent Inspector handles the Responses protocol and displays the reply inline.

> Multi-turn conversation is supported — the Inspector maintains session context across messages.

</details>

## Work IQ Configuration

`agent.manifest.yaml` declares two user-authenticated remote-tool connections and one toolbox:

- `workiq-teams-conn` targets the Microsoft Teams MCP server
- `workiq-calendar-conn` targets the Outlook Calendar MCP server
- `teams-tools` combines those MCP connections into the toolbox consumed by the agent

The sample keeps `ENABLE_WORK_IQ=false` by default. Set it to `true` only after the Work IQ connections and toolbox are provisioned and users are ready to sign in. When disabled, the agent still runs and can be published, but it does not answer Teams or calendar data questions using tools.

![Using Work IQ tool](./teams-activity.png)

## Publishing the Agent

1. After deploying the agent, open the Foundry portal, click **Publish**, then choose **Publish to Teams and Microsoft 365**. For the full flow, see [Publish an agent to Microsoft 365 Copilot](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/publish-copilot).
2. The Foundry portal creates the Azure Bot resource and configures the messaging endpoint automatically.
3. End users need to sign in the first time they access the agent.

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
azd ai agent invoke "How many meetings do I have tomorrow?"
```

**PowerShell:**
```powershell
azd ai agent invoke "How many meetings do I have tomorrow?"
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