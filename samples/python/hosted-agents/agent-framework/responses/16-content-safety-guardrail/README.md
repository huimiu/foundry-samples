# Content Safety Guardrail Agent (Responses Protocol)

An [Agent Framework](https://github.com/microsoft/agent-framework) agent hosted on Microsoft Foundry using the **Responses protocol**, with a Responsible AI (RAI) content safety guardrail attached at the hosted-agent definition level. The Python code is intentionally close to the basic hosted agent; the guardrail behavior is configured in `agent.yaml` and applied by the platform.

## How It Works

1. `main.py` creates a `FoundryChatClient` pointed at your Foundry project endpoint and model deployment, authenticated with `DefaultAzureCredential`
2. The client is wrapped in an Agent Framework `Agent` with a short system prompt (*"You are a friendly assistant. Keep your answers brief."*)
3. `agent.yaml` and `agent.manifest.yaml` declare a `policies` list with a `rai_policy` entry and a full ARM resource ID in `rai_policy_name`
4. Microsoft Foundry applies the referenced RAI policy at runtime to screen prompts and responses according to your guardrail configuration
5. `ResponsesHostServer` exposes the agent through an OpenAI-compatible `POST /responses` endpoint and starts on `http://localhost:8088/`

See [main.py](main.py), [agent.yaml](agent.yaml), and [agent.manifest.yaml](agent.manifest.yaml) for the full implementation.

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `FOUNDRY_PROJECT_ENDPOINT` | Yes | Azure AI Foundry project endpoint URL. Auto-injected when hosted — only needed locally |
| `AZURE_AI_MODEL_DEPLOYMENT_NAME` | Yes | Model deployment name (e.g. `gpt-4.1-mini`). Declared in `agent.yaml` |

When deployed as a hosted agent, `FOUNDRY_PROJECT_ENDPOINT` is auto-injected by the platform — you only need to set `AZURE_AI_MODEL_DEPLOYMENT_NAME`. Authentication uses Managed Identity via `DefaultAzureCredential`.

## Running Locally

### Prerequisites

- Python 3.10+
- An Azure AI Foundry project with a model deployment (or let `azd provision` create one)
- An RAI policy created on your Foundry resource, with its full ARM resource ID available for `rai_policy_name`

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
azd ai agent invoke --local "Write a short friendly hello message."
```

**PowerShell:**
```powershell
azd ai agent invoke --local "Write a short friendly hello message."
```

**Test with curl:**

```bash
curl -sS -X POST http://localhost:8088/responses \
  -H "Content-Type: application/json" \
  -d '{"input": "Write a short friendly hello message.", "stream": false}' | jq .
```

<details>
<summary><h3>Using Foundry Toolkit VS Code Extension</h3></summary>

Open the **Agent Inspector** directly from the Foundry Toolkit extension to invoke the agent — no `curl` or CLI commands needed.

1. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Open Agent Inspector`.
2. The Inspector auto-connects to your running agent at `http://localhost:8088/`.
3. Type a message and send it. The Agent Inspector handles the protocol and displays the response inline.

> Multi-turn conversation is supported — the Inspector maintains session context across messages.

</details>

## Content Safety

Set `rai_policy_name` in both [agent.yaml](agent.yaml) and [agent.manifest.yaml](agent.manifest.yaml) to the full ARM resource ID of an RAI policy on your Foundry resource:

```yaml
policies:
  - type: rai_policy
    rai_policy_name: /subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.CognitiveServices/accounts/<account>/raiPolicies/<policy-name>
```

Use the full ARM resource ID, not the bare policy name. If you omit the `policies` block, the hosted agent deploys without a guardrail. If you keep the `policies` block but omit `rai_policy_name`, the platform applies the default policy, `Microsoft.DefaultV2`.

To verify the guardrail after deployment, send a benign prompt and then a prompt that violates your configured policy. A blocked prompt returns `HTTP 400` with a `content_filter` error:

```json
{
  "error": {
    "code": "content_filter",
    "message": "The request was blocked due to content safety policy violation at input stage.",
    "type": "content_safety_error"
  }
}
```

Useful references:

- [Add a content safety guardrail to a hosted agent](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/add-hosted-agent-guardrails)
- [Configure guardrails and controls](https://learn.microsoft.com/en-us/azure/foundry/guardrails/how-to-create-guardrails)
- [Guardrails and controls overview](https://learn.microsoft.com/en-us/azure/foundry/guardrails/guardrails-overview)

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
azd ai agent invoke "Write a short friendly hello message."
```

**PowerShell:**
```powershell
azd ai agent invoke "Write a short friendly hello message."
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

### Deployment fails because the RAI policy is missing

Replace the placeholder `rai_policy_name` with the full ARM resource ID of an RAI policy that exists on the Foundry resource you deploy to. If `azd provision` created a new Foundry resource, create or select an RAI policy there before deploying.

### A prompt you expected to block is allowed

Confirm the referenced policy filters the relevant content category and severity. The sample only attaches the policy; filtering behavior comes from the policy configuration.
