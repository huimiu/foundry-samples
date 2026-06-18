# File Tools Agent (Responses Protocol)

A hosted file-question-answering agent using the [Agent Framework](https://github.com/microsoft/agent-framework) and the **Responses protocol**. It demonstrates two security-scoped tool pairs: one for bundled files packaged with the image and one for per-session files uploaded by the user.

## How It Works

1. `Program.cs` loads local `.env` values via `DotNetEnv`, reads the Foundry project endpoint, and uses `AZURE_AI_MODEL_DEPLOYMENT_NAME` or the `gpt-4.1-mini` default
2. The bundled-files root comes from `BUNDLED_FILES_DIR` or `<process base dir>/resources`; `file-tools.csproj` copies [resources/](resources/) into the publish output
3. The session-files root comes from `HOME` or `/home/session`; files uploaded with `azd ai agent files upload` land in that per-session location
4. Four C# functions are registered as tools: `ListBundledFiles`, `ReadBundledFile`, `ListSessionFiles`, and `ReadSessionFile`
5. `SafeRead` strips path components, canonicalizes paths, and checks that reads stay inside the selected root before returning file contents
6. `AgentHost.CreateBuilder(args)` plus `AddFoundryResponses(agent)` and `MapFoundryResponses()` host the agent behind `POST /responses`
7. The agent starts on `http://localhost:8088/`

See [Program.cs](Program.cs) for the full implementation.

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `FOUNDRY_PROJECT_ENDPOINT` | Yes | Azure AI Foundry project endpoint URL. Auto-injected when hosted — only needed locally |
| `AZURE_AI_MODEL_DEPLOYMENT_NAME` | No | Model deployment name. Defaults to `gpt-4.1-mini`. Declared in `agent.yaml` and `agent.manifest.yaml` |
| `ASPNETCORE_URLS` | No | ASP.NET Core bind address. `.env.example` sets `http://+:8088` |
| `ASPNETCORE_ENVIRONMENT` | No | ASP.NET Core environment. `.env.example` sets `Development` |
| `BUNDLED_FILES_DIR` | No | Override the bundled-files root. Defaults to `<process base dir>/resources` |
| `HOME` | No | Per-session uploaded-files root. Set by the Foundry platform; defaults to `/home/session` if absent |

When deployed as a hosted agent, `FOUNDRY_PROJECT_ENDPOINT` is auto-injected by the platform. Authentication uses Managed Identity via `DefaultAzureCredential`. Locally, the sample loads a `.env` file via `DotNetEnv` if present.

## Running Locally

### Prerequisites

- .NET 10.0 SDK or later (`dotnet --version`)
- An Azure AI Foundry project with a model deployment (or let `azd provision` create one)
- Optional demo files from [resources/](resources/) and [example-upload/](example-upload/) if you want to exercise both file sources

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
azd ai agent invoke --local "What is the headline total revenue in the bundled Contoso report?"
```

**PowerShell:**
```powershell
azd ai agent invoke --local "What is the headline total revenue in the bundled Contoso report?"
```

**Test with curl:**

```bash
curl -sS -X POST http://localhost:8088/responses \
  -H "Content-Type: application/json" \
  -d '{"input": "What is the headline total revenue in the bundled Contoso report?", "stream": false}' | jq .
```

<details>
<summary><h3>Using Foundry Toolkit VS Code Extension</h3></summary>

Open the **Agent Inspector** directly from the Foundry Toolkit extension to invoke the agent — no `curl` or CLI commands needed.

1. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Open Agent Inspector`.
2. The Inspector auto-connects to your running agent at `http://localhost:8088/`.
3. Type a message and send it. The Agent Inspector handles the Responses protocol and displays the reply inline.

> Multi-turn conversation is supported — the Inspector maintains session context across messages.

</details>

## File Sources and Uploads

| Tool | Source | Default root |
|------|--------|--------------|
| `ListBundledFiles` / `ReadBundledFile` | Files shipped with the agent image | `<process base dir>/resources` (`/app/resources` in the container) |
| `ListSessionFiles` / `ReadSessionFile` | Files uploaded by the user for the current session | `$HOME` (`/home/session` in hosted sessions) |

The bundled demo report [resources/contoso_q1_2026_report.txt](resources/contoso_q1_2026_report.txt) contains fictional Contoso Q1 2026 figures, including total revenue of `$1,482.6M`. To test session files, upload the included notes file and then ask about it:

```bash
azd ai agent files upload ./example-upload/user_notes.txt
azd ai agent invoke --local "What magic token is in user_notes.txt?"
```

The uploaded file contains `GREENFIELD-7421`, which lets you verify the session-file round trip.

To add more bundled files, place them under [resources/](resources/). The project file copies `resources\**\*` to the build and publish output so they are available to the bundled-file tools.

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
azd ai agent invoke "What is the headline total revenue in the bundled Contoso report?"
```

**PowerShell:**
```powershell
azd ai agent invoke "What is the headline total revenue in the bundled Contoso report?"
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
