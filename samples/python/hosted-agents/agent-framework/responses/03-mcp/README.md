# Remote MCP Tools Agent (Responses Protocol)

An [Agent Framework](https://github.com/microsoft/agent-framework) agent hosted on Microsoft Foundry using the **Responses protocol**. It demonstrates how to connect to a remote MCP server — the GitHub Copilot MCP endpoint — so tools can be discovered and invoked at runtime.

## How It Works

1. `main.py` creates a `FoundryChatClient` pointed at your Foundry project endpoint and model deployment, authenticated with `DefaultAzureCredential`
2. The sample reads `GITHUB_PAT` and registers a remote MCP tool named `GitHub` with `client.get_mcp_tool(...)`, pointing at `https://api.githubcopilot.com/mcp/`
3. The MCP tool sends the PAT as a bearer token and uses `approval_mode="never_require"` for tool calls
4. The client is wrapped in an Agent Framework `Agent` with the remote MCP tool and a short system prompt (*"You are a friendly assistant. Keep your answers brief."*)
5. The agent is served by `ResponsesHostServer`, which exposes an OpenAI-compatible `POST /responses` endpoint on `http://localhost:8088/`

See [main.py](main.py) for the full implementation.

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `FOUNDRY_PROJECT_ENDPOINT` | Yes | Azure AI Foundry project endpoint URL. Auto-injected when hosted — only needed locally |
| `AZURE_AI_MODEL_DEPLOYMENT_NAME` | Yes | Model deployment name (e.g. `gpt-4o`). Declared in `agent.yaml` |
| `GITHUB_PAT` | Yes | GitHub Personal Access Token used to authenticate to the GitHub Copilot MCP server. Declared in `agent.yaml` |

When deployed as a hosted agent, `FOUNDRY_PROJECT_ENDPOINT` is auto-injected by the platform — you only need to set `AZURE_AI_MODEL_DEPLOYMENT_NAME` and `GITHUB_PAT`. Authentication to Foundry uses Managed Identity via `DefaultAzureCredential`.

## Running Locally

### Prerequisites

- Python 3.10+
- An Azure AI Foundry project with a model deployment (or let `azd provision` create one)
- A GitHub Personal Access Token for authenticating with the GitHub Copilot MCP server

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
azd ai agent invoke --local "List my GitHub repositories."
```

**PowerShell:**
```powershell
azd ai agent invoke --local "List my GitHub repositories."
```

**Test with curl:**

```bash
curl -sS -X POST http://localhost:8088/responses \
  -H "Content-Type: application/json" \
  -d '{"input": "List my GitHub repositories.", "stream": false}' | jq .
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

The agent connects to the GitHub Copilot MCP server at `https://api.githubcopilot.com/mcp/`. Set `GITHUB_PAT` in `.env` for local runs:

```bash
GITHUB_PAT="ghp_your_token_here"
```

If you plan to deploy with `azd`, also store the token in the azd environment so it can be injected into the hosted agent:

```bash
azd env set GITHUB_PAT="ghp_your_token_here"
```

The current code requires the `GITHUB_PAT` environment variable to exist. If it is present but empty, the agent logs a warning and runs without registering the GitHub MCP tool.

## Next Steps

- [Quickstart: Create a hosted agent](https://learn.microsoft.com/en-us/azure/foundry/agents/quickstarts/quickstart-hosted-agent) — end-to-end walkthrough using `azd`
- [Tool catalog](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/tool-catalog) — browse available tools to extend your agent
- [Manage hosted agents](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/manage-hosted-agent) — monitor and manage deployed agents
- [Basic agent](../01-basic/) — minimal agent with no tools
- [Add local tools](../02-tools/) — sample with locally defined Python tool functions
- [Use Foundry Toolbox](../04-foundry-toolbox/) — sample with Azure Foundry Toolbox integration

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
azd ai agent invoke "List my GitHub repositories."
```

**PowerShell:**
```powershell
azd ai agent invoke "List my GitHub repositories."
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
