# A2A Caller Agent (Responses Protocol)

A Foundry-hosted [Agent Framework](https://github.com/microsoft/agent-framework) agent using the **Responses protocol**. It acts as a friendly concierge that can answer directly or delegate math questions to the companion executor agent over the [A2A protocol](https://a2a-protocol.org/latest/) through a Foundry Toolbox.

## How It Works

1. `main.py` builds the toolbox MCP endpoint from `FOUNDRY_PROJECT_ENDPOINT` and `TOOLBOX_NAME`
2. `DefaultAzureCredential` and an Azure AI bearer token provider authenticate toolbox calls, and the HTTP client opts into the `Toolboxes=V1Preview` feature
3. `MCPStreamableHTTPTool` connects to the Foundry Toolbox, which exposes a `math_expert` `a2a_preview` tool backed by the executor's RemoteA2A connection
4. `FoundryChatClient` connects the Agent Framework `Agent` to your Foundry model deployment
5. The agent's instructions tell it to delegate specialist questions to the remote A2A tool, then summarize the result in a concise, friendly tone
6. `_ResilientResponsesHostServer` serves the agent through `POST /responses` on `http://localhost:8088/` and tolerates transient history fetch failures

See [main.py](main.py) for the full implementation.

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `FOUNDRY_PROJECT_ENDPOINT` | Yes | Azure AI Foundry project endpoint URL. Auto-injected when hosted — only needed locally |
| `AZURE_AI_MODEL_DEPLOYMENT_NAME` | Yes | Model deployment name (e.g. `gpt-4o`). Declared in `agent.yaml` |
| `TOOLBOX_NAME` | Yes | Foundry Toolbox name that exposes the executor through an `a2a_preview` tool. Defaults to `a2a-delegation-tools` |

When deployed as a hosted agent, `FOUNDRY_PROJECT_ENDPOINT` is auto-injected by the platform — you only need to set `AZURE_AI_MODEL_DEPLOYMENT_NAME` and `TOOLBOX_NAME`. Authentication uses Managed Identity via `DefaultAzureCredential`.

## Running Locally

### Prerequisites

- Python 3.10+
- An Azure AI Foundry project with a model deployment (or let `azd provision` create one)
- To exercise delegation, deploy the companion executor, enable incoming A2A, and provision the caller toolbox before running the caller

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
azd ai agent invoke --local "What is 15 multiplied by 23?"
```

**PowerShell:**
```powershell
azd ai agent invoke --local "What is 15 multiplied by 23?"
```

**Test with curl:**

```bash
curl -sS -X POST http://localhost:8088/responses \
  -H "Content-Type: application/json" \
  -d '{"input": "What is 15 multiplied by 23?", "stream": false}' | jq .
```

<details>
<summary><h3>Using Foundry Toolkit VS Code Extension</h3></summary>

Open the **Agent Inspector** directly from the Foundry Toolkit extension to invoke the agent — no `curl` or CLI commands needed.

1. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Open Agent Inspector`.
2. The Inspector auto-connects to your running agent at `http://localhost:8088/`.
3. Type a message and send it. The Agent Inspector handles the protocol and displays the response inline.

> Multi-turn conversation is supported — the Inspector maintains session context across messages.

</details>

## Delegation Relationship

This caller and the companion [executor](../executor/README.md) are a two-agent sample. The executor is the math specialist; the caller is the user-facing concierge. The caller does not call the executor directly from Python code — it uses a Foundry Toolbox named `a2a-delegation-tools` with a `math_expert` `a2a_preview` tool.

The caller's [agent.manifest.yaml](agent.manifest.yaml) declares the `RemoteA2A` connection and toolbox. When the manifest asks for `a2a_executor_endpoint`, use the A2A endpoint printed by the executor's `scripts/setup-a2a` script. For the full walkthrough, see the [parent README](../README.md).

## Running and Deploying the Delegation Pair

1. In the `executor` directory, configure `.env`, deploy the executor, and run `scripts\setup-a2a.ps1` or `scripts/setup-a2a.sh` to enable incoming A2A.
2. Copy the printed A2A endpoint URL.
3. In this `caller` directory, provide that URL as the `a2a_executor_endpoint` manifest parameter; `azd provision` creates the `RemoteA2A` connection and the `a2a-delegation-tools` toolbox.
4. Run the caller locally or deploy it, then invoke it with a math prompt such as `What is 15 multiplied by 23?`.

The local caller still uses the remote Foundry Toolbox and remote executor; running `python main.py` does not start the executor locally.

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
azd ai agent invoke "What is 15 multiplied by 23?"
```

**PowerShell:**
```powershell
azd ai agent invoke "What is 15 multiplied by 23?"
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

### Toolbox lookup fails

Confirm `TOOLBOX_NAME` matches the toolbox created by caller `azd provision` (default `a2a-delegation-tools`) and that the caller was provisioned after you supplied the executor's A2A endpoint.

### Delegation or discovery fails

Ensure the executor was deployed, `scripts/setup-a2a` completed successfully, and the `a2a_executor_endpoint` value points to the executor's `/endpoint/protocols/a2a/` URL.
