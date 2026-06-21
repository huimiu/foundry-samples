# Teams Activity Agent (Responses Protocol)

An Agent Framework hosted agent that can be deployed to Microsoft Foundry, connected to Work IQ toolbox tools, and published to Teams. It uses the **Responses protocol** for local and hosted invocation.

## How It Works

1. `Program.cs` loads a local `.env` file with `DotNetEnv` when present, then reads the Foundry project endpoint, model deployment, and toolbox name
2. `AIProjectClient.AsAIAgent(...)` creates a concise assistant backed by the configured Foundry model
3. `AgentHost.CreateBuilder(args)` plus `AddFoundryResponses(agent)` expose the agent behind `POST /responses`
4. `AddFoundryToolboxes(toolboxName)` registers the Foundry Toolbox so Work IQ Teams and calendar MCP tools can be discovered through the Foundry toolbox proxy
5. The toolbox and its Teams/calendar connections are declared in [`agent.manifest.yaml`](agent.manifest.yaml)
6. The agent starts on `http://localhost:8088/`

See [Program.cs](Program.cs) for the full implementation.

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `FOUNDRY_PROJECT_ENDPOINT` | Yes | Azure AI Foundry project endpoint URL. Auto-injected when hosted — only needed locally |
| `AZURE_AI_MODEL_DEPLOYMENT_NAME` | Yes | Model deployment name. Declared in `agent.yaml` and created from `agent.manifest.yaml` when using `azd provision` |
| `TOOLBOX_NAME` | Yes | Foundry Toolbox name to load. Defaults to `teams-tools` in `.env.example` and `agent.manifest.yaml` |
| `APPLICATIONINSIGHTS_CONNECTION_STRING` | No | Enables local telemetry when set. Auto-injected in hosted containers |

When deployed as a hosted agent, `FOUNDRY_PROJECT_ENDPOINT` and Application Insights settings are auto-injected by the platform. Authentication uses Managed Identity via `DefaultAzureCredential`. Locally, the sample loads a `.env` file via `DotNetEnv` if present.

## Running Locally

### Prerequisites

- .NET 10.0 SDK or later (`dotnet --version`)
- An Azure AI Foundry project with a model deployment and the `teams-tools` toolbox (or let `azd provision` create them)
- Azure CLI authentication for manual local runs (`az login`)
- Microsoft Teams and Microsoft 365 access for end-user sign-in after publishing

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

Set the environment variables from [Environment Variables](#environment-variables) (or place them in a `.env` file, which the sample loads via `DotNetEnv`), then:

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
azd ai agent invoke --local "How many meetings do I have tomorrow?"
```

**PowerShell:**
```powershell
azd ai agent invoke --local "How many meetings do I have tomorrow?"
```

**Test with curl:**

```bash
curl -sS -X POST http://localhost:8088/responses \
  -H "Content-Type: application/json" \
  -d '{"input": "How many meetings do I have tomorrow?", "stream": false}' | jq .
```

<details>
<summary><h3>Using Foundry Toolkit VS Code Extension</h3></summary>

Open the **Agent Inspector** directly from the Foundry Toolkit extension to invoke the agent — no `curl` or CLI commands needed.

1. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Open Agent Inspector`.
2. The Inspector auto-connects to your running agent at `http://localhost:8088/`.
3. Type a message and send it. The Agent Inspector handles the protocol and displays the response inline.

> Multi-turn conversation is supported — the Inspector maintains session context across messages.

</details>

## Publishing the Agent

After deployment, publish the agent to Teams and Microsoft 365 from the Foundry portal. Choose **Publish**, then **Publish to Teams and Microsoft 365**; the portal creates the Azure Bot resource and configures the messaging endpoint automatically. For the full flow, see [Publish a Foundry agent as a copilot](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/publish-copilot).

End users sign in the first time they use the agent. Once signed in, they can send messages with file attachments and ask questions about Teams and calendar data through the Work IQ toolbox tools.

![Using Work IQ tool](./teams-activity.png)

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
azd ai agent invoke "How many meetings do I have tomorrow?"
```

**PowerShell:**
```powershell
azd ai agent invoke "How many meetings do I have tomorrow?"
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

### Teams or calendar data is unavailable

Confirm the `teams-tools` toolbox and its Work IQ connections were created from `agent.manifest.yaml`, then have the end user sign in through Teams after publishing. Without sign-in, Teams and calendar tools cannot access user data.
