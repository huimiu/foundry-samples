# Downstream Azure Agent (Storage + Service Bus, Responses Protocol)

An [Agent Framework](https://github.com/microsoft/agent-framework) hosted agent that performs Azure Blob Storage and Azure Service Bus data-plane operations using Microsoft Entra authentication and the **Responses protocol**. It demonstrates how the same `DefaultAzureCredential` code path works locally with your developer identity and in Foundry with the per-agent managed identity.

## How It Works

1. `main.py` loads local environment variables, creates a `FoundryChatClient`, and authenticates to Foundry with `DefaultAzureCredential`
2. The agent registers four `@tool` functions from `tools/`: `storage_put_blob`, `storage_get_blob`, `servicebus_send_message`, and `servicebus_peek_messages`
3. Storage tools read `AZURE_STORAGE_ACCOUNT_NAME` and `AZURE_STORAGE_CONTAINER_NAME`, then use `BlobServiceClient` to upload or download text blobs
4. Service Bus tools read `AZURE_SERVICEBUS_FQDN` and `AZURE_SERVICEBUS_QUEUE_NAME`, then use `ServiceBusClient` to send or peek queue messages
5. The agent is instructed to pick the matching tool for the named Azure service and briefly confirm completed actions
6. `ResponsesHostServer` exposes the agent through an OpenAI-compatible `POST /responses` endpoint and starts on `http://localhost:8088/`

See [main.py](main.py), [tools/storage.py](tools/storage.py), and [tools/servicebus.py](tools/servicebus.py) for the full implementation.

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `FOUNDRY_PROJECT_ENDPOINT` | Yes | Azure AI Foundry project endpoint URL. Auto-injected when hosted — only needed locally |
| `AZURE_AI_MODEL_DEPLOYMENT_NAME` | Yes | Model deployment name (e.g. `gpt-4.1-mini`). Declared in `agent.yaml` |
| `AZURE_STORAGE_ACCOUNT_NAME` | Yes | Storage account name used by the Blob Storage tools |
| `AZURE_STORAGE_CONTAINER_NAME` | Yes | Blob container name used by the Blob Storage tools |
| `AZURE_SERVICEBUS_FQDN` | Yes | Fully qualified Service Bus namespace, such as `<namespace>.servicebus.windows.net` |
| `AZURE_SERVICEBUS_QUEUE_NAME` | Yes | Service Bus queue name used by the Service Bus tools |
| `APPLICATIONINSIGHTS_CONNECTION_STRING` | No | Optional local telemetry connection string. Auto-injected when hosted if telemetry is configured |

When deployed as a hosted agent, `FOUNDRY_PROJECT_ENDPOINT` is auto-injected by the platform. The Azure resource names are declared in `agent.yaml` and should be set in your `azd` environment before deployment. Authentication uses Managed Identity via `DefaultAzureCredential`.

## Running Locally

### Prerequisites

- Python 3.10+
- An Azure AI Foundry project with a model deployment (or let `azd provision` create one)
- An Azure Storage account and blob container
- An Azure Service Bus namespace and queue
- RBAC assignments for your local user identity on the target Storage container and Service Bus queue

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
azd ai agent invoke --local "Upload a blob named hello.txt with the content hi from the agent."
```

**PowerShell:**
```powershell
azd ai agent invoke --local "Upload a blob named hello.txt with the content hi from the agent."
```

**Test with curl:**

```bash
curl -sS -X POST http://localhost:8088/responses \
  -H "Content-Type: application/json" \
  -d '{"input": "Upload a blob named hello.txt with the content hi from the agent.", "stream": false}' | jq .
```

<details>
<summary><h3>Using Foundry Toolkit VS Code Extension</h3></summary>

Open the **Agent Inspector** directly from the Foundry Toolkit extension to invoke the agent — no `curl` or CLI commands needed.

1. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Open Agent Inspector`.
2. The Inspector auto-connects to your running agent at `http://localhost:8088/`.
3. Type a message and send it. The Agent Inspector handles the protocol and displays the response inline.

> Multi-turn conversation is supported — the Inspector maintains session context across messages.

</details>

## Granting Azure Data-Plane Access

Both downstream services use Azure RBAC. For local runs, assign roles to your signed-in user. For hosted runs, assign roles to the per-agent service principal after `azd deploy`.

Capture the principal:

**Local (your developer identity):**

```bash
PRINCIPAL_ID=$(az ad signed-in-user show --query id -o tsv)
PRINCIPAL_TYPE="User"
```

**Foundry (per-agent identity, after `azd deploy`):**

```bash
PRINCIPAL_ID=$(azd ai agent show -o json | jq -r '.instance_identity.principal_id')
PRINCIPAL_TYPE="ServicePrincipal"
```

Assign Storage access on the container:

```bash
STORAGE_SCOPE=$(az storage account show \
  --name "$AZURE_STORAGE_ACCOUNT_NAME" \
  --query id -o tsv)/blobServices/default/containers/$AZURE_STORAGE_CONTAINER_NAME

az role assignment create \
  --assignee-object-id "$PRINCIPAL_ID" \
  --assignee-principal-type "$PRINCIPAL_TYPE" \
  --role "Storage Blob Data Contributor" \
  --scope "$STORAGE_SCOPE"
```

Assign Service Bus access on the queue:

```bash
QUEUE_SCOPE=$(az servicebus queue show \
  --namespace-name "<namespace>" \
  --resource-group "<resource-group>" \
  --name "$AZURE_SERVICEBUS_QUEUE_NAME" \
  --query id -o tsv)

az role assignment create \
  --assignee-object-id "$PRINCIPAL_ID" \
  --assignee-principal-type "$PRINCIPAL_TYPE" \
  --role "Azure Service Bus Data Sender" \
  --scope "$QUEUE_SCOPE"

az role assignment create \
  --assignee-object-id "$PRINCIPAL_ID" \
  --assignee-principal-type "$PRINCIPAL_TYPE" \
  --role "Azure Service Bus Data Receiver" \
  --scope "$QUEUE_SCOPE"
```

Role assignments can take a few minutes to propagate.

## Preparing Hosted Deployment Values

`agent.yaml` resolves the downstream Azure values from the `azd` environment at deploy time. Set them before deploying:

```bash
azd env set AZURE_STORAGE_ACCOUNT_NAME "<storage-account-name>"
azd env set AZURE_STORAGE_CONTAINER_NAME "<container-name>"
azd env set AZURE_SERVICEBUS_FQDN "<namespace>.servicebus.windows.net"
azd env set AZURE_SERVICEBUS_QUEUE_NAME "<queue-name>"
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
azd ai agent invoke "Upload a blob named hello.txt with the content hi from the agent."
```

**PowerShell:**
```powershell
azd ai agent invoke "Upload a blob named hello.txt with the content hi from the agent."
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

### Storage returns `AuthorizationPermissionMismatch`

The role assignment may not have propagated, or it may be scoped incorrectly. Confirm the principal has **Storage Blob Data Contributor** on the target container scope.

### Service Bus returns `Unauthorized`

The agent needs **Azure Service Bus Data Sender** to send messages and **Azure Service Bus Data Receiver** to peek messages. Assign both roles on the queue scope when using both tools.

### Local runs fail with credential errors

`DefaultAzureCredential` uses your developer identity locally. Run `az login`, then assign your user the same data-plane roles on the same scopes you expect the hosted agent to use.
