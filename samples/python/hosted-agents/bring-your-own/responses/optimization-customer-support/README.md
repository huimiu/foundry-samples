<!-- Begin standard disclaimer — do not modify -->
**IMPORTANT!** All samples and other resources made available in this GitHub repository ("samples") are designed to assist in accelerating development of agents, solutions, and agent workflows for various scenarios. Review all provided resources and carefully test output behavior in the context of your use case. AI responses may be inaccurate and AI actions should be monitored with human oversight. Learn more in the transparency note for [Agent Service](https://learn.microsoft.com/en-us/azure/ai-foundry/responsible-ai/agents/transparency-note).

Agents, solutions, or other output you create may be subject to legal and regulatory requirements, may require licenses, or may not be suitable for all industries, scenarios, or use cases. By using any sample, you are acknowledging that any output created using those samples are solely your responsibility, and that you will comply with all applicable laws, regulations, and relevant safety standards, terms of service, and codes of conduct.

Third-party samples contained in this folder are subject to their own designated terms, and they have not been tested or verified by Microsoft or its affiliates.

Microsoft has no responsibility to you or others with respect to any of these samples or any resulting output.
<!-- End standard disclaimer -->

> [!IMPORTANT]
> Agent Optimizer is currently in limited preview and only available through a sign-up process. To access the service, complete the [intake form](https://aka.ms/ao/preview-form). This preview is provided without a service-level agreement, and we don't recommend it for production workloads. Certain features might not be supported or might have constrained capabilities. For more information, see [Supplemental Terms of Use for Microsoft Azure Previews](https://azure.microsoft.com/en-us/support/legal/preview-supplemental-terms/).

# Optimization Customer Support Agent (Responses Protocol)

A realistic customer support agent for a consumer electronics company, hosted on Microsoft Foundry using the **Responses protocol**. It demonstrates agent optimization by starting with a deliberately sparse baseline prompt and loading optimized instructions and skills at runtime.

## How It Works

1. [main.py](main.py) calls `load_config()` from `azure-ai-agentserver-optimization` at startup to resolve optimizer candidate, local baseline, or fallback instructions
2. The baseline configuration in `.agent_configs/baseline/` contains the bare instruction `You are a helpful customer support agent.` and points to the sample `skills/` directory
3. An `AIProjectClient` authenticated with `DefaultAzureCredential` creates a Foundry OpenAI Responses client
4. `ResponsesAgentServerHost` exposes an OpenAI-compatible `POST /responses` endpoint on `http://localhost:8088/`
5. The handler reads the request `input`, includes prior Responses protocol history, calls the model with the composed instructions, and emits a Responses event stream with usage metadata
6. During optimizer evaluations, optimizer-provided candidate configuration is selected by `load_config()` before the agent processes requests

See [main.py](main.py) for the full implementation.

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `FOUNDRY_PROJECT_ENDPOINT` | Yes | Azure AI Foundry project endpoint URL. Auto-injected when hosted — only needed locally |
| `AZURE_AI_MODEL_DEPLOYMENT_NAME` | Yes | Runtime model deployment name (e.g. `gpt-4.1-mini`). Declared in `agent.yaml` |
| `OPTIMIZATION_LOCAL_DIR` | Yes | Local optimization config directory. `agent.yaml` sets `.agent_configs` |
| `OPTIMIZATION_MODEL_DEPLOYMENT_NAME` | Optional | Optimization model deployment used by the optimizer workflow; provisioned by the manifest, not read by `main.py` at runtime |
| `APPLICATIONINSIGHTS_CONNECTION_STRING` | Optional | Application Insights connection string for local telemetry. Hosted containers may inject it |

When deployed as a hosted agent, `FOUNDRY_PROJECT_ENDPOINT` is auto-injected by the platform — you need to set `AZURE_AI_MODEL_DEPLOYMENT_NAME` and keep `OPTIMIZATION_LOCAL_DIR` at `.agent_configs` for the bundled baseline.

## Running Locally

### Prerequisites

- Python 3.12+
- An Azure AI Foundry project with a runtime model deployment
- Azure credentials available to `DefaultAzureCredential` (for example, run `az login`)
- Azure Developer CLI with Foundry agent commands; install the agent optimizer extension when running optimization workflows
- Agent Optimizer preview access if you plan to run `azd ai agent optimize`

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
azd ai agent invoke --local "I ordered a laptop 3 days ago and have not received a shipping confirmation. Order #12345."
```

**PowerShell:**
```powershell
azd ai agent invoke --local "I ordered a laptop 3 days ago and have not received a shipping confirmation. Order #12345."
```

**Test with curl:**

```bash
curl -sS -X POST http://localhost:8088/responses \
  -H "Content-Type: application/json" \
  -d '{"input": "I ordered a laptop 3 days ago and have not received a shipping confirmation. Order #12345.", "stream": false}' | jq .
```

<details>
<summary><h3>Using Foundry Toolkit VS Code Extension</h3></summary>

Open the **Agent Inspector** directly from the Foundry Toolkit extension to invoke the agent — no `curl` or CLI commands needed.

1. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Open Agent Inspector`.
2. The Inspector auto-connects to your running agent at `http://localhost:8088/`.
3. Type a message and send it. The Agent Inspector handles the protocol and displays the response inline.

> Multi-turn conversation is supported — the Inspector maintains session context across messages.

</details>

## Customer Support Scenario

The sample represents **Contoso Electronics** support. The evaluation set covers order status, returns and exchanges, warranty claims, device troubleshooting, angry-customer complaint handling, escalation, recommendations, and safety boundaries for electrical or medical questions.

The baseline is intentionally weak so optimization can show a meaningful quality delta:

| Optimization target | What it improves |
|---------------------|------------------|
| Instructions | Rewrites the system prompt with policy, tone, structure, and guardrails |
| Skills | Adds procedural guidance for returns, troubleshooting, escalation, and safety boundaries |

## Agent Optimization

> [!IMPORTANT]
> Agent Optimizer is currently in limited preview and only available through a sign-up process. To access the service, complete the [intake form](https://aka.ms/ao/preview-form). This preview is provided without a service-level agreement, and we don't recommend it for production workloads. Certain features might not be supported or might have constrained capabilities. For more information, see [Supplemental Terms of Use for Microsoft Azure Previews](https://azure.microsoft.com/en-us/support/legal/preview-supplemental-terms/).

This sample ships with [eval.yaml](eval.yaml), [eval.jsonl](eval.jsonl), and [eval-quick.jsonl](eval-quick.jsonl).

```bash
# Run optimization from the azd project created for this agent
azd ai agent optimize

# Monitor progress
azd ai agent optimize status <job-id> --watch

# Apply the selected candidate
azd ai agent optimize apply --candidate <candidate-id>
```

After applying a candidate, invoke the agent again and compare whether the response is more empathetic, policy-aware, and structured.

## Optimization Assets

| Path | Purpose |
|------|---------|
| `.agent_configs/baseline/instructions.md` | Deliberately sparse baseline instructions |
| `.agent_configs/baseline/metadata.yaml` | Points `load_config()` at the baseline instruction file and `skills/` directory |
| `skills/escalation/SKILL.md` | Escalation guidance for frustrated users or requests for a person |
| `skills/handle-return/SKILL.md` | Return and refund guidance |
| `skills/safety-boundaries/SKILL.md` | Boundaries for dangerous electrical work or medical-risk questions |
| `skills/troubleshoot-device/SKILL.md` | Basic device troubleshooting guidance |
| `eval.jsonl` | Full evaluation dataset with 10 tasks and roughly 40 criteria |
| `eval-quick.jsonl` | Smaller three-task dataset for faster iteration |

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
azd ai agent invoke "I ordered a laptop 3 days ago and have not received a shipping confirmation. Order #12345."
```

**PowerShell:**
```powershell
azd ai agent invoke "I ordered a laptop 3 days ago and have not received a shipping confirmation. Order #12345."
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
