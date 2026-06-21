**IMPORTANT!** All samples and other resources made available in this GitHub repository ("samples") are designed to assist in accelerating development of agents, solutions, and agent workflows for various scenarios. Review all provided resources and carefully test output behavior in the context of your use case. AI responses may be inaccurate and AI actions should be monitored with human oversight.

# Background Agent (Responses Protocol)

A bring-your-own hosted research agent in C# using the **Responses protocol** and `Azure.AI.AgentServer.Responses`. It demonstrates background execution for long-running model work while the SDK handles polling and cancellation.

## How It Works

1. `Program.cs` creates a Foundry `ProjectResponsesClient` and starts `ResponsesServer.Run<BackgroundResearchHandler>()` on `http://localhost:8088/`.
2. The handler receives `POST /responses` requests and reads the user topic with `ResponseContext.GetInputTextAsync()`.
3. For `background: true`, the Responses SDK returns immediately, runs the handler asynchronously, and exposes status through `GET /responses/{id}`.
4. The handler asks the model for a multi-section research analysis and streams text deltas through `TextResponse`.
5. In-flight background responses can be cancelled with `POST /responses/{id}/cancel`; synchronous requests are also supported.

See [Program.cs](Program.cs) for the full implementation.

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `FOUNDRY_PROJECT_ENDPOINT` | Yes (local) | Azure AI Foundry project endpoint URL. Auto-injected when hosted and supplied by `azd ai agent run` locally |
| `AZURE_AI_MODEL_DEPLOYMENT_NAME` | Yes | Model deployment name. Declared in `agent.yaml` / `agent.manifest.yaml` and must match a deployment in your Foundry project |
| `APPLICATIONINSIGHTS_CONNECTION_STRING` | Optional | Enables telemetry. Auto-injected in hosted containers; set manually for local dev |

When deployed as a hosted agent, `FOUNDRY_PROJECT_ENDPOINT` and `APPLICATIONINSIGHTS_CONNECTION_STRING` are auto-injected by the platform. Authentication uses Managed Identity via `DefaultAzureCredential`. Locally, set environment variables directly or use `azd ai agent run`.

## Running Locally

### Prerequisites

- .NET 10.0 SDK or later (`dotnet --version`)
- An Azure AI Foundry project with a model deployment (or let `azd provision` create one)
- Azure authentication available to `DefaultAzureCredential` (for example, `az login`)

### Using `azd`

Start the agent locally with the `run` command — `azd` restores dependencies, sets environment variables, builds, and starts the agent:

```bash
azd ai agent run
```

The agent starts on `http://localhost:8088/`.

<details>
<summary><h3>Using the Foundry Toolkit VS Code Extension</h3></summary>

The [Foundry Toolkit VS Code extension](https://learn.microsoft.com/en-us/azure/foundry/agents/quickstarts/quickstart-hosted-agent?view=foundry&pivots=vscode) sets up a one-click **F5** debug experience for this sample.

1. Scaffold the project using the Foundry Toolkit extension: open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Create new Hosted Agent`. The extension automatically creates the VS Code debug configuration files and `.env`.
2. Edit `.env` and fill in the required environment variables (see [Environment Variables](#environment-variables) above for the full list).
3. Restore dependencies:
   ```bash
   dotnet restore
   ```
4. Press **F5** to start the agent in debug mode. The agent starts on `http://localhost:8088/`.

</details>
<details>
<summary><h3>Manual setup</h3></summary>

Set the environment variables from [Environment Variables](#environment-variables) (.NET does not read `.env` files natively), then:

```bash
dotnet run
```

The agent starts on `http://localhost:8088/`.

</details>

## Invoke

### Using azd

**Local:**

**Bash:**
```bash
azd ai agent invoke --local "Analyze the impact of AI on healthcare"
```

**PowerShell:**
```powershell
azd ai agent invoke --local "Analyze the impact of AI on healthcare"
```

**Test with curl:**

```bash
curl -sS -X POST http://localhost:8088/responses \
  -H "Content-Type: application/json" \
  -d '{"input": "Analyze the impact of AI on healthcare", "stream": false}' | jq .
```

<details>
<summary><h3>Using Foundry Toolkit VS Code Extension</h3></summary>

Open the **Agent Inspector** directly from the Foundry Toolkit extension to invoke the agent — no `curl` or CLI commands needed.

1. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Open Agent Inspector`.
2. The Inspector auto-connects to your running agent at `http://localhost:8088/`.
3. Type a message and send it. The Agent Inspector handles the protocol and displays the response inline.

> Multi-turn conversation is supported — the Inspector maintains session context across messages.

</details>

## Background Mode

Submit a background request, poll for completion, or cancel in-flight work:

```bash
# Submit a background research analysis
curl -X POST http://localhost:8088/responses \
  -H "Content-Type: application/json" \
  -d '{"input": "Analyze the impact of AI on healthcare", "background": true, "store": true}'

# Poll for result using the id from the response
curl http://localhost:8088/responses/<response_id>

# Cancel an in-flight request
curl -X POST http://localhost:8088/responses/<response_id>/cancel
```

For a synchronous response, omit `background: true`.

## Project Structure

| File | Purpose |
|------|---------|
| `Program.cs` | Agent entry point and `ResponseHandler` implementation |
| `background-agent.csproj` | .NET dependencies for the Responses protocol and Foundry SDK |
| `agent.yaml` | Hosted container agent configuration using the Responses protocol |
| `agent.manifest.yaml` | Foundry manifest with model resource and environment-variable mapping |

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
azd ai agent invoke "Analyze the impact of AI on healthcare"
```

**PowerShell:**
```powershell
azd ai agent invoke "Analyze the impact of AI on healthcare"
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
