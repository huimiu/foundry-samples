# LangGraph Files Agent (Responses Protocol)

A [LangGraph](https://langchain-ai.github.io/langgraph/) file-analysis agent hosted on Microsoft Foundry using the **Responses protocol**. It combines local filesystem tools with a Foundry Toolbox `code_interpreter` tool so the agent can find, read, and analyze files.

## How It Works

1. `main.py` loads local settings from `.env`, then builds a Foundry-backed `ChatOpenAI` model from `FOUNDRY_PROJECT_ENDPOINT` and `AZURE_AI_MODEL_DEPLOYMENT_NAME` using `DefaultAzureCredential`.
2. Three local LangChain tools are registered: `get_cwd`, `list_files`, and `read_file`.
3. `_load_toolbox_tools()` loads tools from `AzureAIProjectToolbox(toolbox_name=os.environ["TOOLBOX_NAME"])`; the manifest provisions a toolbox named `agent-tools` with `code_interpreter`.
4. `langchain.agents.create_agent(...)` compiles a LangGraph ReAct agent with the local file tools, toolbox tools, and a system prompt requiring mathematical calculations to use code interpreter.
5. The bundled `resources/contoso_q1_2026_report.txt` ships in the container image, and hosted-session uploads are mounted into the agent's working directory.
6. `ResponsesHostServer` exposes the OpenAI-compatible `POST /responses` endpoint on `http://localhost:8088/`.

See [main.py](main.py) for the full implementation.

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `FOUNDRY_PROJECT_ENDPOINT` | Yes | Azure AI Foundry project endpoint URL. Auto-injected when hosted — only needed locally |
| `AZURE_AI_MODEL_DEPLOYMENT_NAME` | Yes | Model deployment name (e.g. `gpt-4o`). Declared in `agent.yaml` |
| `TOOLBOX_NAME` | Yes | Foundry Toolbox name. Defaults to `agent-tools`, which `azd provision` creates from `agent.manifest.yaml` |

When deployed as a hosted agent, `FOUNDRY_PROJECT_ENDPOINT` is auto-injected by the platform — you only need to set `AZURE_AI_MODEL_DEPLOYMENT_NAME` and ensure `TOOLBOX_NAME` points at a toolbox with `code_interpreter`. Authentication uses Managed Identity via `DefaultAzureCredential`.

## Running Locally

### Prerequisites

- Python 3.12+
- An Azure AI Foundry project with a model deployment (or let `azd provision` create one)
- A Foundry Toolbox named `agent-tools` exposing the `code_interpreter` tool, or `azd provision` against [agent.manifest.yaml](agent.manifest.yaml) to create it automatically

### Using `azd`

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

The [Foundry Toolkit VS Code extension](https://learn.microsoft.com/en-us/azure/foundry/agents/quickstarts/quickstart-hosted-agent?view=foundry&pivots=vscode) sets up a one-click **F5** debug experience for this sample.

1. Scaffold the project using the Foundry Toolkit extension: open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Create new Hosted Agent`. The extension automatically creates the VS Code debug configuration files and `.env`.
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
# Create and activate a virtual environment
python -m venv .venv
source .venv/bin/activate  # Windows (PowerShell): .\.venv\Scripts\Activate.ps1

# Install dependencies
pip install -r requirements.txt

# Configure environment variables
cp .env.example .env  # skip if .env already exists
# Edit .env — see Environment Variables above

# Run the agent
python main.py
```

The agent starts on `http://localhost:8088/`.

</details>

## Invoke

### Using azd

**Local:**

**Bash:**
```bash
azd ai agent invoke --local "Find the quarterly report in the resources folder and tell me the revenue difference between Q1 2026 and Q1 2025."
```

**PowerShell:**
```powershell
azd ai agent invoke --local "Find the quarterly report in the resources folder and tell me the revenue difference between Q1 2026 and Q1 2025."
```

**Test with curl:**

```bash
curl -sS -X POST http://localhost:8088/responses \
  -H "Content-Type: application/json" \
  -d '{"input": "Find the quarterly report in the resources folder and tell me the revenue difference between Q1 2026 and Q1 2025.", "stream": false}' | jq .
```

<details>
<summary><h3>Using Foundry Toolkit VS Code Extension</h3></summary>

Open the **Agent Inspector** directly from the Foundry Toolkit extension to invoke the agent — no `curl` or CLI commands needed.

1. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Open Agent Inspector`.
2. The Inspector auto-connects to your running agent at `http://localhost:8088/`.
3. Type a message and send it. The Agent Inspector handles the protocol and displays the response inline.

> Multi-turn conversation is supported — the Inspector maintains session context across messages.

</details>

## Working with Files

| Tool | Source | Purpose |
|------|--------|---------|
| `get_cwd` | Local `@tool` | Returns the agent's current working directory |
| `list_files` | Local `@tool` | Lists entries under a directory |
| `read_file` | Local `@tool` | Reads a UTF-8 text file |
| `code_interpreter` | Foundry Toolbox | Runs Python in a managed sandbox for math and data work |

The bundled report at [resources/contoso_q1_2026_report.txt](resources/contoso_q1_2026_report.txt) lets the sample run without uploading anything. The agent should call `get_cwd` and `list_files` to locate it, `read_file` to load it, and `code_interpreter` to calculate the revenue delta.

After deployment, upload files to the hosted agent session and ask follow-up questions about them:

```bash
azd ai agent files upload -f resources/contoso_q1_2026_report.txt
azd ai agent invoke "Read the quarterly report I just uploaded and summarize the year-over-year revenue change."
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
azd ai agent invoke "Find the quarterly report in the resources folder and tell me the revenue difference between Q1 2026 and Q1 2025."
```

**PowerShell:**
```powershell
azd ai agent invoke "Find the quarterly report in the resources folder and tell me the revenue difference between Q1 2026 and Q1 2025."
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
