# Calculator Agent — LangGraph

This LangGraph agent can perform arithmetic calculations such as addition, subtraction, multiplication, and division.

## Running the Agent Locally

### Using `azd` (Recommended)

The [Azure Developer CLI `ai agent` extension](https://aka.ms/azdaiagent/docs) is the quickest way to run, invoke, and deploy this hosted agent.

**Run locally**

Set the required environment variables (defined in `agent.yaml`) in your `azd` environment so they are injected when running and deploying:

```bash
azd env set AZURE_OPENAI_ENDPOINT "https://<your-resource>.openai.azure.com/"
```

Start the agent locally:

```bash
azd ai agent run
```

The agent starts on `http://localhost:8088/`.

**Invoke**

**Bash:**
```bash
azd ai agent invoke --local '{"input": "What is the size of France in square miles, divided by 27?"}'
```

**PowerShell:**
```powershell
azd ai agent invoke --local '{\"input\": \"What is the size of France in square miles, divided by 27?\"}'
```

**Deploy to Microsoft Foundry**

```bash
# Provision Azure resources (skip if already provisioned)
azd provision

# Build, push, and deploy the agent to Foundry
azd deploy
```

After deploying, invoke the agent running in Foundry:

```bash
azd ai agent invoke '{"input": "What is the size of France in square miles, divided by 27?"}'
```

Stream logs from the running agent:

```bash
azd ai agent monitor
```

For the full deployment guide, see [Azure AI Foundry hosted agents](https://aka.ms/azdaiagent/docs).

<details>
<summary><h3>Using the Foundry Toolkit VS Code Extension</h3></summary>

The [Foundry Toolkit VS Code extension](https://learn.microsoft.com/en-us/azure/foundry/agents/quickstarts/quickstart-hosted-agent?view=foundry&pivots=vscode) has a built-in sample gallery that scaffolds this project into a new workspace — no manual cloning needed.

**Run in debug mode**

1. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Create new Hosted Agent`. The extension creates the VS Code debug configuration files and `.env`.
2. Set the required environment variables (defined in `agent.yaml`).
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

**Invoke with the Agent Inspector**

1. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Open Agent Inspector`.
2. The Inspector auto-connects to your running agent at `http://localhost:8088/`.
3. Type a message and send it.

**Deploy with the Deploy wizard**

1. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Deploy Hosted Agent`. The extension opens a tab-based **Deploy Hosted Agent** wizard and reads `agent.yaml` to auto-populate what it can.
2. If prompted, complete **Foundry Project Setup** to pick the subscription and Foundry project (or create a new one).
3. On the **Basics** tab, set the deployment method (**Code** or **Container**), the packaging/registry options, and the **Hosted Agent Name**.
4. On the **Review + Deploy** tab, confirm the runtime details, pick a **CPU and Memory** size, and click **Deploy**.
5. After deployment, invoke the agent in the Agent Playground and stream live logs from the **Logs** tab.

</details>

## Troubleshooting

### Images built on Apple Silicon or other ARM64 machines do not work on our service

We **recommend using `azd` cloud build**, which always builds images with the correct architecture.

If you choose to **build locally**, and your machine is **not `linux/amd64`** (for example, an Apple Silicon Mac), the image will **not be compatible with our service**, causing runtime failures.

**Fix for local builds**

Use this command to build the image locally:

```shell
docker build --platform=linux/amd64 -t image .
```

This forces the image to be built for the required `amd64` architecture.
