**IMPORTANT!** All samples and other resources made available in this GitHub repository ("samples") are designed to assist in accelerating development of agents, solutions, and agent workflows for various scenarios. Review all provided resources and carefully test output behavior in the context of your use case. AI responses may be inaccurate and AI actions should be monitored with human oversight.

# Env Vars Agent (Responses Protocol)

A Bring Your Own hosted agent built with [azure-ai-agentserver-responses](https://pypi.org/project/azure-ai-agentserver-responses/) that demonstrates Foundry connection-templated environment-variable injection. It uses the **Responses protocol** and a function-calling tool that safely reports non-secret values while returning only fingerprints for secrets.

## How It Works

1. [agent.yaml](agent.yaml) and [agent.manifest.yaml](agent.manifest.yaml) declare `AZURE_AI_MODEL_DEPLOYMENT_NAME` plus four example environment variables that reference Foundry connections.
2. At container startup, the Foundry hosting platform resolves placeholders such as `${{connections.dummy-api-key.credentials.key}}` and injects the resulting values into the process environment.
3. [main.py](main.py) creates a `ResponsesAgentServerHost`, which exposes `POST /responses`; the handler reads the request body `input` field and runs a Responses API function-calling loop.
4. The model can call `get_env_var(name, kind)`. The tool returns full values for `metadata` and `target` fields, but only a length and first-four-character fingerprint for `credentials` fields.
5. The handler detects unset, empty, and unresolved placeholder values, feeds tool outputs back to the model, and streams the final answer through Responses protocol lifecycle events.

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `FOUNDRY_PROJECT_ENDPOINT` | Yes, for local runs | Azure AI Foundry project endpoint URL. Auto-injected when hosted — only needed locally |
| `AZURE_AI_MODEL_DEPLOYMENT_NAME` | Yes | Model deployment name (for example, `gpt-4.1-mini`). Declared in `agent.yaml` and `agent.manifest.yaml` |
| `SECRET_API_KEY` | For the env-var injection demo | ApiKey connection credential from `${{connections.dummy-api-key.credentials.key}}`. Returned only as a fingerprint |
| `TARGET` | For the env-var injection demo | ApiKey connection target from `${{connections.dummy-api-key.target}}`. Returned as a non-secret full value |
| `SECRET_KEY` | For the env-var injection demo | CustomKeys secret from `${{connections.dummy-custom-keys.credentials.secret-key}}`. Returned only as a fingerprint |
| `NON_SECRET_KEY` | For the env-var injection demo | CustomKeys metadata value from `${{connections.dummy-custom-keys.metadata.plain-key}}`. Returned as a non-secret full value |
| `APPLICATIONINSIGHTS_CONNECTION_STRING` | No | Enables local telemetry. Auto-injected in hosted containers when monitoring is configured |

When deployed as a hosted agent, `FOUNDRY_PROJECT_ENDPOINT` is auto-injected by the platform. Authentication uses Managed Identity via `DefaultAzureCredential`; locally, use `az login` or another supported credential source.

## Running Locally

### Prerequisites

- Python 3.10+
- Azure CLI installed and authenticated (`az login`) or another `DefaultAzureCredential` source
- An Azure AI Foundry project with a model deployment (or let `azd provision` create one)
- For hosted deployment, an ApiKey connection named `dummy-api-key` and a CustomKeys connection named `dummy-custom-keys`, or updated placeholders that match your project
- For local testing, sample values in `.env` for `SECRET_API_KEY`, `TARGET`, `SECRET_KEY`, and `NON_SECRET_KEY`

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
azd ai agent invoke --local "did SECRET_API_KEY resolve? it is a credentials placeholder."
```

**PowerShell:**
```powershell
azd ai agent invoke --local "did SECRET_API_KEY resolve? it is a credentials placeholder."
```

**Test with curl:**

```bash
curl -sS -X POST http://localhost:8088/responses \
  -H "Content-Type: application/json" \
  -d '{"input": "did SECRET_API_KEY resolve? it is a credentials placeholder.", "stream": false}' | jq .
```

<details>
<summary><h3>Using Foundry Toolkit VS Code Extension</h3></summary>

Open the **Agent Inspector** directly from the Foundry Toolkit extension to invoke the agent — no `curl` or CLI commands needed.

1. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Open Agent Inspector`.
2. The Inspector auto-connects to your running agent at `http://localhost:8088/`.
3. Type a message and send it. The Agent Inspector handles the protocol and displays the response inline.

> Multi-turn conversation is supported — the Inspector maintains session context across messages.

</details>

## Connection Placeholder Matrix

| Env var | Connection kind | Placeholder | Tool `kind` | Returned to the model |
|---------|-----------------|-------------|-------------|-----------------------|
| `SECRET_API_KEY` | ApiKey | `${{connections.dummy-api-key.credentials.key}}` | `credentials` | Fingerprint only |
| `TARGET` | ApiKey | `${{connections.dummy-api-key.target}}` | `target` | Whole value |
| `SECRET_KEY` | CustomKeys | `${{connections.dummy-custom-keys.credentials.secret-key}}` | `credentials` | Fingerprint only |
| `NON_SECRET_KEY` | CustomKeys | `${{connections.dummy-custom-keys.metadata.plain-key}}` | `metadata` | Whole value |

Replace `dummy-api-key`, `dummy-custom-keys`, `secret-key`, and `plain-key` with names from your Foundry project before deploying. For local testing, put any representative values in `.env`; the tool behavior is the same, but no platform resolver is involved.

## How Connection Resolution Works

| Path | Template syntax | Source on the connection | Use for |
|------|-----------------|--------------------------|---------|
| `credentials.<field>` | `${{connections.<name>.credentials.key}}` or `${{connections.<name>.credentials.<key-name>}}` | The connection secret store | API keys and CustomKeys entries marked as secret |
| `target` | `${{connections.<name>.target}}` | The connection target endpoint | Base URLs and endpoints |
| `metadata.<key>` | `${{connections.<name>.metadata.<key-name>}}` | The connection metadata bag | Plain CustomKeys values such as region, account name, or feature flags |

CustomKeys connections can mix secret and non-secret values. Secret keys land in `credentials`; non-secret keys land in `metadata`. The agent mirrors that split with the `kind` argument and never returns raw credential values.

## Local Test Prompts

```bash
azd ai agent invoke --local "what is TARGET? it is the target of an ApiKey connection."
azd ai agent invoke --local "did SECRET_API_KEY resolve? it is a credentials placeholder."
azd ai agent invoke --local "what is NON_SECRET_KEY? it is metadata from a CustomKeys connection."
azd ai agent invoke --local "did SECRET_KEY resolve? it is a credentials placeholder."
```

To observe streaming Responses protocol events directly, use `stream: true`:

```bash
curl -N -X POST http://localhost:8088/responses \
  -H "Content-Type: application/json" \
  -d '{"input": "what is TARGET? it is the target of an ApiKey connection.", "stream": true}'
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
azd ai agent invoke "did SECRET_API_KEY resolve? it is a credentials placeholder."
```

**PowerShell:**
```powershell
azd ai agent invoke "did SECRET_API_KEY resolve? it is a credentials placeholder."
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

### An env var shows status `UNRESOLVED_PLACEHOLDER`

The platform resolver did not replace the placeholder before the container started. Check that the connection name and field name match exactly, the field is stored in the expected connection area (`credentials`, `target`, or `metadata`), and the agent was redeployed after `agent.yaml` or `agent.manifest.yaml` changed.

### Azure OpenAI permission denied (401)

The identity running the agent needs RBAC permissions on the Foundry project and model deployment. Assign roles such as **Cognitive Services OpenAI User** and **Azure AI User**, then allow a few minutes for propagation before retrying.