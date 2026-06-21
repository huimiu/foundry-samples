# LangGraph Observability Agent (Responses Protocol)

An instrumented [LangGraph](https://langchain-ai.github.io/langgraph/) agent hosted on Microsoft Foundry using the **Responses protocol**. It enables GenAI OpenTelemetry tracing so each LangGraph node, LLM call, and tool invocation is emitted to Azure Monitor / Application Insights.

## How It Works

1. `main.py` loads local settings from `.env`, then calls `enable_auto_tracing()` before building the graph.
2. The Foundry-backed `ChatOpenAI` model is built from `FOUNDRY_PROJECT_ENDPOINT` and `AZURE_AI_MODEL_DEPLOYMENT_NAME` using `DefaultAzureCredential`, with `use_responses_api=True` and `output_version="responses/v1"` so spans include response IDs.
3. Two LangChain tools are registered: `get_current_time` returns the current UTC time, and `calculator` evaluates a simple math expression in a restricted `eval` environment.
4. `langchain.agents.create_agent(...)` compiles a LangGraph ReAct agent with those tools and a brief system prompt.
5. `ResponsesHostServer` exposes the OpenAI-compatible `POST /responses` endpoint on `http://localhost:8088/` while OpenTelemetry spans flow to the configured exporter.

See [main.py](main.py) for the full implementation.

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `FOUNDRY_PROJECT_ENDPOINT` | Yes | Azure AI Foundry project endpoint URL. Auto-injected when hosted — only needed locally |
| `AZURE_AI_MODEL_DEPLOYMENT_NAME` | Yes | Model deployment name (e.g. `gpt-4o`). Declared in `agent.yaml` |
| `OTEL_AUTO_CONFIGURE_AZURE_MONITOR` | No | Set to `true` in `agent.yaml` so `enable_auto_tracing()` configures Azure Monitor automatically |
| `AZURE_TRACING_GEN_AI_CONTENT_RECORDING_ENABLED` | No | Set to `true` in `agent.yaml` and `.env.example` to capture prompts, completions, and tool I/O on spans |
| `APPLICATIONINSIGHTS_CONNECTION_STRING` | No | Injected by Foundry when hosted. Set manually only to export telemetry from a local `python main.py` run |

When deployed as a hosted agent, `FOUNDRY_PROJECT_ENDPOINT` and `APPLICATIONINSIGHTS_CONNECTION_STRING` are auto-injected by the platform — you only need to set `AZURE_AI_MODEL_DEPLOYMENT_NAME`. Authentication uses Managed Identity via `DefaultAzureCredential`.

## Running Locally

### Prerequisites

- Python 3.12+
- An Azure AI Foundry project with a model deployment (or let `azd provision` create one)
- Optional: an Application Insights connection string if you want local `python main.py` runs to export telemetry

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
azd ai agent invoke --local "What time is it right now, and what is 42 multiplied by 17?"
```

**PowerShell:**
```powershell
azd ai agent invoke --local "What time is it right now, and what is 42 multiplied by 17?"
```

**Test with curl:**

```bash
curl -sS -X POST http://localhost:8088/responses \
  -H "Content-Type: application/json" \
  -d '{"input": "What time is it right now, and what is 42 multiplied by 17?", "stream": false}' | jq .
```

<details>
<summary><h3>Using Foundry Toolkit VS Code Extension</h3></summary>

Open the **Agent Inspector** directly from the Foundry Toolkit extension to invoke the agent — no `curl` or CLI commands needed.

1. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Open Agent Inspector`.
2. The Inspector auto-connects to your running agent at `http://localhost:8088/`.
3. Type a message and send it. The Agent Inspector handles the protocol and displays the response inline.

> Multi-turn conversation is supported — the Inspector maintains session context across messages.

</details>

## Observability and Telemetry

LangGraph tracing is enabled with one startup call:

```python
from langchain_azure_ai.callbacks.tracers import enable_auto_tracing

enable_auto_tracing()
```

`enable_auto_tracing()` injects an Azure AI OpenTelemetry tracer into LangChain callback managers and LangGraph helper factories. The sample's agent and manifest set:

| Variable | Purpose |
|----------|---------|
| `OTEL_AUTO_CONFIGURE_AZURE_MONITOR` | Lets tracing configure the `TracerProvider` and Azure Monitor exporter using `APPLICATIONINSIGHTS_CONNECTION_STRING` |
| `AZURE_TRACING_GEN_AI_CONTENT_RECORDING_ENABLED` | Captures prompt, completion, and tool payload content on GenAI spans |

A typical span hierarchy for the invoke prompt is:

- `invoke_agent` — the overall agent turn.
- `chat` — each call to the underlying model.
- `execute_tool` — each tool invocation, such as `get_current_time` or `calculator`.

In hosted mode, Foundry injects the Application Insights connection string and traces appear in the Foundry portal under the agent's **Traces** tab. For local `python main.py` telemetry, set `APPLICATIONINSIGHTS_CONNECTION_STRING` and `OTEL_AUTO_CONFIGURE_AZURE_MONITOR=true` in `.env`.

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
azd ai agent invoke "What time is it right now, and what is 42 multiplied by 17?"
```

**PowerShell:**
```powershell
azd ai agent invoke "What time is it right now, and what is 42 multiplied by 17?"
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
