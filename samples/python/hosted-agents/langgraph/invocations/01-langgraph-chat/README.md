# LangGraph Chat Agent (Invocations Protocol)

A [LangGraph](https://langchain-ai.github.io/langgraph/) multi-turn chat agent hosted on Microsoft Foundry using the **Invocations protocol**. It demonstrates a LangGraph ReAct loop with two local tools and server-side conversation state keyed by `agent_session_id`.

## How It Works

1. `main.py` loads local settings from `.env`, then builds a Foundry-backed `ChatOpenAI` model from `FOUNDRY_PROJECT_ENDPOINT` and `AZURE_AI_MODEL_DEPLOYMENT_NAME` using `DefaultAzureCredential`.
2. Two LangChain tools are registered: `get_current_time` returns the current UTC time, and `calculator` evaluates a simple math expression in a restricted `eval` environment.
3. `langchain.agents.create_agent(...)` compiles a LangGraph ReAct agent with a `MemorySaver` checkpointer so each `agent_session_id` maps to a persistent LangGraph thread.
4. `InvocationsHostServer` exposes `POST /invocations` on `http://localhost:8088/` and reads the user prompt from the JSON `message` field.
5. The host returns final assistant text for normal requests and streams token deltas as SSE when the request includes `"stream": true`.

See [main.py](main.py) for the full implementation.

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `FOUNDRY_PROJECT_ENDPOINT` | Yes | Azure AI Foundry project endpoint URL. Auto-injected when hosted — only needed locally |
| `AZURE_AI_MODEL_DEPLOYMENT_NAME` | Yes | Model deployment name (e.g. `gpt-4o`). Declared in `agent.yaml` |

When deployed as a hosted agent, `FOUNDRY_PROJECT_ENDPOINT` is auto-injected by the platform — you only need to set `AZURE_AI_MODEL_DEPLOYMENT_NAME`. Authentication uses Managed Identity via `DefaultAzureCredential`.

## Running Locally

### Prerequisites

- Python 3.12+
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
azd ai agent invoke --local '{"message": "What time is it right now?"}'
```

**PowerShell:**
```powershell
azd ai agent invoke --local '{\"message\": \"What time is it right now?\"}'
```

**Test with curl:**

```bash
curl -N -X POST http://localhost:8088/invocations \
  -H "Content-Type: application/json" \
  -d '{"message": "What time is it right now?"}'
```

**SSE Event Format:**

Add `"stream": true` to the same `message` payload to receive token deltas as SSE `data:` lines, followed by `event: done`:

```
data: The current

data:  UTC time is

...
event: done
data: {}
```

<details>
<summary><h3>Using Foundry Toolkit VS Code Extension</h3></summary>

Open the **Agent Inspector** directly from the Foundry Toolkit extension to invoke the agent — no `curl` or CLI commands needed.

1. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Open Agent Inspector`.
2. The Inspector auto-connects to your running agent at `http://localhost:8088/`.
3. Type a message and send it. The Agent Inspector handles the protocol and displays the response inline.

> Multi-turn conversation is supported — the Inspector maintains session context across messages.

</details>

## Multi-Turn Sessions and Streaming

The Invocations protocol sample uses `message` rather than `input` in request bodies. For multi-turn conversations, capture the `x-agent-session-id` response header from the first turn and pass it as `agent_session_id` on follow-up requests:

```bash
curl -i -X POST http://localhost:8088/invocations \
  -H "Content-Type: application/json" \
  -d '{"message": "What time is it right now?"}'

curl -X POST 'http://localhost:8088/invocations?agent_session_id=REPLACE_WITH_SESSION_ID' \
  -H "Content-Type: application/json" \
  -d '{"message": "What is 42 * 17?"}'
```

To stream, include `"stream": true`:

```bash
curl -N -X POST http://localhost:8088/invocations \
  -H "Content-Type: application/json" \
  -d '{"message": "Count to 5.", "stream": true}'
```

## The LangGraph Agent

The compiled graph is created by `langchain.agents.create_agent(model, tools=[...], checkpointer=MemorySaver())`, which implements the standard ReAct loop: call the model, execute requested tools, loop back to the model, and return the final assistant message. The `MemorySaver` checkpointer is appropriate for a demo; use a durable checkpointer such as Redis, Cosmos DB, or another persistent store for production.

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
azd ai agent invoke '{"message": "What time is it right now?"}'
```

**PowerShell:**
```powershell
azd ai agent invoke '{\"message\": \"What time is it right now?\"}'
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
