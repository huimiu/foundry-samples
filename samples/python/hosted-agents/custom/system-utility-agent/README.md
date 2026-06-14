**IMPORTANT!** All samples and other resources made available in this GitHub repository ("samples") are designed to assist in accelerating development of agents, solutions, and agent workflows for various scenarios. Review all provided resources and carefully test output behavior in the context of your use case. AI responses may be inaccurate and AI actions should be monitored with human oversight. Learn more in the transparency documents for [Agent Service](https://learn.microsoft.com/en-us/azure/ai-foundry/responsible-ai/agents/transparency-note) and [Agent Framework](https://github.com/microsoft/agent-framework/blob/main/TRANSPARENCY_FAQ.md).

Agents, solutions, or other output you create may be subject to legal and regulatory requirements, may require licenses, or may not be suitable for all industries, scenarios, or use cases. By using any sample, you are acknowledging that any output created using those samples are solely your responsibility, and that you will comply with all applicable laws, regulations, and relevant safety standards, terms of service, and codes of conduct.

Third-party samples contained in this folder are subject to their own designated terms, and they have not been tested or verified by Microsoft or its affiliates.

Microsoft has no responsibility to you or others with respect to any of these samples or any resulting output.

# What this sample demonstrates

This sample demonstrates how to build and host a **System Utility Agent** that can inspect its runtime environment using tool calls (processes, ports, resources, DNS, and environment variables), hosted using the
[Azure AI AgentServer SDK](https://learn.microsoft.com/en-us/dotnet/api/overview/azure/ai.agentserver.agentframework-readme) and deployed to Microsoft Foundry using the Azure Developer CLI [ai agent](https://aka.ms/azdaiagent/docs) extension.

## How It Works

### System Utility Agent

This agent is designed for diagnostics and “what’s running here?” questions. It is **container-aware** and will report whether the agent is likely seeing the container namespace vs the host.

The agent exposes a small set of tools (implemented locally in Python) and uses an OpenAI-style tool-calling loop to answer questions.

### Tracing (custom spans)

This sample also demonstrates how to add **custom spans** to hosted agent traces using OpenTelemetry. The agent creates spans around the overall request and each tool-calling iteration, and annotates them with useful attributes (conversation ID, model name, token usage, tool name, tool arguments, and tool result). This is useful when you want richer observability than the default hosted-agent traces.

Tools included:

1. **capability_report** - Report what the agent can likely observe (host vs container scope)
2. **system_info** - OS / Python / CPU metadata
3. **resource_snapshot** - CPU / memory / disk snapshot
4. **list_processes** - List running processes (visibility depends on container scope)
5. **process_details** - Get details for a specific process
6. **check_port** - Check whether a TCP port is listening / reachable
7. **dns_lookup** - Resolve a hostname
8. **list_environment_variables** - List environment variables (supports redaction)

### Agent Hosting

The agent is hosted using the [Azure AI AgentServer SDK](https://learn.microsoft.com/en-us/dotnet/api/overview/azure/ai.agentserver.agentframework-readme),
which provisions a REST API endpoint compatible with the OpenAI Responses protocol. This allows interaction with the agent using OpenAI Responses compatible clients.

### Agent Deployment

The hosted agent can be deployed to Microsoft Foundry using the Azure Developer CLI [ai agent](https://aka.ms/azdaiagent/docs) extension.
The extension builds a container image for the agent, deploys it to Azure Container Instances (ACI), and creates a hosted agent version and deployment on Foundry Agent Service.

## Running the Agent Locally

### Prerequisites

Before running this sample, ensure you have:

1. Python 3.10+ installed
2. Either:
	- A Microsoft Foundry project endpoint configured (recommended), or
	- An Azure OpenAI endpoint + API key configured
3. If using Foundry project auth, Azure CLI installed and authenticated (`az login`) so `DefaultAzureCredential` can acquire a token

### Environment Variables

This sample supports loading environment variables from a `.env` file (via `python-dotenv`) or your shell environment.

#### Option A (Recommended): Microsoft Foundry project endpoint

- `AZURE_AI_PROJECT_ENDPOINT` - Your Foundry project endpoint (required for this option)
- `AZURE_AI_MODEL_DEPLOYMENT_NAME` - Model deployment name (optional, defaults to `gpt-4o-mini`)

```powershell
# Replace with your Foundry project endpoint
$env:AZURE_AI_PROJECT_ENDPOINT="https://your-project.ai.azure.com"

# Optional, defaults to gpt-4o-mini
$env:AZURE_AI_MODEL_DEPLOYMENT_NAME="gpt-4o-mini"
```

#### Option B: Azure OpenAI key-based configuration

- `AZURE_ENDPOINT` - Your Azure OpenAI endpoint URL (required for this option)
- `OPENAI_API_KEY` - Your Azure OpenAI API key (required for this option)
- `OPENAI_API_VERSION` - Optional, defaults to `2025-03-01-preview`
- `AZURE_AI_MODEL_DEPLOYMENT_NAME` - Optional, defaults to `gpt-4o-mini`

```powershell
# Replace with your Azure OpenAI endpoint
$env:AZURE_ENDPOINT="https://your-openai-resource.openai.azure.com/"

$env:OPENAI_API_KEY="<your key>"

# Optional
$env:OPENAI_API_VERSION="2025-03-01-preview"
$env:AZURE_AI_MODEL_DEPLOYMENT_NAME="gpt-4o-mini"
```

### Using `azd` (Recommended)

The [Azure Developer CLI `ai agent` extension](https://aka.ms/azdaiagent/docs) is the quickest way to run, invoke, and deploy this hosted agent.

**Run locally**

Set the required environment variables (see [Environment Variables](#environment-variables) above) in your `azd` environment so they are injected when running and deploying:

```bash
azd env set AZURE_AI_PROJECT_ENDPOINT "https://your-project.ai.azure.com"
```

Start the agent locally:

```bash
azd ai agent run
```

The agent starts on `http://localhost:8088/`.

**Invoke**

**Bash:**
```bash
azd ai agent invoke --local '{"input": "What environment are you running in? Summarize what you can observe","stream":false}'
```

**PowerShell:**
```powershell
azd ai agent invoke --local '{\"input\": \"What environment are you running in? Summarize what you can observe\",\"stream\":false}'
```

**Deploy to Microsoft Foundry**

```bash
# Provision Azure resources (skip if already provisioned)
azd provision

# Build, push, and deploy the agent to Foundry
azd deploy
```

After deploying, invoke the agent running in Foundry:

```bash
azd ai agent invoke '{"input": "What environment are you running in? Summarize what you can observe","stream":false}'
```

Stream logs from the running agent:

```bash
azd ai agent monitor
```

For the full deployment guide, see [Azure AI Foundry hosted agents](https://aka.ms/azdaiagent/docs).

<details>
<summary><h3>Using the Foundry Toolkit VS Code Extension</h3></summary>

The [Foundry Toolkit VS Code extension](https://learn.microsoft.com/en-us/azure/foundry/agents/quickstarts/quickstart-hosted-agent?view=foundry&pivots=vscode) has a built-in sample gallery that scaffolds this project into a new workspace — no manual cloning needed.

**Run in debug mode**

1. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Create new Hosted Agent`. The extension creates the VS Code debug configuration files and `.env`.
2. Set the required environment variables (see [Environment Variables](#environment-variables) above).
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

**Invoke with the Agent Inspector**

1. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Open Agent Inspector`.
2. The Inspector auto-connects to your running agent at `http://localhost:8088/`.
3. Type a message and send it.

**Deploy with the Deploy wizard**

1. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Deploy Hosted Agent`. The extension opens a tab-based **Deploy Hosted Agent** wizard and reads `agent.yaml` to auto-populate what it can.
2. If prompted, complete **Foundry Project Setup** to pick the subscription and Foundry project (or create a new one).
3. On the **Basics** tab, set the deployment method (**Code** or **Container**), the packaging/registry options, and the **Hosted Agent Name**.
4. On the **Review + Deploy** tab, confirm the runtime details, pick a **CPU and Memory** size, and click **Deploy**.
5. After deployment, invoke the agent in the Agent Playground and stream live logs from the **Logs** tab.

</details>

<details>
<summary><h3>Manual setup (without `azd`)</h3></summary>

### Installing Dependencies

Install the required Python dependencies using pip:

```powershell
pip install -r requirements.txt
```

### Running the Sample

To run the agent locally, execute:

```powershell
python main.py
```

This will start the hosted agent locally and expose an OpenAI Responses-compatible endpoint (typically on `http://localhost:8088/`).

### Interacting with the Agent

```powershell
curl -sS -H "Content-Type: application/json" -X POST http://localhost:8088/responses -d '{"input": "What environment are you running in? Summarize what you can observe","stream":false}'
```

### Deploying the Agent to Microsoft Foundry

To deploy your agent to Microsoft Foundry, follow the comprehensive deployment guide at https://aka.ms/azdaiagent/docs

</details>

## Troubleshooting

### After deployed, the agent appears stateless (chat history is not preserved)

- Make sure you are using a model deployment hosted in foundry.
- If you are not chat with your agent from UI, be sure to pass an existing foundry conversation ID in the `conversation` field of your create responses request.

### Images built on Apple Silicon or other ARM64 machines do not work on our service

We **recommend using `azd` cloud build**, which always builds images with the correct architecture.

If you choose to **build locally**, and your machine is **not `linux/amd64`** (for example, an Apple Silicon Mac), the image will **not be compatible with our service**, causing runtime failures.

**Fix for local builds**

Use this command to build the image locally:

```shell
docker build --platform=linux/amd64 -t image .
```

This forces the image to be built for the required `amd64` architecture.

