**IMPORTANT!** All samples and other resources made available in this GitHub repository ("samples") are designed to assist in accelerating development of agents, solutions, and agent workflows for various scenarios. Review all provided resources and carefully test output behavior in the context of your use case. AI responses may be inaccurate and AI actions should be monitored with human oversight.

# Note-Taking Agent (Responses Protocol)

A bring-your-own hosted note-taking agent in C# using the **Responses protocol** and `Azure.AI.AgentServer.Responses`. It uses Foundry Responses API function calling to save and retrieve session-scoped notes stored in local JSONL files.

## How It Works

1. `Program.cs` creates a Foundry `ProjectResponsesClient` and starts `ResponsesServer.Run<NoteTakingHandler>()` on `http://localhost:8088/`.
2. `POST /responses` input is read with `ResponseContext.GetInputTextAsync()`.
3. The model can call two local tools: `save_note` writes a note, and `get_notes` retrieves notes for the current session.
4. [NoteStore.cs](NoteStore.cs) stores notes in `$HOME` as `notes_{session_id}.jsonl`; the session ID comes from `agent_session_id` in the Responses request body, or defaults to `default`.
5. `TextResponse` streams the final natural-language reply through the Responses protocol.

See [Program.cs](Program.cs) and [NoteStore.cs](NoteStore.cs) for the full implementation.

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
azd ai agent invoke --local "save a note - book reservation for dinner"
```

**PowerShell:**
```powershell
azd ai agent invoke --local "save a note - book reservation for dinner"
```

**Test with curl:**

```bash
curl -sS -X POST http://localhost:8088/responses \
  -H "Content-Type: application/json" \
  -d '{"input": "save a note - book reservation for dinner", "stream": false}' | jq .
```

<details>
<summary><h3>Using Foundry Toolkit VS Code Extension</h3></summary>

Open the **Agent Inspector** directly from the Foundry Toolkit extension to invoke the agent — no `curl` or CLI commands needed.

1. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Open Agent Inspector`.
2. The Inspector auto-connects to your running agent at `http://localhost:8088/`.
3. Type a message and send it. The Agent Inspector handles the protocol and displays the response inline.

> Multi-turn conversation is supported — the Inspector maintains session context across messages.

</details>

## Note Storage and Sessions

Include `agent_session_id` in the Responses request body to keep notes scoped to the same session:

```bash
curl -X POST http://localhost:8088/responses \
  -H "Content-Type: application/json" \
  -d '{"input": "save a note - book reservation for dinner", "agent_session_id": "my-session"}' \
  --no-buffer

curl -X POST http://localhost:8088/responses \
  -H "Content-Type: application/json" \
  -d '{"input": "get all my notes", "agent_session_id": "my-session"}' \
  --no-buffer
```

For `azd`, run commands from this sample directory so the local session state is persisted across calls. Use a new session or clear the conversation in Agent Inspector to start a separate note file.

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
azd ai agent invoke "save a note - book reservation for dinner"
```

**PowerShell:**
```powershell
azd ai agent invoke "save a note - book reservation for dinner"
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

### Azure OpenAI Permission Denied (401)

If the model call returns a permission error, the identity running the agent likely lacks the required roles on the Azure AI Foundry project. Assign the following roles:

- **Cognitive Services OpenAI User**
- **Foundry User**

> It may take a few minutes for role assignments to propagate. Retry the request after waiting.
