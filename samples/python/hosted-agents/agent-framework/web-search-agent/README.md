**IMPORTANT!** All samples and other resources made available in this GitHub repository ("samples") are designed to assist in accelerating development of agents, solutions, and agent workflows for various scenarios. Review all provided resources and carefully test output behavior in the context of your use case. AI responses may be inaccurate and AI actions should be monitored with human oversight. Learn more in the transparency documents for [Agent Service](https://learn.microsoft.com/en-us/azure/ai-foundry/responsible-ai/agents/transparency-note) and [Agent Framework](https://github.com/microsoft/agent-framework/blob/main/TRANSPARENCY_FAQ.md).

Agents, solutions, or other output you create may be subject to legal and regulatory requirements, may require licenses, or may not be suitable for all industries, scenarios, or use cases. By using any sample, you are acknowledging that any output created using those samples are solely your responsibility, and that you will comply with all applicable laws, regulations, and relevant safety standards, terms of service, and codes of conduct.

Third-party samples contained in this folder are subject to their own designated terms, and they have not been tested or verified by Microsoft or its affiliates.

Microsoft has no responsibility to you or others with respect to any of these samples or any resulting output.

# What this sample demonstrates

This sample demonstrates how to build a web search agent using Bing Grounding, hosted using
[Azure AI AgentServer SDK](https://pypi.org/project/azure-ai-agentserver-agentframework/) and
deploy it to Microsoft Foundry using the Azure Developer CLI [ai agent](https://aka.ms/azdaiagent/docs) extension.

## How It Works

### Web Search Agent

The agent uses Bing Grounding to search the web for current information and provide accurate, well-sourced answers. This demonstrates:

- How to integrate Bing Grounding as a tool in an AI agent
- How to use the `HostedWebSearchTool` from the Agent Framework

### Agent Hosting

The agent is hosted using the [Azure AI AgentServer SDK](https://pypi.org/project/azure-ai-agentserver-agentframework/),
which provisions a REST API endpoint compatible with the OpenAI Responses protocol. This allows interaction with the agent using OpenAI Responses compatible clients.

### Agent Deployment

The hosted agent can be seamlessly deployed to Microsoft Foundry using the Azure Developer CLI [ai agent](https://learn.microsoft.com/en-us/azure/ai-foundry/agents/concepts/hosted-agents?view=foundry&tabs=cli#create-a-hosted-agent) extension.
The extension builds a container image into Azure Container Registry (ACR), and creates a hosted agent version and deployment on Microsoft Foundry.

## Running the Agent Locally

### Prerequisites

Before running this sample, ensure you have:

1. An Azure AI Foundry project configured
2. A deployment of a chat model (e.g., `gpt-4.1-mini`)
3. A Bing Grounding connection in your project
4. Azure CLI installed and authenticated (`az login`)
5. Python 3.10+ installed

### Environment Variables

Create a `.env` file with the following environment variables:

> **Note:** The `.env` file is for local development only. When deploying to Azure AI Foundry, remove the `.env` file and configure environment variables in `agent.yaml` instead.

```bash
AZURE_AI_PROJECT_ENDPOINT=https://<your-foundry-account>.services.ai.azure.com/api/projects/<your-project>
AZURE_AI_MODEL_DEPLOYMENT_NAME=<your-model-deployment>  # e.g., gpt-4.1-mini
BING_GROUNDING_CONNECTION_ID=/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.CognitiveServices/accounts/<foundry-account>/projects/<project>/connections/<bing-connection-name>
```

### Using `azd` (Recommended)

The [Azure Developer CLI `ai agent` extension](https://aka.ms/azdaiagent/docs) is the quickest way to run, invoke, and deploy this hosted agent.

**Run locally**

Set the required environment variables (see [Environment Variables](#environment-variables) above) in your `azd` environment so they are injected when running and deploying:

```bash
azd env set AZURE_AI_PROJECT_ENDPOINT "https://<your-foundry-account>.services.ai.azure.com/api/projects/<your-project>"
azd env set BING_GROUNDING_CONNECTION_ID "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.CognitiveServices/accounts/<foundry-account>/projects/<project>/connections/<bing-connection-name>"
```

Start the agent locally:

```bash
azd ai agent run
```

The agent starts on `http://localhost:8088/`.

**Invoke**

**Bash:**
```bash
azd ai agent invoke --local '{"input": "What is the latest news in AI?"}'
```

**PowerShell:**
```powershell
azd ai agent invoke --local '{\"input\": \"What is the latest news in AI?\"}'
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
azd ai agent invoke '{"input": "What is the latest news in AI?"}'
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

```bash
pip install -r requirements.txt
```

### Running the Sample

To run the agent, execute the following command in your terminal:

```bash
python main.py
```

This will start the hosted agent locally on `http://localhost:8088/`.

### Interacting with the Agent

```bash
curl -X POST http://localhost:8088/responses \
  -H "Content-Type: application/json" \
  -d '{"input": "What is the latest news in AI?"}' | jq .
```

### Deploying the Agent to Microsoft Foundry

To deploy your agent to Microsoft Foundry, follow the comprehensive deployment guide at https://aka.ms/azdaiagent/docs

</details>

## Troubleshooting

### Images built on Apple Silicon or other ARM64 machines do not work on our service

We **recommend using `azd` cloud build**, which always builds images with the correct architecture.

If you choose to **build locally**, and your machine is **not `linux/amd64`** (for example, an Apple Silicon Mac), the image will **not be compatible with our service**, causing runtime failures.

**Fix for local builds**

Use this command to build the image locally:

```shell
docker build --platform=linux/amd64 -t image .
```

This forces the image to be built for the required `amd64` architecture.