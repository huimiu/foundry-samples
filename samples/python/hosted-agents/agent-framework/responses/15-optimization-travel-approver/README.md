<!-- Begin standard disclaimer — do not modify -->
**IMPORTANT!** All samples and other resources made available in this GitHub repository ("samples") are designed to assist in accelerating development of agents, solutions, and agent workflows for various scenarios. Review all provided resources and carefully test output behavior in the context of your use case. AI responses may be inaccurate and AI actions should be monitored with human oversight. Learn more in the transparency note for [Agent Service](https://learn.microsoft.com/en-us/azure/ai-foundry/responsible-ai/agents/transparency-note).

Agents, solutions, or other output you create may be subject to legal and regulatory requirements, may require licenses, or may not be suitable for all industries, scenarios, or use cases. By using any sample, you are acknowledging that any output created using those samples are solely your responsibility, and that you will comply with all applicable laws, regulations, and relevant safety standards, terms of service, and codes of conduct.

Third-party samples contained in this folder are subject to their own designated terms, and they have not been tested or verified by Microsoft or its affiliates.

Microsoft has no responsibility to you or others with respect to any of these samples or any resulting output.
<!-- End standard disclaimer -->

> [!IMPORTANT]
> Agent Optimizer is currently in limited preview and only available through a sign-up process. To access the service, complete the [intake form](https://aka.ms/ao/preview-form). This preview is provided without a service-level agreement, and we don't recommend it for production workloads. Certain features might not be supported or might have constrained capabilities. For more information, see [Supplemental Terms of Use for Microsoft Azure Previews](https://azure.microsoft.com/en-us/support/legal/preview-supplemental-terms/).

# Optimization Travel Approver Agent (Responses Protocol)

A travel request approval agent built with [Agent Framework](https://github.com/microsoft/agent-framework), hosted on Microsoft Foundry using the **Responses protocol**, and structured for the Foundry Agent Optimizer. It demonstrates optimizing instructions, skill content, and tool descriptions for a policy-heavy business workflow.

## How It Works

1. `main.py` calls `load_config()` from `azure.ai.agentserver.optimization` to load an optimizer candidate, local baseline config, or fallback settings
2. If no optimized skills are present, it loads baseline skills from `.agent_configs/baseline/metadata.yaml` and the [skills](skills/) directory
3. `config.compose_instructions()` combines the selected system instructions and skills into the prompt used by the Agent Framework agent
4. Three mocked `@tool` functions return JSON for travel policy, department budget, and flight alternatives
5. `config.apply_tool_descriptions(tools)` applies optimized tool descriptions when an optimizer candidate supplies them
6. `FoundryChatClient` connects to the configured Foundry model deployment with `DefaultAzureCredential`
7. `ResponsesHostServer` exposes the agent through an OpenAI-compatible `POST /responses` endpoint and starts on `http://localhost:8088/`

See [main.py](main.py), [eval.yaml](eval.yaml), [.agent_configs/baseline](.agent_configs/baseline/), [skills](skills/), and [evaluators](evaluators/) for the full implementation.

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `FOUNDRY_PROJECT_ENDPOINT` | Yes | Azure AI Foundry project endpoint URL. Auto-injected when hosted — only needed locally |
| `AZURE_AI_MODEL_DEPLOYMENT_NAME` | Yes | Model deployment name (e.g. `gpt-4.1-mini`). Declared in `agent.yaml` |
| `OPTIMIZATION_LOCAL_DIR` | Yes | Local optimization config directory. Defaults to `.agent_configs` in `agent.yaml` |
| `OPTIMIZATION_MODEL_DEPLOYMENT_NAME` | For optimization | Optimizer model deployment declared by `agent.manifest.yaml`; used by optimization workflows, not directly by `main.py` |
| `OPTIMIZATION_CANDIDATE_ID` | No | Set by optimizer apply flows when a candidate should override the local baseline |
| `APPLICATIONINSIGHTS_CONNECTION_STRING` | No | Optional local telemetry connection string. Auto-injected when hosted if telemetry is configured |

When deployed as a hosted agent, `FOUNDRY_PROJECT_ENDPOINT` is auto-injected by the platform — you only need to set `AZURE_AI_MODEL_DEPLOYMENT_NAME` and keep `OPTIMIZATION_LOCAL_DIR` aligned with the packaged config. Authentication uses Managed Identity via `DefaultAzureCredential`.

## Running Locally

### Prerequisites

- Python 3.12+
- An Azure AI Foundry project with a model deployment (or let `azd provision` create one)
- Azure Developer CLI with the agent optimizer extension used by this sample
- Access to the Agent Optimizer limited preview

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
azd ai agent invoke --local "I need to book a trip to Tokyo next week for a client meeting. Budget is 5000 dollars."
```

**PowerShell:**
```powershell
azd ai agent invoke --local "I need to book a trip to Tokyo next week for a client meeting. Budget is 5000 dollars."
```

**Test with curl:**

```bash
curl -sS -X POST http://localhost:8088/responses \
  -H "Content-Type: application/json" \
  -d '{"input": "I need to book a trip to Tokyo next week for a client meeting. Budget is 5000 dollars.", "stream": false}' | jq .
```

<details>
<summary><h3>Using Foundry Toolkit VS Code Extension</h3></summary>

Open the **Agent Inspector** directly from the Foundry Toolkit extension to invoke the agent — no `curl` or CLI commands needed.

1. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Open Agent Inspector`.
2. The Inspector auto-connects to your running agent at `http://localhost:8088/`.
3. Type a message and send it. The Agent Inspector handles the protocol and displays the response inline.

> Multi-turn conversation is supported — the Inspector maintains session context across messages.

</details>

## Agent Optimizer

> [!IMPORTANT]
> Agent Optimizer is currently in limited preview and only available through a sign-up process. To access the service, complete the [intake form](https://aka.ms/ao/preview-form). This preview is provided without a service-level agreement, and is not recommended for production workloads.

This sample includes everything needed to run an optimization job:

| Path | Purpose |
|------|---------|
| `eval.yaml` | Agent optimizer configuration, including dataset, evaluators, and model names |
| `eval/travel_approval_golden.jsonl` | Golden travel approval scenarios and expected answers |
| `evaluators/rubric_dimensions.json` | Domain-specific rubric dimensions |
| `.agent_configs/baseline/` | Baseline instructions, skill directory reference, and tool descriptions |
| `skills/policy-reviewer/SKILL.md` | Deliberately weak baseline skill for the optimizer to improve |

Run optimization:

```bash
azd ai agent optimize
```

Monitor progress:

```bash
azd ai agent optimize status <job-id> --watch
```

Apply the best candidate:

```bash
azd ai agent optimize apply --candidate <candidate-id>
```

After applying a candidate, restart or redeploy the agent so `load_config()` can pick up the optimized instructions, skill content, model, and tool descriptions.

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
azd ai agent invoke "I need to book a trip to Tokyo next week for a client meeting. Budget is 5000 dollars."
```

**PowerShell:**
```powershell
azd ai agent invoke "I need to book a trip to Tokyo next week for a client meeting. Budget is 5000 dollars."
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

### Optimizer commands are unavailable

Confirm you have installed the agent optimizer extension required by your `azd` version and that your account has access to the limited preview.

### Optimized changes do not appear in agent behavior

Confirm `azd ai agent optimize apply --candidate <candidate-id>` completed, then restart the local host or redeploy the hosted agent. `load_config()` reads the selected config at startup.

### Baseline config is not loaded

Verify `OPTIMIZATION_LOCAL_DIR` points to `.agent_configs` and that `.agent_configs/baseline/metadata.yaml`, `instructions.md`, and `tools.json` are present in the deployed package.
