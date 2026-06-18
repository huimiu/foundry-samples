# A2A Executor Agent (Responses Protocol)

A Foundry-hosted [Agent Framework](https://github.com/microsoft/agent-framework) math specialist using the **Responses protocol**. After deployment, a setup script enables incoming A2A so the companion caller agent can discover and invoke this executor through Foundry's A2A endpoint.

## How It Works

1. `main.py` loads local `.env` values, then creates a `FoundryChatClient` pointed at your Foundry project endpoint and model deployment, authenticated with `DefaultAzureCredential`
2. The client is wrapped in an Agent Framework `Agent` with math-expert instructions and `store=False`
3. `ResponsesHostServer` exposes the agent through `POST /responses` on `http://localhost:8088/`
4. After deployment, `scripts/setup-a2a.ps1` or `scripts/setup-a2a.sh` patches the hosted agent to publish an `agent_card` and add `a2a` to `agent_endpoint.protocols`
5. The companion caller uses that A2A endpoint through a `RemoteA2A` connection and `a2a_preview` toolbox

See [main.py](main.py) for the full implementation.

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `FOUNDRY_PROJECT_ENDPOINT` | Yes | Azure AI Foundry project endpoint URL. Auto-injected when hosted — only needed locally and by the setup scripts |
| `AZURE_AI_MODEL_DEPLOYMENT_NAME` | Yes | Model deployment name (e.g. `gpt-4o`). Declared in `agent.yaml` |

When deployed as a hosted agent, `FOUNDRY_PROJECT_ENDPOINT` is auto-injected by the platform — you only need to set `AZURE_AI_MODEL_DEPLOYMENT_NAME`. Authentication uses Managed Identity via `DefaultAzureCredential`.

## Running Locally

### Prerequisites

- Python 3.10+
- An Azure AI Foundry project with a model deployment (or let `azd provision` create one)
- Azure CLI authenticated with a user that has the Foundry User role or higher on the project to enable incoming A2A

### Using `azd` (Recommended)

Create a local `.env` file from the sample template and fill in the required values:

```bash
cp .env.example .env  # skip if .env already exists
# Edit .env — see Environment Variables above
```

The sample loads `.env` automatically when running locally. Next, start the agent locally with the `run` command:

```bash
azd ai agent run
```

The agent starts on `http://localhost:8088/`.

<details>
<summary><h3>Using the Foundry Toolkit VS Code Extension</h3></summary>

The [Foundry Toolkit VS Code extension](https://learn.microsoft.com/en-us/azure/foundry/agents/quickstarts/quickstart-hosted-agent?view=foundry&pivots=vscode) has a built-in sample gallery that scaffolds this project directly into a new workspace — no manual cloning needed.

1. It's recommended to scaffold the project using the Foundry Toolkit extension. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Create new Hosted Agent`. The extension automatically creates the VS Code debug configuration files and `.env`.
2. Edit `.env` and fill in the required environment variables (see [Environment Variables](#environment-variables) above for the full list).
3. Set up a Python virtual environment:

   **Windows (PowerShell):**
   ```powershell
   python -m venv .venv
   .\.venv\Scripts\Activate.ps1
   ```

   **macOS/Linux:**
   ```bash
   python -m venv .venv
   source .venv/bin/activate
   ```

4. Install dependencies:
   ```bash
   pip install -r requirements.txt
   pip install debugpy
   ```

5. Press **F5** to start the agent in debug mode. The agent starts on `http://localhost:8088/`.

</details>
<details>
<summary><h3>Manual setup</h3></summary>

```bash
pip install -r requirements.txt
cp .env.example .env  # skip if .env already exists
# Edit .env — see Environment Variables above
python main.py
```

The agent starts on `http://localhost:8088/`.

</details>

## Invoke

### Using azd

**Local:**

**Bash:**
```bash
azd ai agent invoke --local "What is 15 multiplied by 23?"
```

**PowerShell:**
```powershell
azd ai agent invoke --local "What is 15 multiplied by 23?"
```

**Test with curl:**

```bash
curl -sS -X POST http://localhost:8088/responses \
  -H "Content-Type: application/json" \
  -d '{"input": "What is 15 multiplied by 23?", "stream": false}' | jq .
```

<details>
<summary><h3>Using Foundry Toolkit VS Code Extension</h3></summary>

Open the **Agent Inspector** directly from the Foundry Toolkit extension to invoke the agent — no `curl` or CLI commands needed.

1. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Open Agent Inspector`.
2. The Inspector auto-connects to your running agent at `http://localhost:8088/`.
3. Type a message and send it. The Agent Inspector handles the protocol and displays the response inline.

> Multi-turn conversation is supported — the Inspector maintains session context across messages.

</details>

## Incoming A2A Endpoint

A Responses-protocol hosted agent is reachable through Responses by default. To let other agents call this executor over A2A, run the setup script after deployment. The script patches the deployed agent to publish a math-expert `agent_card` and to support both `responses` and `a2a` protocols.

The script reads `FOUNDRY_PROJECT_ENDPOINT` from `.env` and defaults the agent name to `agent-framework-a2a-executor-responses`. It prints the A2A endpoint and agent card URL. If you adapt the executor for another specialty, update the hard-coded `agent_card` in both setup scripts because the caller's routing depends on the advertised skill descriptions.

## Running and Deploying the Delegation Pair

1. Deploy this executor first.
2. Enable incoming A2A:

   **Windows (PowerShell):**
   ```powershell
   .\scripts\setup-a2a.ps1
   ```

   **macOS/Linux:**
   ```bash
   ./scripts/setup-a2a.sh
   ```

3. Copy the printed A2A endpoint URL.
4. In the companion [caller](../caller/README.md), provide that URL as `a2a_executor_endpoint`; caller `azd provision` creates the `RemoteA2A` connection and `a2a-delegation-tools` toolbox.
5. Run or deploy the caller, then invoke it with a math prompt such as `What is 15 multiplied by 23?`.

For the full two-agent walkthrough, see the [parent README](../README.md).

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
azd ai agent invoke "What is 15 multiplied by 23?"
```

**PowerShell:**
```powershell
azd ai agent invoke "What is 15 multiplied by 23?"
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

### `setup-a2a` cannot find the project endpoint

Create `.env` from `.env.example` and set `FOUNDRY_PROJECT_ENDPOINT`, or pass `-ProjectEndpoint` to `scripts\setup-a2a.ps1`.

### Caller cannot discover the executor

Confirm `scripts/setup-a2a` completed successfully and that the caller uses the printed `/endpoint/protocols/a2a/` URL as `a2a_executor_endpoint`.
