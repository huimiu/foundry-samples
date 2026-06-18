**IMPORTANT!** All samples and other resources made available in this GitHub repository ("samples") are designed to assist in accelerating development of agents, solutions, and agent workflows for various scenarios. Review all provided resources and carefully test output behavior in the context of your use case. AI responses may be inaccurate and AI actions should be monitored with human oversight.

# Human-in-the-Loop Agent (Invocations Protocol)

A Bring Your Own hosted agent that demonstrates an approval-gate workflow using the [azure-ai-agentserver-invocations](https://pypi.org/project/azure-ai-agentserver-invocations/) SDK and a Foundry model. It generates a proposal, persists session state, and resumes after a human approves, requests a revision, rejects, or cancels the work.

## How It Works

1. [main.py](main.py) creates a Foundry Responses client with `DefaultAzureCredential` and the configured model deployment
2. `InvocationAgentServerHost(openapi_spec=OPENAPI_SPEC)` exposes `POST /invocations`, `GET /invocations/{id}`, `POST /invocations/{id}/cancel`, and the OpenAPI document at `/invocations/docs/openapi.json`
3. A new request with `{"task": "..."}` calls the model to generate a proposal and returns JSON with `status: "awaiting_approval"`
4. Session state is stored in memory and persisted as `hitl_session_<session>_<hash>.json` files under `$HOME`, making it accessible through the Session Files API when hosted
5. Follow-up requests in the same `agent_session_id` use `{"decision": "approve"}`, `{"decision": "revise", "feedback": "..."}`, or `{"decision": "reject"}` to complete, revise, or reject the proposal
6. `GET /invocations/{id}` returns current status, and the cancel handler marks a pending session as `cancelled`

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
azd ai agent invoke --local '{"task": "Draft a marketing email for our new AI product launch"}'
```

**PowerShell:**
```powershell
azd ai agent invoke --local '{\"task\": \"Draft a marketing email for our new AI product launch\"}'
```

**Test with curl:**

```bash
# Step 1: Submit a task — the agent generates a proposal
curl -sS -X POST "http://localhost:8088/invocations?agent_session_id=session-1" \
  -H "Content-Type: application/json" \
  -d '{"task": "Draft a marketing email for our new AI product launch"}' | jq .

# Step 2: Approve the proposal in the same session
curl -sS -X POST "http://localhost:8088/invocations?agent_session_id=session-1" \
  -H "Content-Type: application/json" \
  -d '{"decision": "approve"}' | jq .
```

<details>
<summary><h3>Using Foundry Toolkit VS Code Extension</h3></summary>

Open the **Agent Inspector** directly from the Foundry Toolkit extension to invoke the agent — no `curl` or CLI commands needed.

1. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Open Agent Inspector`.
2. The Inspector auto-connects to your running agent at `http://localhost:8088/`.
3. Type a message and send it. The Agent Inspector handles the protocol and displays the response inline.

> Multi-turn conversation is supported — the Inspector maintains session context across messages.

</details>

## Approval Workflow

```
[new task] ──► AWAITING_APPROVAL ──► (approve) ──► COMPLETED
                    │
                    ├──► (revise + feedback) ──► AWAITING_APPROVAL (loop)
                    │
                    └──► (reject) ──► REJECTED
```

Useful follow-up calls:

```bash
# Check status by invocation ID
curl -sS http://localhost:8088/invocations/<invocation_id> | jq .

# Request a revision
curl -sS -X POST "http://localhost:8088/invocations?agent_session_id=session-1" \
  -H "Content-Type: application/json" \
  -d '{"decision": "revise", "feedback": "Make the tone more casual and add a call-to-action"}' | jq .

# Reject the proposal
curl -sS -X POST "http://localhost:8088/invocations?agent_session_id=session-1" \
  -H "Content-Type: application/json" \
  -d '{"decision": "reject"}' | jq .

# Cancel a pending session
curl -sS -X POST http://localhost:8088/invocations/<invocation_id>/cancel | jq .
```

## OpenAPI Spec

The inline OpenAPI 3.0 specification documents the request and response contract:

```bash
curl -sS http://localhost:8088/invocations/docs/openapi.json | jq .
```

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
azd ai agent invoke '{"task": "Draft a marketing email for our new AI product launch"}'
```

**PowerShell:**
```powershell
azd ai agent invoke '{\"task\": \"Draft a marketing email for our new AI product launch\"}'
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

If model calls fail with `PermissionDenied` or `401 Unauthorized`, the identity running the agent does not have the required RBAC roles on the Azure AI Foundry project. Assign **Foundry User** and, if needed for the model endpoint, **Cognitive Services OpenAI User**, then wait a few minutes for role assignments to propagate.