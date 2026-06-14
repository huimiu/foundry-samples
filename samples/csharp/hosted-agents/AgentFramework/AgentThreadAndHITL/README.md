**IMPORTANT!** All samples and other resources made available in this GitHub repository ("samples") are designed to assist in accelerating development of agents, solutions, and agent workflows for various scenarios. Review all provided resources and carefully test output behavior in the context of your use case. AI responses may be inaccurate and AI actions should be monitored with human oversight. Learn more in the transparency documents for [Agent Service](https://learn.microsoft.com/en-us/azure/ai-foundry/responsible-ai/agents/transparency-note) and [Agent Framework](https://github.com/microsoft/agent-framework/blob/main/TRANSPARENCY_FAQ.md).

Agents, solutions, or other output you create may be subject to legal and regulatory requirements, may require licenses, or may not be suitable for all industries, scenarios, or use cases. By using any sample, you are acknowledging that any output created using those samples are solely your responsibility, and that you will comply with all applicable laws, regulations, and relevant safety standards, terms of service, and codes of conduct.

Third-party samples contained in this folder are subject to their own designated terms, and they have not been tested or verified by Microsoft or its affiliates.

Microsoft has no responsibility to you or others with respect to any of these samples or any resulting output.

# What this sample demonstrates

This sample demonstrates how to use AI agents as executors within a workflow, hosted using
[Azure AI AgentServer SDK](https://learn.microsoft.com/en-us/dotnet/api/overview/azure/ai.agentserver.agentframework-readme) and
deploy it to Microsoft Foundry using the Azure Developer CLI [ai agent](https://aka.ms/azdaiagent/docs) extension.

## How It Works

### Weather Assistant Agent

This sample demonstrates the integration of AI agents with a function tool involving human approval.

The agents are connected sequentially in a workflow, creating a translation chain that demonstrates:

- How AgentServer adapter use a ThreadRepository to manage conversation history.
- The AgentServer adapter converts human approval request to a FunctionCall with function name `__hosted_agent_adapter_hitl__`
- User approve or deny the request by responding `approved` or `denied` as a FunctionCallOutput.

### Agent Hosting

The agent workflow is hosted using the [Azure AI AgentServer SDK](https://learn.microsoft.com/en-us/dotnet/api/overview/azure/ai.agentserver.agentframework-readme),
which provisions a REST API endpoint compatible with the OpenAI Responses protocol. This allows interaction with the agent workflow using OpenAI Responses compatible clients.

### Agent Deployment

The hosted agent workflow can be seamlessly deployed to Microsoft Foundry using the Azure Developer CLI [ai agent](https://aka.ms/azdaiagent/docs) extension.
The extension builds a container image for the agent, deploys it to Azure Container Instances (ACI), and creates a hosted agent version and deployment on Foundry Agent Service.

## Running the Agent Locally

### Prerequisites

Before running this sample, ensure you have:

1. An Azure OpenAI endpoint configured
2. A deployment of a chat model (e.g., `gpt-4o-mini`)
3. Azure CLI installed and authenticated (`az login`)
4. .NET 9.0 SDK or later installed

### Environment Variables

Set the following environment variables:

- `AZURE_OPENAI_ENDPOINT` - Your Azure OpenAI endpoint URL (required)
- `AZURE_OPENAI_DEPLOYMENT_NAME` - The deployment name for your chat model (optional, defaults to `gpt-4o-mini`)

**PowerShell:**

```powershell
# Replace with your Azure OpenAI endpoint
$env:AZURE_OPENAI_ENDPOINT="https://your-openai-resource.openai.azure.com/"

# Optional, defaults to gpt-4o-mini
$env:AZURE_OPENAI_DEPLOYMENT_NAME="gpt-4o-mini"
```

### Using `azd` (Recommended)

The [Azure Developer CLI `ai agent` extension](https://aka.ms/azdaiagent/docs) is the quickest way to run, invoke, and deploy this hosted agent.

**Run locally**

Set the required environment variables (see [Environment Variables](#environment-variables) above) in your `azd` environment so they are injected when running and deploying:

```bash
azd env set AZURE_OPENAI_ENDPOINT "https://your-openai-resource.openai.azure.com/"
```

Start the agent locally:

```bash
azd ai agent run
```

The agent starts on `http://localhost:8088/`.

**Invoke**

**Bash:**
```bash
azd ai agent invoke --local '{"input": "What is the weather like in Vancouver?"}'
```

**PowerShell:**
```powershell
azd ai agent invoke --local '{\"input\": \"What is the weather like in Vancouver?\"}'
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
azd ai agent invoke '{"input": "What is the weather like in Vancouver?"}'
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

1. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Create new Hosted Agent`. The extension creates the VS Code debug configuration files.
2. Set the required environment variables (see [Environment Variables](#environment-variables) above).
3. Restore dependencies:
   ```bash
   dotnet restore
   ```
4. Build the project:
   ```bash
   dotnet build
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

### Running the Sample

To run the agent, execute the following command in your terminal:

```powershell
dotnet run
```

This will start the hosted agent workflow locally on `http://localhost:8088/`.

### Interacting with the Agent

You can interact with the agent workflow using:

- The `test_requests.py` file in this directory to test and prompt the agent. It now generates a valid `conversation.id` automatically and reuses it across turns.
- Any OpenAI Responses compatible client by sending requests to `http://localhost:8088/`. For this HITL sample, you must provide a stable `conversation.id` in every request (including the first one) to keep thread state and pending approvals.

Try providing text to ask the weather assistant agent about the weather in a city.

### Deploying the Agent to Microsoft Foundry

To deploy your agent to Microsoft Foundry, follow the comprehensive deployment guide at https://aka.ms/azdaiagent/docs

</details>

## Troubleshooting

### Images built on Apple Silicon or other ARM64 machines do not work on our service

We **recommend using `azd` cloud build**, which always builds images with the correct architecture.

If you choose to **build locally**, and your machine is **not `linux/amd64`** (for example, an Apple Silicon Mac), the image will **not be compatible with our service**, causing runtime failures.

**Fix for local builds**

Add this line at the top of your `Dockerfile`:

```dockerfile
FROM --platform=linux/amd64 python:3.12-slim
```

This forces the image to be built for the required `amd64` architecture.
