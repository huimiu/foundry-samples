# Event Grid Blob Trigger Agent (Invocations Protocol)

A Bring Your Own hosted agent that receives Azure Storage BlobCreated events through the Microsoft Foundry **Invocations protocol**. Event Grid posts directly to the agent with managed identity authentication; the agent downloads the new blob, summarizes it with a Foundry model, and writes a summary JSON file to a separate container.

## How It Works

1. Event Grid delivers events to `POST /invocations` using a system topic system-assigned managed identity with audience `https://ai.azure.com`
2. [main.py](main.py) parses the JSON request body and accepts three shapes: Event Grid `SubscriptionValidationEvent`, Event Grid `Microsoft.Storage.BlobCreated` batches, or direct `{"container": "...", "name": "..."}` test payloads
3. For BlobCreated and direct payloads, the agent extracts the container/blob name, downloads `.txt` or `.md` content from Azure Storage with the per-agent managed identity, and truncates input to 64 KiB
4. The agent calls the Foundry Responses API with the configured model deployment to summarize the file in 3–5 bullet points
5. It writes `<name>.summary.json` to `AZURE_STORAGE_SUMMARY_CONTAINER_NAME`, logs the output path, and returns a JSON response
6. Writing to a sibling summary container avoids re-triggering the Event Grid subscription that watches the input container

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `FOUNDRY_PROJECT_ENDPOINT` | Yes | Azure AI Foundry project endpoint URL. Auto-injected when hosted — only needed locally |
| `AZURE_AI_MODEL_DEPLOYMENT_NAME` | Yes | Model deployment name (e.g. `gpt-4.1-mini`). Declared in `agent.yaml` |
| `AZURE_STORAGE_ACCOUNT_NAME` | Yes | Storage account containing the input and summary containers |
| `AZURE_STORAGE_SUMMARY_CONTAINER_NAME` | Yes | Container where the agent writes `<name>.summary.json`. Must be different from the watched input container |
| `APPLICATIONINSIGHTS_CONNECTION_STRING` | No | Application Insights connection string. Auto-injected when hosted — only needed locally for telemetry |

## Running Locally

### Prerequisites

- Python 3.10+
- An Azure AI Foundry project with a model deployment (or let `azd provision` create one)
- An Azure Storage account with an input container and a separate summary container
- Azure credentials with permission to assign RBAC roles and create Event Grid system topics/subscriptions

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
azd ai agent invoke --local '{"container": "input", "name": "hello.txt"}'
```

**PowerShell:**
```powershell
azd ai agent invoke --local '{\"container\": \"input\", \"name\": \"hello.txt\"}'
```

**Test with curl:**

```bash
curl -sS -X POST http://localhost:8088/invocations \
  -H "Content-Type: application/json" \
  -d '{"container": "input", "name": "hello.txt"}' | jq .
```

The direct invoke payload assumes `input/hello.txt` exists in `AZURE_STORAGE_ACCOUNT_NAME` and that the caller identity has blob read access.

<details>
<summary><h3>Using Foundry Toolkit VS Code Extension</h3></summary>

Open the **Agent Inspector** directly from the Foundry Toolkit extension to invoke the agent — no `curl` or CLI commands needed.

1. Open the Command Palette (`Ctrl+Shift+P`) and run `Foundry Toolkit: Open Agent Inspector`.
2. The Inspector auto-connects to your running agent at `http://localhost:8088/`.
3. Type a message and send it. The Agent Inspector handles the protocol and displays the response inline.

> Multi-turn conversation is supported — the Inspector maintains session context across messages.

</details>

## Event Grid Setup and Verification

After you deploy the agent using the section below, complete these Event Grid-specific steps.

1. Store the storage settings in your `azd` environment so they are injected into the hosted agent:

   ```bash
   azd env set AZURE_STORAGE_ACCOUNT_NAME "<storage-account-name>"
   azd env set AZURE_STORAGE_SUMMARY_CONTAINER_NAME "<summary-container-name>"
   ```

2. Grant the per-agent identity the required roles:

   - **Storage Blob Data Reader** on the input container
   - **Storage Blob Data Contributor** on the summary container
   - **Foundry User** on the Foundry project

   Get the identity with `azd ai agent show -o json` and use `instance_identity.principal_id` in role assignments.

3. Create an Event Grid system topic on the storage account with system-assigned managed identity enabled, then grant that topic identity **Foundry User** on the Foundry project. Event delivery must request tokens for the `https://ai.azure.com` audience.

4. Create the event subscription with `az rest` so you can set `deliveryWithResourceIdentity`:

   ```bash
   AGENT_URL=$(azd ai agent show -o json | jq -r '.agent_endpoints.invocations')
   SUB_ID=$(az account show --query id -o tsv)
   TENANT_ID=$(az account show --query tenantId -o tsv)

   cat > eg-sub.json <<EOF
   {
     "properties": {
       "deliveryWithResourceIdentity": {
         "identity": { "type": "SystemAssigned" },
         "destination": {
           "endpointType": "WebHook",
           "properties": {
             "endpointUrl": "$AGENT_URL",
             "azureActiveDirectoryTenantId": "$TENANT_ID",
             "azureActiveDirectoryApplicationIdOrUri": "https://ai.azure.com"
           }
         }
       },
       "filter": {
         "includedEventTypes": ["Microsoft.Storage.BlobCreated"],
         "subjectBeginsWith": "/blobServices/default/containers/$INPUT_CONTAINER/"
       },
       "eventDeliverySchema": "EventGridSchema"
     }
   }
   EOF

   az rest --method put \
     --url "https://management.azure.com/subscriptions/$SUB_ID/resourceGroups/$RG/providers/Microsoft.EventGrid/systemTopics/$TOPIC/eventSubscriptions/$SUB_NAME?api-version=2025-07-15-preview" \
     --headers "Content-Type=application/json" \
     --body @eg-sub.json
   ```

5. Upload a `.txt` or `.md` blob to the input container and verify `<name>.summary.json` appears in the summary container:

   ```bash
   echo "Hosted agents process Event Grid blob-created events end to end." > hello.txt
   az storage blob upload --account-name "$AZURE_STORAGE_ACCOUNT_NAME" -c "$INPUT_CONTAINER" -f hello.txt -n hello.txt --auth-mode login
   az storage blob download --account-name "$AZURE_STORAGE_ACCOUNT_NAME" -c "$SUMMARY_CONTAINER" -n hello.txt.summary.json --auth-mode login -f - | cat
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
azd ai agent invoke '{"container": "input", "name": "hello.txt"}'
```

**PowerShell:**
```powershell
azd ai agent invoke '{\"container\": \"input\", \"name\": \"hello.txt\"}'
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

| Symptom | Likely cause |
|---|---|
| Event Grid subscription provisioning fails with `Webhook validation handshake failed` | The agent did not return `validationResponse`, or the system topic's managed identity is missing **Foundry User** on the Foundry project. |
| `401 Unauthorized` from the agent on real events | The system topic identity is missing **Foundry User**, the subscription audience is not `https://ai.azure.com`, or the tenant ID is wrong. |
| Agent trace shows `401 PermissionDenied` calling the model | The per-agent identity is missing **Foundry User** on the Foundry project. |
| Agent returns `AuthorizationPermissionMismatch` reading the blob | The per-agent identity is missing **Storage Blob Data Reader** on the input container. |
| Summary blob is never written | The per-agent identity is missing **Storage Blob Data Contributor** on the summary container, or the summary container does not exist. |
| Agent fires twice per upload | The summary container is the same container watched by Event Grid. Use a distinct summary container. |

## See also

- [Deliver events using managed identity](https://learn.microsoft.com/en-us/azure/event-grid/managed-service-identity) — Event Grid managed-identity delivery
- [`09-downstream-azure`](https://github.com/microsoft-foundry/foundry-samples/blob/main/samples/python/hosted-agents/agent-framework/responses/09-downstream-azure/README.md) — per-agent identity and Azure RBAC in a chat-driven agent
