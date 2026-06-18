# Toolbox Auth Paths Agent (Responses Protocol)

An Agent Framework hosted agent that demonstrates authenticated Foundry Toolbox MCP tools with the **Responses protocol**. The agent code carries no upstream auth logic; Foundry resolves each toolbox connection credential server-side when it proxies tool calls.

## How It Works

1. `Program.cs` loads a local `.env` file with `DotNetEnv` when present, then reads the Foundry project endpoint, model deployment, and toolbox name
2. If `FOUNDRY_AGENT_TOOLSET_ENDPOINT` is absent locally, the sample derives it from the project endpoint so the toolbox MCP proxy can still be reached
3. `AIProjectClient.AsAIAgent(...)` creates a developer assistant backed by the configured Foundry model
4. `AddFoundryResponses(agent)` registers the Responses protocol host, and `AddFoundryToolboxes(options => options.ApiVersion = "v1", toolboxName)` registers the authenticated Foundry Toolbox
5. The hosting layer connects to the Foundry toolbox MCP proxy, discovers the toolbox's GitHub tools, and injects them at request time as host-executed MCP tools
6. The agent starts on `http://localhost:8088/`

See [Program.cs](Program.cs) for the full implementation.

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `FOUNDRY_PROJECT_ENDPOINT` | Yes | Azure AI Foundry project endpoint URL. Auto-injected when hosted — only needed locally |
| `AZURE_AI_MODEL_DEPLOYMENT_NAME` | Yes | Model deployment name. Declared in `agent.yaml` and created from `agent.manifest.yaml` when using `azd provision` |
| `TOOLBOX_NAME` | Yes | Foundry Toolbox name to load. Defaults to `auth-paths-tools` in `.env.example` and `agent.manifest.yaml` |
| `FOUNDRY_AGENT_TOOLSET_ENDPOINT` | No | Toolbox MCP proxy base URL. Auto-injected when hosted; derived locally from `FOUNDRY_PROJECT_ENDPOINT` if absent |
| `APPLICATIONINSIGHTS_CONNECTION_STRING` | No | Enables local telemetry when set. Auto-injected in hosted containers |

The GitHub PAT for the key-based auth path is **not** a container environment variable. It is the `gh_pat` secret parameter in [`agent.manifest.yaml`](agent.manifest.yaml), and `azd` stores it in the Foundry connection secret store.

When deployed as a hosted agent, `FOUNDRY_PROJECT_ENDPOINT`, `FOUNDRY_AGENT_TOOLSET_ENDPOINT`, and Application Insights settings are auto-injected by the platform. Authentication uses Managed Identity via `DefaultAzureCredential`. Locally, the sample loads a `.env` file via `DotNetEnv` if present.

## Running Locally

### Prerequisites

- .NET 10.0 SDK or later (`dotnet --version`)
- An Azure AI Foundry project with a model deployment and the `auth-paths-tools` toolbox (or let `azd provision` create them)
- Azure CLI authentication for manual local runs (`az login`)
- A GitHub Personal Access Token for the key-based `CustomKeys` path, scoped to read the repositories your prompts query

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

## Auth Path Matrix

| # | Path | `authType` | Where the secret lives | Wired by default? |
|---|------|------------|------------------------|-------------------|
| 1 | Key-based | `CustomKeys` | `gh_pat` secret parameter → Foundry connection secret store, injected as `Authorization: Bearer ...` | Yes, for GitHub MCP |
| 2 | Microsoft Entra agent identity | `AgenticIdentity` | No secret — the agent's managed identity gets an Entra token for the target `audience` | Optional, see below |

Do not embed an `Authorization` header inline in the manifest. That commits the token in plain text. Prefer a `CustomKeys` connection so the secret stays in the Foundry connection store.

## Provisioning the Toolbox in Your Environment

The agent reads its toolbox from your Foundry project at startup, so the `auth-paths-tools` toolbox and the `github-mcp-conn` connection must exist before you run.

### Option 1 — `azd provision` (recommended)

`azd provision` reads [`agent.manifest.yaml`](agent.manifest.yaml) and creates the connection and toolbox for you. If your azd environment does not already have the secret parameter, set it before provisioning:

```bash
azd env set gh_pat="..."
azd provision
```

The `gh_pat` value is stored only in the Foundry connection secret store. It is never written to `.env` and is never passed to the container as an environment variable. Use a GitHub PAT (classic `ghp_...` or fine-grained `github_pat_...`) scoped to read the repositories your prompts ask about.

### Option 2 — Create the toolbox yourself

Create the same resources in the [Foundry portal](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/tools/toolbox) or in code with the [Foundry Toolbox CRUD sample](https://github.com/Azure/azure-sdk-for-python/blob/main/sdk/ai/azure-ai-projects/samples/hosted_agents/sample_toolboxes_crud.py):

1. A `CustomKeys` connection named `github-mcp-conn`, target `https://api.githubcopilot.com/mcp`, key `Authorization` = `Bearer <your-pat>`.
2. A toolbox named `auth-paths-tools` with one MCP tool, `github`, referencing `github-mcp-conn`.

Once the toolbox exists, set `TOOLBOX_NAME=auth-paths-tools` and run the agent.

## Adding Microsoft Entra Agent Identity

The `AgenticIdentity` path is documented rather than wired by default because it requires a post-deploy RBAC grant before the toolbox can enumerate it. To add it, extend [`agent.manifest.yaml`](agent.manifest.yaml) with an Entra-protected MCP connection and toolbox entry:

```yaml
parameters:
  properties:
    - name: entra_audience
      secret: false
      description: Entra ID token audience for the target MCP server.
    - name: entra_mcp_target
      secret: false
      description: URL of the Entra-protected MCP server.
resources:
  - kind: connection
    name: entra-agent-conn
    category: RemoteTool
    authType: AgenticIdentity
    audience: "{{ entra_audience }}"
    target: "{{ entra_mcp_target }}"
  - kind: toolbox
    name: auth-paths-tools
    tools:
      - type: mcp
        server_label: entra
        project_connection_id: entra-agent-conn
```

After deployment, assign the agent's managed identity whatever role the target MCP server requires. Until that RBAC grant is in place, the path fails to enumerate and the toolbox host can fail startup.

## Continuous Integration

The `hosted-agents-cloud-e2e` workflow treats this as a toolbox sample and can run with `SKIP_PROVISION=true`, so it may consume a toolbox that already exists in a shared Foundry project. To enable that path, register an `auth-paths-tools` toolbox in the project and add a `label=url|query` line to the `TOOLBOX_ENDPOINT` repository variable:

```text
auth-paths=https://<account>.services.ai.azure.com/api/projects/<project>/toolboxes/auth-paths-tools/mcp?api-version=v1|<query that exercises the github tool>
```

The workflow derives `TOOLBOX_NAME` from the URL slug and drives the toolbox with that query.

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

We **recommend deploying with `azd deploy`**, which uses ACR remote build and always produces images with the correct architecture.

If you choose to **build locally**, and your machine is **not `linux/amd64`** (for example, an Apple Silicon Mac), the image will **not be compatible with our service**, causing runtime failures.

**Fix for local builds:**

```bash
docker build --platform=linux/amd64 -t image .
```

This forces the image to be built for the required `amd64` architecture.

### Toolbox enumeration fails

Toolbox enumeration is all-or-nothing. If any configured source is misconfigured, has a bad PAT, lacks RBAC, or points to an unreachable MCP server, startup can fail. Add and validate one auth path at a time.

### GitHub tool returns `401` or `403`

A GitHub answer means the key-based `CustomKeys` path resolved correctly. A `401` or `403` means the connection credential did not resolve or the PAT lacks access to the requested repository.
