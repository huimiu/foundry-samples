**IMPORTANT!** All samples and other resources made available in this GitHub repository ("samples") are designed to assist in accelerating development of agents, solutions, and agent workflows for various scenarios. Review all provided resources and carefully test output behavior in the context of your use case. AI responses may be inaccurate and AI actions should be monitored with human oversight. Learn more in the transparency documents for [Agent Service](https://learn.microsoft.com/en-us/azure/ai-foundry/responsible-ai/agents/transparency-note) and [Agent Framework](https://github.com/microsoft/agent-framework/blob/main/TRANSPARENCY_FAQ.md).

Agents, solutions, or other output you create may be subject to legal and regulatory requirements, may require licenses, or may not be suitable for all industries, scenarios, or use cases. By using any sample, you are acknowledging that any output created using those samples are solely your responsibility, and that you will comply with all applicable laws, regulations, and relevant safety standards, terms of service, and codes of conduct.

Third-party samples contained in this folder are subject to their own designated terms, and they have not been tested or verified by Microsoft or its affiliates.

Microsoft has no responsibility to you or others with respect to any of these samples or any resulting output.

# What this sample demonstrates

This sample demonstrates a **C# System Utility Agent** hosted using the Azure AI AgentServer SDK.

It shows:

- How to host a custom agent by implementing `IAgentInvocation`
- How to emit OpenAI Responses-compatible REST responses (streaming and non-streaming)
- How to add **custom spans** into hosted agent traces (via `ActivitySource`)
- How to call **Azure OpenAI** (Responses API) and use the model to decide which tool(s) to invoke (tool-calling loop)
- A set of cross-platform “system utility” actions (processes, ports, DNS, env vars) that are container-aware

## How It Works

### System Utility Agent

This agent is designed for diagnostics and “what’s running here?” questions. It is **container-aware** and will report whether the agent is likely seeing the container namespace vs the host.

The agent exposes a small set of utility actions and returns results as JSON.

At runtime, the agent:

1. Sends the user request to Azure OpenAI (Responses API) along with the available tool definitions
2. Executes any tool calls the model requests
3. Feeds tool results back to the model as `function_call_output`
4. Repeats until the model returns a final assistant message

Actions supported:

1. **capability_report** - Report what the agent can likely observe (host vs container scope)
2. **system_info** - OS / runtime metadata
3. **resource_snapshot** - Process + GC + disk snapshot (best-effort cross-platform)
4. **list_processes** - List running processes (visibility depends on container scope)
5. **process_details** - Get details for a specific PID
6. **check_port** - Check whether a TCP port is reachable
7. **dns_lookup** - Resolve a hostname
8. **list_environment_variables** - List environment variables (supports redaction)

### Tracing (custom spans)

This sample demonstrates how to add **custom spans** to hosted agent traces using .NET `ActivitySource`.

Spans are created for:

- Each tool-calling iteration of the agent loop (`SystemUtilityAgent.agent_run_iteration`)
- Each tool call execution (`SystemUtilityAgent.tool_call_execution`), including tags such as:
  - `gen_ai.tool.name`
  - `gen_ai.tool.call.arguments` (truncated)
  - `gen_ai.tool.call.result`

The hosted endpoint also emits the standard AgentServer request spans for the overall invocation.

### Agent Hosting

The agent is hosted using the Azure AI AgentServer SDK, which provisions a REST API endpoint compatible with the OpenAI Responses protocol.

## Running the Agent Locally

### Prerequisites

Before running this sample, ensure you have:

1. .NET 8 SDK installed
2. An Azure OpenAI endpoint configured
3. A deployment of a chat model (e.g., `gpt-4o-mini`)

### Environment Variables

Set the following environment variables:

- `AZURE_OPENAI_ENDPOINT` - Your Azure OpenAI endpoint URL (required)
- `AZURE_AI_MODEL_DEPLOYMENT_NAME` - The Azure OpenAI model deployment name (optional, defaults to `gpt-4o-mini`)

Authentication (choose one):

- `AZURE_OPENAI_API_KEY` for key-based auth, OR
- `az login` so `DefaultAzureCredential` can acquire a token

Optional:

- `OPENAI_API_VERSION` - Defaults to `2025-03-01-preview`

```powershell
$env:AZURE_OPENAI_ENDPOINT="https://your-openai-resource.openai.azure.com/"
$env:AZURE_AI_MODEL_DEPLOYMENT_NAME="gpt-4o-mini"

# Option A: key-based
$env:AZURE_OPENAI_API_KEY="<your key>"

# Option B: AAD-based
# az login

# Optional
$env:OPENAI_API_VERSION="2025-03-01-preview"
```

### Using `azd` (Recommended)

The [Azure Developer CLI `ai agent` extension](https://aka.ms/azdaiagent/docs) is the quickest way to run, invoke, and deploy this hosted agent.

**Run locally**

Set the required environment variables (see [Environment Variables](#environment-variables) above) in your `azd` environment so they are injected when running and deploying:

```bash
azd env set AZURE_AI_PROJECT_ENDPOINT "<your-value>"
```

Start the agent locally:

```bash
azd ai agent run
```

The agent starts on `http://localhost:8088/`.

**Invoke**

**Bash:**
```bash
azd ai agent invoke --local '{"input": "What environment are you running in? Summarize what you can observe"}'
```

**PowerShell:**
```powershell
azd ai agent invoke --local '{\"input\": \"What environment are you running in? Summarize what you can observe\"}'
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
azd ai agent invoke '{"input": "What environment are you running in? Summarize what you can observe"}'
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

From the project folder:

```powershell
cd samples/csharp/hosted-agents/AgentWithCustomFramework/SystemUtilityAgent

dotnet run
```

By default, the AgentServer host listens on `http://localhost:8088`.

### Interacting with the Agent

You can interact with the agent workflow using:

- The `run-requests.http` file in this directory to test and prompt the agent
- Any OpenAI Responses compatible client by sending requests to `http://localhost:8080/`

### Deploying the Agent to Microsoft Foundry

To deploy your agent to Microsoft Foundry, follow the comprehensive deployment guide at https://aka.ms/azdaiagent/docs


</details>
## Troubleshooting

### After deployed, the agent appears stateless (chat history is not preserved)

This agent will remain stateless before Azure.AI.Projects supports conversation client.

### Images built on Apple Silicon or other ARM64 machines do not work on our service

We **recommend using `azd` cloud build**, which always builds images with the correct architecture.

If you choose to **build locally**, and your machine is **not `linux/amd64`** (for example, an Apple Silicon Mac), the image will **not be compatible with our service**, causing runtime failures.

**Fix for local builds**

Use this command to build the image locally:

```shell
docker build --platform=linux/amd64 -t image .
```

This forces the image to be built for the required `amd64` architecture.
