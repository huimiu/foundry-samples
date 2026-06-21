# Agent Skills (Responses Protocol)

A Contoso Outdoors customer-support agent hosted on Microsoft Foundry using the [Agent Framework](https://github.com/microsoft/agent-framework) and the **Responses protocol**. It downloads behavioral guidelines from Foundry Skills at startup and exposes them to the model through `AgentSkillsProvider` progressive disclosure.

## How It Works

1. `Program.cs` loads local `.env` values via `DotNetEnv`, reads `FOUNDRY_PROJECT_ENDPOINT`, `AZURE_AI_MODEL_DEPLOYMENT_NAME`, and optional `SKILL_NAMES`, then creates an `AIProjectClient`
2. If `PROVISION_SAMPLE_SKILLS=true`, the sample convenience helper uploads matching local `skills/*/SKILL.md` folders to Foundry when they do not already exist
3. The agent downloads each named Foundry skill into a runtime `downloaded_skills/<name>/` folder and validates that every archive contains a root `SKILL.md`
4. `AgentSkillsProvider` advertises skill names/descriptions in the system prompt and lets the model call `load_skill` only when a full skill body is needed
5. `.AsAIAgent(new ChatClientAgentOptions { ... AIContextProviders = [skillsProvider] })` creates the Contoso Outdoors support assistant
6. `AgentHost.CreateBuilder(args)` plus `AddFoundryResponses(agent)` and `MapFoundryResponses()` host the agent behind `POST /responses`
7. The agent starts on `http://localhost:8088/`

See [Program.cs](Program.cs) and the source skills under [skills/](skills/) for the full implementation.

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `FOUNDRY_PROJECT_ENDPOINT` | Yes | Azure AI Foundry project endpoint URL. Auto-injected when hosted — only needed locally |
| `AZURE_AI_MODEL_DEPLOYMENT_NAME` | Yes | Model deployment name. Declared in `agent.yaml` and `agent.manifest.yaml` |
| `SKILL_NAMES` | No | Comma-separated Foundry skill names to download at startup. If empty, the agent still starts without skills |
| `PROVISION_SAMPLE_SKILLS` | No | Sample convenience flag. Set to `true` on a first run to upload this sample's `SKILL.md` files before downloading them |
| `APPLICATIONINSIGHTS_CONNECTION_STRING` | No | Enables telemetry. Auto-injected in hosted containers; set manually for local development |

When deployed as a hosted agent, `FOUNDRY_PROJECT_ENDPOINT` and Application Insights are auto-injected by the platform. Authentication uses Managed Identity via `DefaultAzureCredential`. Locally, the sample loads a `.env` file via `DotNetEnv` if present.

## Running Locally

### Prerequisites

- .NET 10.0 SDK or later (`dotnet --version`)
- An Azure AI Foundry project with a model deployment (or let `azd provision` create one)
- Azure AI User access on the Foundry project when creating or downloading skills

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
azd ai agent invoke --local "Hi, I am Alex. Can I return my tent within 30 days?"
```

**PowerShell:**
```powershell
azd ai agent invoke --local "Hi, I am Alex. Can I return my tent within 30 days?"
```

**Test with curl:**

```bash
curl -sS -X POST http://localhost:8088/responses \
  -H "Content-Type: application/json" \
  -d '{"input": "Hi, I am Alex. Can I return my tent within 30 days?", "stream": false}' | jq .
```

<details>
<summary><h3>Using Foundry Toolkit VS Code Extension</h3></summary>

Open the **Agent Inspector** directly from the Foundry Toolkit extension to invoke the agent — no `curl` or CLI commands needed.

1. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Open Agent Inspector`.
2. The Inspector auto-connects to your running agent at `http://localhost:8088/`.
3. Type a message and send it. The Agent Inspector handles the Responses protocol and displays the reply inline.

> Multi-turn conversation is supported — the Inspector maintains session context across messages.

</details>

## Working with Foundry Skills

This sample ships two source skills under [skills/](skills/) for onboarding convenience:

| Skill | Purpose | Canary |
|-------|---------|--------|
| [`support-style`](skills/support-style/SKILL.md) | Voice, formatting, and signature rules for Contoso Outdoors support replies | `STYLE-CANARY-3318` |
| [`escalation-policy`](skills/escalation-policy/SKILL.md) | When and how to escalate customer-support tickets | `ESC-CANARY-7742` |

Set `SKILL_NAMES=support-style,escalation-policy` to download both skills at startup. On a first sample run, set `PROVISION_SAMPLE_SKILLS=true` to upload the local `SKILL.md` folders if they are missing in Foundry. In production, provision skills outside the hosted agent, such as in CI/CD or a management workflow.

Foundry Skills preview APIs require the `Foundry-Features: Skills=V1Preview` header. `Program.cs` adds that header through a small `FoundryFeaturesPolicy` on `AgentAdministrationClient` for uploads, lookups, and downloads.

The `name` and `description` values in each skill's YAML front matter are intentionally unquoted. The sample README previously called this out because quoted values can fail import on the preview Skills REST API.

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
azd ai agent invoke "Hi, I am Alex. Can I return my tent within 30 days?"
```

**PowerShell:**
```powershell
azd ai agent invoke "Hi, I am Alex. Can I return my tent within 30 days?"
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
