# Foundry Skills Agent (Responses Protocol)

An [Agent Framework](https://github.com/microsoft/agent-framework) agent that downloads behavioral guidance from Foundry Skills at startup and serves it through the **Responses protocol**. The sample shows how skill instructions can be authored as `SKILL.md`, uploaded to Foundry, and loaded on demand by the model without changing agent code.

## How It Works

1. `main.py` reads `FOUNDRY_PROJECT_ENDPOINT`, `AZURE_AI_MODEL_DEPLOYMENT_NAME`, and the comma-separated `SKILL_NAMES` list
2. `_bootstrap_skills()` uses `AIProjectClient(..., allow_preview=True)` and `DefaultAzureCredential` to download each named Foundry Skill as a ZIP archive
3. Each downloaded archive is safely extracted into `downloaded_skills/<name>/SKILL.md`, replacing the runtime folder on every startup
4. `SkillsProvider.from_paths()` advertises downloaded skill names and descriptions, then exposes `load_skill` so the model can fetch full skill content only when needed
5. The agent uses `FoundryChatClient` with Contoso Outdoors support instructions and the skills context provider
6. `ResponsesHostServer` exposes the agent through an OpenAI-compatible `POST /responses` endpoint and starts on `http://localhost:8088/`

See [main.py](main.py), [provision_skills.py](provision_skills.py), and the source [skills](skills/) for the full implementation.

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `FOUNDRY_PROJECT_ENDPOINT` | Yes | Azure AI Foundry project endpoint URL. Auto-injected when hosted — only needed locally |
| `AZURE_AI_MODEL_DEPLOYMENT_NAME` | Yes | Model deployment name (e.g. `gpt-4.1-mini`). Declared in `agent.yaml` |
| `SKILL_NAMES` | Yes, for skill loading | Comma-separated Foundry Skill names to download at startup, such as `support-style,escalation-policy` |

When deployed as a hosted agent, `FOUNDRY_PROJECT_ENDPOINT` is auto-injected by the platform. `SKILL_NAMES` is declared in `agent.yaml` and should be set in your `azd` environment before deployment. Authentication uses Managed Identity via `DefaultAzureCredential`.

## Running Locally

### Prerequisites

- Python 3.10+
- An Azure AI Foundry project with a model deployment (or let `azd provision` create one)
- **Azure AI User** on the Foundry project for the identity provisioning and downloading skills
- The skills uploaded to the same Foundry project with [provision_skills.py](provision_skills.py)

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
azd ai agent invoke --local "I want to confirm I can return my tent within 30 days."
```

**PowerShell:**
```powershell
azd ai agent invoke --local "I want to confirm I can return my tent within 30 days."
```

**Test with curl:**

```bash
curl -sS -X POST http://localhost:8088/responses \
  -H "Content-Type: application/json" \
  -d '{"input": "I want to confirm I can return my tent within 30 days.", "stream": false}' | jq .
```

<details>
<summary><h3>Using Foundry Toolkit VS Code Extension</h3></summary>

Open the **Agent Inspector** directly from the Foundry Toolkit extension to invoke the agent — no `curl` or CLI commands needed.

1. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Open Agent Inspector`.
2. The Inspector auto-connects to your running agent at `http://localhost:8088/`.
3. Type a message and send it. The Agent Inspector handles the protocol and displays the response inline.

> Multi-turn conversation is supported — the Inspector maintains session context across messages.

</details>

## Foundry Skills

This sample ships two source skills under [skills/](skills/):

| Skill | Purpose |
|---|---|
| [`support-style`](skills/support-style/SKILL.md) | Voice, formatting, and signature rules for Contoso Outdoors support replies. Includes canary token `STYLE-CANARY-3318` |
| [`escalation-policy`](skills/escalation-policy/SKILL.md) | When and how to escalate customer tickets. Includes canary token `ESC-CANARY-7742` |

The `name` and `description` values in `SKILL.md` YAML front matter must remain unquoted because the Foundry Skills API expects plain scalar values.

## Provisioning Skills

Run [provision_skills.py](provision_skills.py) once for the Foundry project you will use. The script packages each `skills/*/SKILL.md` file as a ZIP, deletes an existing skill with the same name, imports the new package through `AIProjectClient.beta.skills.create_from_package`, and verifies the skills can be listed.

```bash
export FOUNDRY_PROJECT_ENDPOINT="https://<account>.services.ai.azure.com/api/projects/<project>"
python provision_skills.py
```

PowerShell:

```powershell
$env:FOUNDRY_PROJECT_ENDPOINT="https://<account>.services.ai.azure.com/api/projects/<project>"
python provision_skills.py
```

On agent startup, the downloaded runtime copies land under `downloaded_skills/<name>/SKILL.md`. That folder is recreated on every run and should not be edited by hand.

## Preparing Hosted Deployment Values

Set the skill list in your `azd` environment before deploying:

```bash
azd env set SKILL_NAMES "support-style,escalation-policy"
```

The hosted agent's managed identity needs **Azure AI User** on the Foundry project to download skills at startup.

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
azd ai agent invoke "I want to confirm I can return my tent within 30 days."
```

**PowerShell:**
```powershell
azd ai agent invoke "I want to confirm I can return my tent within 30 days."
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

### Skill download returns `404`

Run `python provision_skills.py` against the same Foundry project endpoint used by the agent, and confirm `SKILL_NAMES` matches the skill folder names exactly.

### No skill behavior appears in responses

Confirm `SKILL_NAMES` is not empty and watch startup logs for the download messages. The model loads full skill bodies on demand through the `load_skill` tool, so ask a prompt that clearly matches a skill's description.

### Skill API calls fail with authorization errors

Assign **Azure AI User** on the Foundry project to the identity running locally or to the hosted agent's managed identity after deployment.
