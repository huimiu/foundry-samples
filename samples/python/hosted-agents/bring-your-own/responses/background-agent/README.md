**IMPORTANT!** All samples and other resources made available in this GitHub repository ("samples") are designed to assist in accelerating development of agents, solutions, and agent workflows for various scenarios. Review all provided resources and carefully test output behavior in the context of your use case. AI responses may be inaccurate and AI actions should be monitored with human oversight.

# Background Agent (Responses Protocol)

A long-running Bring Your Own hosted agent built with [azure-ai-agentserver-responses](https://pypi.org/project/azure-ai-agentserver-responses/) and the Foundry SDK. It uses the **Responses protocol** to run asynchronous research-style model calls, including background polling and cancellation.

## How It Works

1. [main.py](main.py) creates a `ResponsesAgentServerHost`, which exposes the Responses protocol endpoint `POST /responses` and the SDK-managed response polling and cancellation routes.
2. The handler reads the request body `input` field with `context.get_input_text()` and defaults to a general AI trends topic when no input is provided.
3. It creates an `AIProjectClient` with `DefaultAzureCredential`, gets a Foundry OpenAI Responses client, and calls the configured `AZURE_AI_MODEL_DEPLOYMENT_NAME` with a research analyst system prompt.
4. Model output is requested with `stream=True`; each `response.output_text.delta` event is yielded as text through `TextResponse`.
5. The Responses SDK handles synchronous, streaming, and `background: true` modes, including response status storage, `GET /responses/{id}` polling, and `POST /responses/{id}/cancel` cancellation.

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `FOUNDRY_PROJECT_ENDPOINT` | Yes, for local runs | Azure AI Foundry project endpoint URL. Auto-injected when hosted — only needed locally |
| `AZURE_AI_MODEL_DEPLOYMENT_NAME` | Yes | Model deployment name (e.g. `gpt-4.1-mini`). Declared in `agent.yaml` and `agent.manifest.yaml` |
| `APPLICATIONINSIGHTS_CONNECTION_STRING` | No | Enables local telemetry. Auto-injected in hosted containers when monitoring is configured |

When deployed as a hosted agent, `FOUNDRY_PROJECT_ENDPOINT` is auto-injected by the platform. Authentication uses Managed Identity via `DefaultAzureCredential`; locally, use `az login` or another supported credential source.

## Running Locally

### Prerequisites

- Python 3.10+
- Azure CLI installed and authenticated (`az login`) or another `DefaultAzureCredential` source
- An Azure AI Foundry project with a model deployment (or let `azd provision` create one)

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
azd ai agent invoke --local "Analyze the impact of AI on healthcare"
```

**PowerShell:**
```powershell
azd ai agent invoke --local "Analyze the impact of AI on healthcare"
```

**Test with curl:**

```bash
curl -sS -X POST http://localhost:8088/responses \
  -H "Content-Type: application/json" \
  -d '{"input": "Analyze the impact of AI on healthcare", "stream": false}' | jq .
```

<details>
<summary><h3>Using Foundry Toolkit VS Code Extension</h3></summary>

Open the **Agent Inspector** directly from the Foundry Toolkit extension to invoke the agent — no `curl` or CLI commands needed.

1. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Open Agent Inspector`.
2. The Inspector auto-connects to your running agent at `http://localhost:8088/`.
3. Type a message and send it. The Agent Inspector handles the protocol and displays the response inline.

> Multi-turn conversation is supported — the Inspector maintains session context across messages.

</details>

## Background Requests

The canonical invoke above sends a synchronous Responses request. To exercise the long-running path, submit a background request, poll for its result, or cancel it while it is still running:

```bash
# Submit a background research analysis
curl -sS -X POST http://localhost:8088/responses \
  -H "Content-Type: application/json" \
  -d '{"input": "Analyze the impact of AI on healthcare", "background": true, "store": true}' | jq .

# Poll for result (use the id from the response)
curl -sS http://localhost:8088/responses/<response_id> | jq .

# Cancel an in-flight request
curl -sS -X POST http://localhost:8088/responses/<response_id>/cancel | jq .
```

The same handler also supports regular streaming when the request includes `"stream": true`; the SDK emits Responses protocol SSE lifecycle events and text deltas.

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
azd ai agent invoke "Analyze the impact of AI on healthcare"
```

**PowerShell:**
```powershell
azd ai agent invoke "Analyze the impact of AI on healthcare"
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