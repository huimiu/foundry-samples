# Foundry Memory RAG Agent (Responses Protocol)

A personal coach agent hosted on Microsoft Foundry using the [Agent Framework](https://github.com/microsoft/agent-framework), `FoundryMemoryProvider`, and the **Responses protocol**. It remembers training goals, dietary preferences, injuries, equipment, and scheduling constraints across requests by grounding responses in a Foundry Memory store.

## How It Works

1. `Program.cs` loads local `.env` values via `DotNetEnv`, then reads the Foundry project endpoint, chat deployment, embedding deployment, and memory store name
2. An `AIProjectClient` authenticates with `DefaultAzureCredential`
3. `FoundryMemoryProvider` is configured with the memory store and a sample `FoundryMemoryProviderScope("foundry-memory-rag-user")`
4. `EnsureMemoryStoreCreatedAsync` creates the Foundry Memory store on startup if it does not already exist
5. `.AsAIAgent(new ChatClientAgentOptions { ... AIContextProviders = [memoryProvider] })` creates a personal coach that stores and retrieves user memories
6. `AgentHost.CreateBuilder(args)` plus `AddFoundryResponses(agent)` and `MapFoundryResponses()` host the agent behind `POST /responses`
7. The agent starts on `http://localhost:8088/`

See [Program.cs](Program.cs) for the full implementation.

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `FOUNDRY_PROJECT_ENDPOINT` | Yes | Azure AI Foundry project endpoint URL. Auto-injected when hosted — only needed locally |
| `AZURE_AI_MODEL_DEPLOYMENT_NAME` | No | Chat model deployment name. Defaults to `gpt-4.1-mini`. Declared in `agent.yaml` and `agent.manifest.yaml` |
| `AZURE_AI_EMBEDDING_DEPLOYMENT_NAME` | No | Embedding model deployment name. Defaults to `text-embedding-3-small`. Declared in `agent.yaml` and `agent.manifest.yaml` |
| `AZURE_AI_MEMORY_STORE_ID` | No | Foundry Memory store name. Defaults to `foundry-memory-rag-store`; created on startup if missing |
| `APPLICATIONINSIGHTS_CONNECTION_STRING` | No | Enables telemetry. Auto-injected in hosted containers; set manually for local development |

When deployed as a hosted agent, `FOUNDRY_PROJECT_ENDPOINT` and Application Insights are auto-injected by the platform. Authentication uses Managed Identity via `DefaultAzureCredential`. Locally, the sample loads a `.env` file via `DotNetEnv` if present.

## Running Locally

### Prerequisites

- .NET 10.0 SDK or later (`dotnet --version`)
- An Azure AI Foundry project with chat and embedding model deployments (or let `azd provision` create them)
- Foundry Memory support in the project region and identity permissions to create/use the memory store

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
azd ai agent invoke --local "Remember that I want to run my first 5k in October."
```

**PowerShell:**
```powershell
azd ai agent invoke --local "Remember that I want to run my first 5k in October."
```

**Test with curl:**

```bash
curl -sS -X POST http://localhost:8088/responses \
  -H "Content-Type: application/json" \
  -d '{"input": "Remember that I want to run my first 5k in October.", "stream": false}' | jq .
```

<details>
<summary><h3>Using Foundry Toolkit VS Code Extension</h3></summary>

Open the **Agent Inspector** directly from the Foundry Toolkit extension to invoke the agent — no `curl` or CLI commands needed.

1. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Open Agent Inspector`.
2. The Inspector auto-connects to your running agent at `http://localhost:8088/`.
3. Type a message and send it. The Agent Inspector handles the Responses protocol and displays the reply inline.

> Multi-turn conversation is supported — the Inspector maintains session context across messages.

</details>

## Memory Behavior

The sample uses one shared memory scope, `foundry-memory-rag-user`, because the current hosting package does not yet expose per-user isolation headers to application code. Production agents should partition memory per end user once `HostedSessionContext` support is available; the comment near `stateInitializer` in [Program.cs](Program.cs) shows the intended replacement pattern.

Memory extraction and retrieval are asynchronous server-side. After a teaching turn, wait a few seconds before asking the agent to recall what it knows:

```bash
azd ai agent invoke --local "I have a sensitive left knee, please avoid high-impact exercises."
azd ai agent invoke --local "What do you already know about my training goals?"
```

The memory store name is configurable with `AZURE_AI_MEMORY_STORE_ID`; if unset, startup creates or reuses `foundry-memory-rag-store`.

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
azd ai agent invoke "Remember that I want to run my first 5k in October."
```

**PowerShell:**
```powershell
azd ai agent invoke "Remember that I want to run my first 5k in October."
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
