**IMPORTANT!** All samples and other resources made available in this GitHub repository ("samples") are designed to assist in accelerating development of agents, solutions, and agent workflows for various scenarios. Review all provided resources and carefully test output behavior in the context of your use case. AI responses may be inaccurate and AI actions should be monitored with human oversight.

# AG-UI Protocol Agent (Invocations Protocol)

A minimal Bring Your Own hosted agent implementing the [AG-UI protocol](https://docs.ag-ui.com/introduction) over the Microsoft Foundry **Invocations protocol**. It uses [Pydantic AI](https://ai.pydantic.dev/) with an Azure OpenAI Responses model and streams standard AG-UI events as SSE.

## How It Works

1. [main.py](main.py) reads `FOUNDRY_PROJECT_ENDPOINT` and `AZURE_AI_MODEL_DEPLOYMENT_NAME`, derives the Azure OpenAI endpoint, and authenticates with `DefaultAzureCredential`
2. It creates an `AsyncAzureOpenAI` client, wraps it in a Pydantic AI `OpenAIResponsesModel`, and builds an `Agent` with the instruction *"You are a helpful assistant."*
3. `InvocationAgentServerHost` exposes the Foundry `POST /invocations` endpoint
4. The invocation handler passes the full Starlette request to Pydantic AI's `handle_ag_ui_request`, so the request body must be an AG-UI `RunAgentInput` payload (`threadId`, `runId`, `messages`, `tools`, `context`, and related fields)
5. Pydantic AI streams AG-UI lifecycle events such as `RUN_STARTED`, `TEXT_MESSAGE_CONTENT`, and `RUN_FINISHED` as Server-Sent Events
6. The agent starts on `http://localhost:8088/`

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `FOUNDRY_PROJECT_ENDPOINT` | Yes | Azure AI Foundry project endpoint URL. Auto-injected when hosted — only needed locally |
| `AZURE_AI_MODEL_DEPLOYMENT_NAME` | Yes | Model deployment name (e.g. `gpt-4.1-mini`). Declared in `agent.yaml` |
| `APPLICATIONINSIGHTS_CONNECTION_STRING` | No | Application Insights connection string. Auto-injected when hosted — only needed locally for telemetry |

Authentication uses Managed Identity via `DefaultAzureCredential`; no API key is required.

## Running Locally

### Prerequisites

- Python 3.10+
- An Azure AI Foundry project with a model deployment (or let `azd provision` create one)
- Azure credentials available to `DefaultAzureCredential` (for example, `az login`)

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
azd ai agent invoke --local '{"threadId": "thread-1", "runId": "run-1", "state": {}, "messages": [{"id": "msg-1", "role": "user", "content": "Hello from AG-UI"}], "tools": [], "context": [], "forwardedProps": {}}'
```

**PowerShell:**
```powershell
azd ai agent invoke --local '{\"threadId\": \"thread-1\", \"runId\": \"run-1\", \"state\": {}, \"messages\": [{\"id\": \"msg-1\", \"role\": \"user\", \"content\": \"Hello from AG-UI\"}], \"tools\": [], \"context\": [], \"forwardedProps\": {}}'
```

**Test with curl:**

```bash
curl -N -X POST http://localhost:8088/invocations \
  -H "Content-Type: application/json" \
  -H "Accept: text/event-stream" \
  -d '{"threadId": "thread-1", "runId": "run-1", "state": {}, "messages": [{"id": "msg-1", "role": "user", "content": "Hello from AG-UI"}], "tools": [], "context": [], "forwardedProps": {}}'
```

**SSE Event Format:**

Standard AG-UI events are streamed automatically:

```
data: {"type":"RUN_STARTED","threadId":"thread-1","runId":"run-1"}

data: {"type":"TEXT_MESSAGE_START","messageId":"...","role":"assistant"}

data: {"type":"TEXT_MESSAGE_CONTENT","messageId":"...","delta":"Hello"}

...

data: {"type":"RUN_FINISHED","threadId":"thread-1","runId":"run-1"}
```

<details>
<summary><h3>Using Foundry Toolkit VS Code Extension</h3></summary>

Open the **Agent Inspector** directly from the Foundry Toolkit extension to invoke the agent — no `curl` or CLI commands needed.

1. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Open Agent Inspector`.
2. The Inspector auto-connects to your running agent at `http://localhost:8088/`.
3. Type a message and send it. The Agent Inspector handles the protocol and displays the response inline.

> Multi-turn conversation is supported — the Inspector maintains session context across messages.

</details>

## Learn More

- [AG-UI Protocol](https://docs.ag-ui.com/introduction) — event types, lifecycle, and tools
- [Pydantic AI AG-UI docs](https://ai.pydantic.dev/ag-ui/) — `handle_ag_ui_request` integration
- [AG-UI Dojo](https://dojo.ag-ui.com/) — interactive playground for testing AG-UI agents

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
azd ai agent invoke '{"threadId": "thread-1", "runId": "run-1", "state": {}, "messages": [{"id": "msg-1", "role": "user", "content": "Hello from AG-UI"}], "tools": [], "context": [], "forwardedProps": {}}'
```

**PowerShell:**
```powershell
azd ai agent invoke '{\"threadId\": \"thread-1\", \"runId\": \"run-1\", \"state\": {}, \"messages\": [{\"id\": \"msg-1\", \"role\": \"user\", \"content\": \"Hello from AG-UI\"}], \"tools\": [], \"context\": [], \"forwardedProps\": {}}'
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

### Azure OpenAI Permission Denied (401)

If the model request fails with `PermissionDenied` or `401 Unauthorized`, the identity running the agent may not have the required Foundry/OpenAI data-plane roles on the project. Assign the appropriate roles (for example, **Foundry User** and **Cognitive Services OpenAI User**) to the local user or hosted agent identity, then wait a few minutes for RBAC propagation before retrying.