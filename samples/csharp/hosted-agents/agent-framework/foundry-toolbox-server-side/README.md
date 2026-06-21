# Foundry Toolbox Server-Side Tools Agent (Responses Protocol)

An Agent Framework hosted agent that loads a Foundry Toolbox and exposes it through the **Responses protocol**. The C# host fetches toolbox tool definitions from Microsoft Foundry, while the Foundry platform executes tool calls server-side on the agent's behalf.

## How It Works

1. `Program.cs` loads a local `.env` file with `DotNetEnv` when present, then reads the Foundry project endpoint, model deployment, and toolbox name from environment variables
2. `AIProjectClient` authenticates with `DefaultAzureCredential` and calls `GetToolboxToolsAsync(toolboxName)` to fetch the toolbox's current tool definitions
3. `.AsAIAgent(model, instructions, name, description, tools)` creates an `AIAgent` with those toolbox tools attached
4. `AgentHost.CreateBuilder(args)` plus `AddFoundryResponses(agent)` and `MapFoundryResponses()` host the agent behind the OpenAI-compatible `POST /responses` endpoint
5. The agent process does not broker MCP connections or execute toolbox tools locally — Foundry resolves and invokes the tools server-side
6. The agent starts on `http://localhost:8088/`

See [Program.cs](Program.cs) for the full implementation.

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `FOUNDRY_PROJECT_ENDPOINT` | Yes | Azure AI Foundry project endpoint URL. Auto-injected when hosted — only needed locally |
| `AZURE_AI_MODEL_DEPLOYMENT_NAME` | Yes | Model deployment name. Declared in `agent.yaml` and created from `agent.manifest.yaml` when using `azd provision` |
| `TOOLBOX_NAME` | Yes | Foundry Toolbox name to load. Defaults to `agent-tools` in `.env.example` and `agent.manifest.yaml` |
| `APPLICATIONINSIGHTS_CONNECTION_STRING` | No | Enables local telemetry when set. Auto-injected in hosted containers |

When deployed as a hosted agent, `FOUNDRY_PROJECT_ENDPOINT` and Application Insights settings are auto-injected by the platform. Authentication uses Managed Identity via `DefaultAzureCredential`. Locally, the sample loads a `.env` file via `DotNetEnv` if present.

## Running Locally

### Prerequisites

- .NET 10.0 SDK or later (`dotnet --version`)
- An Azure AI Foundry project with a model deployment and a Foundry Toolbox (or let `azd provision` create them)
- Azure CLI authentication for manual local runs (`az login`)

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
azd ai agent invoke --local "What tools do you have available?"
```

**PowerShell:**
```powershell
azd ai agent invoke --local "What tools do you have available?"
```

**Test with curl:**

```bash
curl -sS -X POST http://localhost:8088/responses \
  -H "Content-Type: application/json" \
  -d '{"input": "What tools do you have available?", "stream": false}' | jq .
```

<details>
<summary><h3>Using Foundry Toolkit VS Code Extension</h3></summary>

Open the **Agent Inspector** directly from the Foundry Toolkit extension to invoke the agent — no `curl` or CLI commands needed.

1. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Open Agent Inspector`.
2. The Inspector auto-connects to your running agent at `http://localhost:8088/`.
3. Type a message and send it. The Agent Inspector handles the protocol and displays the response inline.

> Multi-turn conversation is supported — the Inspector maintains session context across messages.

</details>

## Creating a Foundry Toolbox

You can create a Foundry Toolbox in code using the [Foundry Toolbox CRUD sample](https://github.com/Azure/azure-sdk-for-python/blob/main/sdk/ai/azure-ai-projects/samples/hosted_agents/sample_toolboxes_crud.py), or create one in the Foundry portal by following the [Foundry toolbox documentation](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/tools/toolbox).

If you provision this sample with `azd provision`, the toolbox is created from [`agent.manifest.yaml`](agent.manifest.yaml). By default, it creates an `agent-tools` toolbox with `web_search` and `code_interpreter` tools. Swap either tool for the toolbox tool types that fit your scenario.

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
azd ai agent invoke "What tools do you have available?"
```

**PowerShell:**
```powershell
azd ai agent invoke "What tools do you have available?"
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

### Toolbox tools do not appear or the host fails at startup

`GetToolboxToolsAsync()` enumerates the configured toolbox during startup. Ensure `TOOLBOX_NAME` matches a toolbox in the same Foundry project, that the toolbox sources are reachable, and that the model deployment named by `AZURE_AI_MODEL_DEPLOYMENT_NAME` exists.
