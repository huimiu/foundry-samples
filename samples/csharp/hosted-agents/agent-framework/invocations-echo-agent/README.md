# Echo Agent (Invocations Protocol)

A minimal echo agent hosted on Microsoft Foundry using the **Invocations protocol** and the [Agent Framework](https://github.com/microsoft/agent-framework). It reads the request body, passes it through a custom `EchoAIAgent`, and writes the echoed text back. No LLM or Azure credentials are required — making it ideal for testing the hosting infrastructure in isolation.

## How It Works

1. `Program.cs` registers a custom `EchoAIAgent` and hosts it behind the Invocations protocol
2. A `POST /invocations` request with a JSON body containing a `"message"` field is routed to the agent
3. The agent echoes the input back as `"Echo: <input>"` — no model is involved
4. Session continuity is supported via the `x-agent-session-id` response header / `agent_session_id` query parameter
5. The agent starts on `http://localhost:8088/`

See [Program.cs](Program.cs) and [EchoAIAgent.cs](EchoAIAgent.cs) for the full implementation.

## Environment Variables

This agent does **not** require a model deployment — no `FOUNDRY_PROJECT_ENDPOINT` or `AZURE_AI_MODEL_DEPLOYMENT_NAME` is needed.

| Variable | Required | Description |
|----------|----------|-------------|
| `APPLICATIONINSIGHTS_CONNECTION_STRING` | Optional | Enables telemetry. Auto-injected in hosted containers; set manually for local dev |

## Running Locally

### Prerequisites

- .NET 10.0 SDK or later (`dotnet --version`)
- No Foundry project or model deployment required

### Using `azd` (Recommended)

Start the agent locally with the `run` command — `azd` restores dependencies, sets environment variables, builds, and starts the agent:

```bash
azd ai agent run
```

The agent starts on `http://localhost:8088/`.

<details>
<summary><h3>Using the Foundry Toolkit VS Code Extension</h3></summary>

The [Foundry Toolkit VS Code extension](https://learn.microsoft.com/en-us/azure/foundry/agents/quickstarts/quickstart-hosted-agent?view=foundry&pivots=vscode) has a built-in sample gallery that scaffolds this project directly into a new workspace — no manual cloning needed.

1. It's recommended to scaffold the project using the Foundry Toolkit extension. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Create new Hosted Agent`. The extension automatically creates the VS Code debug configuration files and `.env`.
2. Edit `.env` and fill in the required environment variables (see [Environment Variables](#environment-variables) above for the full list).
3. Restore dependencies:
   ```bash
   dotnet restore
   ```
4. Press **F5** to start the agent in debug mode. The agent starts on `http://localhost:8088/`.

</details>
<details>
<summary><h3>Manual setup</h3></summary>

No environment variables are required. Run the agent directly:

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
azd ai agent invoke --local '{"message": "Hello, world!"}'
```

**PowerShell:**
```powershell
azd ai agent invoke --local '{\"message\": \"Hello, world!\"}'
```

**Test with curl:**

```bash
curl -X POST http://localhost:8088/invocations -i \
  -H "Content-Type: application/json" \
  -d '{"message": "Hello, world!"}'
```

The `-i` flag includes the HTTP response headers, which carry the session ID for multi-turn conversations:

```
HTTP/1.1 200
content-type: application/json
x-agent-session-id: 9370b9d4-cd13-4436-a57f-03b843ac0e17

{"response":"Echo: Hello, world!"}
```

**Multi-turn:** reuse the session ID from the previous response in the next request's query string:

```bash
curl -X POST "http://localhost:8088/invocations?agent_session_id=9370b9d4-cd13-4436-a57f-03b843ac0e17" -i \
  -H "Content-Type: application/json" \
  -d '{"message": "How are you?"}'
```

<details>
<summary><h3>Using Foundry Toolkit VS Code Extension</h3></summary>

Open the **Agent Inspector** directly from the Foundry Toolkit extension to invoke the agent — no `curl` or CLI commands needed.

1. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Open Agent Inspector`.
2. The Inspector auto-connects to your running agent at `http://localhost:8088/`.
3. Type a message and send it. The Agent Inspector handles the protocol and displays the response inline.

> Multi-turn conversation is supported — the Inspector maintains session context across messages.

</details>

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
azd ai agent invoke '{"message": "Hello, world!"}'
```

**PowerShell:**
```powershell
azd ai agent invoke '{\"message\": \"Hello, world!\"}'
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

We **recommend deploying with `azd deploy`**, which uses ACR remote build and always produces images with the correct architecture.

If you choose to **build locally**, and your machine is **not `linux/amd64`** (for example, an Apple Silicon Mac), the image will **not be compatible with our service**, causing runtime failures.

**Fix for local builds:**

```bash
docker build --platform=linux/amd64 -t image .
```

This forces the image to be built for the required `amd64` architecture.
