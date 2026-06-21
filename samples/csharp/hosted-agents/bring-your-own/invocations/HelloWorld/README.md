<!-- Begin standard disclaimer — do not modify -->
**IMPORTANT!** All samples and other resources made available in this GitHub repository ("samples") are designed to assist in accelerating development of agents, solutions, and agent workflows for various scenarios. Review all provided resources and carefully test output behavior in the context of your use case. AI responses may be inaccurate and AI actions should be monitored with human oversight. Learn more in the transparency note for [Agent Service](https://learn.microsoft.com/en-us/azure/ai-foundry/responsible-ai/agents/transparency-note).

Agents, solutions, or other output you create may be subject to legal and regulatory requirements, may require licenses, or may not be suitable for all industries, scenarios, or use cases. By using any sample, you are acknowledging that any output created using those samples are solely your responsibility, and that you will comply with all applicable laws, regulations, and relevant safety standards, terms of service, and codes of conduct.

Third-party samples contained in this folder are subject to their own designated terms, and they have not been tested or verified by Microsoft or its affiliates.

Microsoft has no responsibility to you or others with respect to any of these samples or any resulting output.
<!-- End standard disclaimer -->

# Hello World Agent (Invocations Protocol)

A minimal bring-your-own hosted agent in C# using the **Invocations protocol** and `Azure.AI.AgentServer.Invocations`. It forwards user input to a Foundry model through the Responses API and returns a streaming Server-Sent Events (SSE) reply.

## How It Works

1. `Program.cs` starts `InvocationsServer.Run<HelloWorldHandler>()`, which exposes the Invocations protocol endpoint on `http://localhost:8088/`.
2. Startup reads `FOUNDRY_PROJECT_ENDPOINT` and `AZURE_AI_MODEL_DEPLOYMENT_NAME`, creates an `AIProjectClient`, and gets a `ProjectResponsesClient` for the configured model.
3. `POST /invocations` reads the JSON request body. The handler uses the `message` field first, also accepts `input`, and falls back to a plain-text body.
4. Conversation history is kept in an in-memory dictionary keyed by `InvocationContext.SessionId` / `agent_session_id`.
5. The handler calls the Foundry Responses API, streams `token` SSE events, then sends a final `done` event with the invocation ID, session ID, and complete reply.

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
azd ai agent invoke --local '{"message": "What is Microsoft Foundry?"}'
```

**PowerShell:**
```powershell
azd ai agent invoke --local '{\"message\": \"What is Microsoft Foundry?\"}'
```

**Test with curl:**

```bash
curl -N -X POST http://localhost:8088/invocations \
  -H "Content-Type: application/json" \
  -d '{"message": "What is Microsoft Foundry?"}'
```

**SSE Event Format:**

```text
data: {"type":"token","content":"<text delta>"}

data: {"type":"done","invocation_id":"<invocation-id>","session_id":"<session-id>","full_text":"<complete reply>"}
```

<details>
<summary><h3>Using Foundry Toolkit VS Code Extension</h3></summary>

Open the **Agent Inspector** directly from the Foundry Toolkit extension to invoke the agent — no `curl` or CLI commands needed.

1. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Open Agent Inspector`.
2. The Inspector auto-connects to your running agent at `http://localhost:8088/`.
3. Type a message and send it. The Agent Inspector handles the protocol and displays the response inline.

> Multi-turn conversation is supported — the Inspector maintains session context across messages.

</details>

## Multi-Turn Sessions

The Invocations protocol does not provide built-in server-side conversation history. This sample stores history in memory per `agent_session_id`, so state is lost when the process restarts.

```bash
curl -N -X POST "http://localhost:8088/invocations?agent_session_id=chat-001" \
  -H "Content-Type: application/json" \
  -d '{"message": "What is Microsoft Foundry?"}'

curl -N -X POST "http://localhost:8088/invocations?agent_session_id=chat-001" \
  -H "Content-Type: application/json" \
  -d '{"message": "What hosted agent options does it offer?"}'
```

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
azd ai agent invoke '{"message": "What is Microsoft Foundry?"}'
```

**PowerShell:**
```powershell
azd ai agent invoke '{\"message\": \"What is Microsoft Foundry?\"}'
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
