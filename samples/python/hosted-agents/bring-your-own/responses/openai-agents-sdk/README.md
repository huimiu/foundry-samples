# OpenAI Agents SDK Agent (Responses Protocol)

A minimal hosted agent using the [OpenAI Agents SDK](https://pypi.org/project/openai-agents/) with a Microsoft Foundry model deployment and the **Responses protocol**. It uses Entra ID authentication through `DefaultAzureCredential`, so no OpenAI API key is required.

## How It Works

1. [main.py](main.py) creates an `AsyncOpenAI` client whose `base_url` points at the Foundry project `/openai/v1` endpoint
2. `DefaultAzureCredential` supplies bearer tokens for the `https://ai.azure.com/.default` scope, and the OpenAI Agents SDK is pointed at that Foundry-authenticated client
3. SDK tracing is disabled so the sample does not upload traces to `platform.openai.com` or require an OpenAI API key
4. An `Agent` named `foundry-assistant` uses the model deployment from `AZURE_AI_MODEL_DEPLOYMENT_NAME`
5. `ResponsesAgentServerHost` exposes an OpenAI-compatible `POST /responses` endpoint on `http://localhost:8088/`
6. The handler reads the request `input`, converts Responses protocol history into OpenAI Agents SDK role messages, and streams `response.output_text.delta` events through `TextResponse`

See [main.py](main.py) for the full implementation.

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `FOUNDRY_PROJECT_ENDPOINT` | Yes | Azure AI Foundry project endpoint URL. Auto-injected when hosted — only needed locally |
| `AZURE_AI_MODEL_DEPLOYMENT_NAME` | Yes | Model deployment name (e.g. `gpt-4o-mini`). Declared in `agent.yaml` |
| `APPLICATIONINSIGHTS_CONNECTION_STRING` | Optional | Application Insights connection string for local telemetry if tracing is configured; hosted containers may inject it |

When deployed as a hosted agent, `FOUNDRY_PROJECT_ENDPOINT` is auto-injected by the platform — you only need to set `AZURE_AI_MODEL_DEPLOYMENT_NAME`. Authentication uses Managed Identity via `DefaultAzureCredential`.

## Running Locally

### Prerequisites

- Python 3.12+
- An Azure AI Foundry project with a model deployment
- Azure credentials available to `DefaultAzureCredential` (for example, run `az login`)

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
azd ai agent invoke --local "What is Microsoft Foundry?"
```

**PowerShell:**
```powershell
azd ai agent invoke --local "What is Microsoft Foundry?"
```

**Test with curl:**

```bash
curl -sS -X POST http://localhost:8088/responses \
  -H "Content-Type: application/json" \
  -d '{"input": "What is Microsoft Foundry?", "stream": false}' | jq .
```

<details>
<summary><h3>Using Foundry Toolkit VS Code Extension</h3></summary>

Open the **Agent Inspector** directly from the Foundry Toolkit extension to invoke the agent — no `curl` or CLI commands needed.

1. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Open Agent Inspector`.
2. The Inspector auto-connects to your running agent at `http://localhost:8088/`.
3. Type a message and send it. The Agent Inspector handles the protocol and displays the response inline.

> Multi-turn conversation is supported — the Inspector maintains session context across messages.

</details>

## Streaming and Conversation History

The handler calls `context.get_history()` and converts prior Responses protocol input/output items into OpenAI Agents SDK role messages before appending the current user input. This enables multi-turn conversation when the client preserves the same session.

For streaming clients, include `"stream": true` and use `curl -N`:

```bash
curl -sS -N -X POST http://localhost:8088/responses \
  -H "Content-Type: application/json" \
  -d '{"input": "What hosted agent options does Microsoft Foundry offer?", "stream": true, "agent_session_id": "chat-001"}'
```

Only `RawResponsesStreamEvent` values with `response.output_text.delta` are emitted as text.

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
azd ai agent invoke "What is Microsoft Foundry?"
```

**PowerShell:**
```powershell
azd ai agent invoke "What is Microsoft Foundry?"
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

### `FOUNDRY_PROJECT_ENDPOINT` is not set

Use `azd ai agent run`, which sets the endpoint automatically, or set it manually for direct local runs:

```bash
export FOUNDRY_PROJECT_ENDPOINT="https://<account>.services.ai.azure.com/api/projects/<project>"
```

### Authentication fails locally

Run `az login` or configure another credential supported by `DefaultAzureCredential`. The sample requests tokens for `https://ai.azure.com/.default`.

### No streaming output

Use `curl -N` or another streaming-capable client and include `"stream": true` in the `/responses` request body.
