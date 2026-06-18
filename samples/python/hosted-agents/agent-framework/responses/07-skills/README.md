# File-Based Skills Agent (Responses Protocol)

An [Agent Framework](https://github.com/microsoft/agent-framework) agent hosted on Microsoft Foundry using the **Responses protocol**. It demonstrates native file-based skills by loading a local `skills/` folder and running a trusted skill script to generate a PDF travel guide.

## How It Works

1. `main.py` creates a `FoundryChatClient` pointed at your Foundry project endpoint and model deployment, authenticated with `DefaultAzureCredential`
2. `run_local_skill_script()` validates that a requested script belongs to the file-based skill directory, runs it with the current Python interpreter, captures output, and enforces a 60-second timeout
3. `SkillsProvider.from_paths(...)` loads skills from the local [skills](skills/) directory and uses the script runner for skill scripts
4. The included [travel-guide](skills/travel-guide/) skill advertises `scripts/create_travel_guide.py`, which creates a colorful PDF using only the Python standard library
5. The Agent Framework `Agent` receives the skills provider as a context provider and is served by `ResponsesHostServer`, which exposes an OpenAI-compatible `POST /responses` endpoint on `http://localhost:8088/`

See [main.py](main.py), [skills/travel-guide/SKILL.md](skills/travel-guide/SKILL.md), and [skills/travel-guide/scripts/create_travel_guide.py](skills/travel-guide/scripts/create_travel_guide.py) for the full implementation.

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `FOUNDRY_PROJECT_ENDPOINT` | Yes | Azure AI Foundry project endpoint URL. Auto-injected when hosted — only needed locally |
| `AZURE_AI_MODEL_DEPLOYMENT_NAME` | Yes | Model deployment name (e.g. `gpt-4o`). Declared in `agent.yaml` |

When deployed as a hosted agent, `FOUNDRY_PROJECT_ENDPOINT` is auto-injected by the platform — you only need to set `AZURE_AI_MODEL_DEPLOYMENT_NAME`. Authentication uses Managed Identity via `DefaultAzureCredential`.

## Running Locally

### Prerequisites

- Python 3.10+
- An Azure AI Foundry project with a model deployment (or let `azd provision` create one)
- Write access to `$HOME/generated-travel-guides`, where the included skill writes generated PDFs by default

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
azd ai agent invoke --local "Create a colorful 3-day PDF travel guide for Lisbon focused on food, viewpoints, and neighborhoods."
```

**PowerShell:**
```powershell
azd ai agent invoke --local "Create a colorful 3-day PDF travel guide for Lisbon focused on food, viewpoints, and neighborhoods."
```

**Test with curl:**

```bash
curl -sS -X POST http://localhost:8088/responses \
  -H "Content-Type: application/json" \
  -d '{"input": "Create a colorful 3-day PDF travel guide for Lisbon focused on food, viewpoints, and neighborhoods.", "stream": false}' | jq .
```

<details>
<summary><h3>Using Foundry Toolkit VS Code Extension</h3></summary>

Open the **Agent Inspector** directly from the Foundry Toolkit extension to invoke the agent — no `curl` or CLI commands needed.

1. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Open Agent Inspector`.
2. The Inspector auto-connects to your running agent at `http://localhost:8088/`.
3. Type a message and send it. The Agent Inspector handles the Responses protocol and displays the reply inline.

> Multi-turn conversation is supported — the Inspector maintains session context across messages.

</details>

## Adding Skills

Any subdirectory under `skills/` containing a `SKILL.md` file can be loaded by `SkillsProvider`. The included `travel-guide` skill demonstrates this layout:

```text
skills/
└── travel-guide/
    ├── SKILL.md
    └── scripts/
        └── create_travel_guide.py
```

The `SKILL.md` file tells the model when to use the skill and which script arguments are supported. For the included travel guide generator, supported script flags are:

- `--city <city>`: destination city, required
- `--days <number>`: number of itinerary days, optional, defaults to `3`
- `--interests <list>`: comma-separated interests such as `food,art,history,views`, optional
- `--tone <style>`: guide style such as `family-friendly`, `luxury`, `budget`, or `first-time visitor`, optional

Example script arguments advertised by the skill:

```json
["--city", "Lisbon", "--days", "3", "--interests", "food,viewpoints,neighborhoods", "--tone", "first-time visitor"]
```

The skill writes the PDF to `$HOME/generated-travel-guides` and returns a file path. For production scenarios that need durable external sharing, update the script to upload the PDF to storage and return a shareable URL.

## Next Steps

- [Quickstart: Create a hosted agent](https://learn.microsoft.com/en-us/azure/foundry/agents/quickstarts/quickstart-hosted-agent) — end-to-end walkthrough using `azd`
- [Manage hosted agents](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/manage-hosted-agent) — monitor and manage deployed agents

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
azd ai agent invoke "Create a colorful 3-day PDF travel guide for Lisbon focused on food, viewpoints, and neighborhoods."
```

**PowerShell:**
```powershell
azd ai agent invoke "Create a colorful 3-day PDF travel guide for Lisbon focused on food, viewpoints, and neighborhoods."
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
