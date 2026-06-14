**IMPORTANT!** All samples and other resources made available in this GitHub repository ("samples") are designed to assist in accelerating development of agents, solutions, and agent workflows for various scenarios. Review all provided resources and carefully test output behavior in the context of your use case. AI responses may be inaccurate and AI actions should be monitored with human oversight. Learn more in the transparency documents for [Agent Service](https://learn.microsoft.com/en-us/azure/ai-foundry/responsible-ai/agents/transparency-note) and [Agent Framework](https://github.com/microsoft/agent-framework/blob/main/TRANSPARENCY_FAQ.md).

Agents, solutions, or other output you create may be subject to legal and regulatory requirements, may require licenses, or may not be suitable for all industries, scenarios, or use cases. By using any sample, you are acknowledging that any output created using those samples are solely your responsibility, and that you will comply with all applicable laws, regulations, and relevant safety standards, terms of service, and codes of conduct.

Third-party samples contained in this folder are subject to their own designated terms, and they have not been tested or verified by Microsoft or its affiliates.

Microsoft has no responsibility to you or others with respect to any of these samples or any resulting output.

# What this sample demonstrates

This sample demonstrates how to build a Microsoft Agent Framework workflow that persists checkpoints and pauses for human-in-the-loop (HITL) review before completing a response. The workflow is hosted with the [Azure AI AgentServer SDK](https://pypi.org/project/azure-ai-agentserver-agentframework/) and can be deployed to Microsoft Foundry using the Azure Developer CLI [ai agent](https://aka.ms/azdaiagent/docs) extension.

## How It Works

### Checkpoints

Agent-framework workflow can resume by loading a checkpoint. Hosted agent provides a CheckpointRepository API for users to manage their checkpoints. It defines as below:

```py
class CheckpointRepository(ABC):
    """
    Repository interface for storing and retrieving checkpoints.
    
    :meta private:
    """
    @abstractmethod
    async def get_or_create(self, conversation_id: str) -> Optional[CheckpointStorage]:
        """Retrieve or create a checkpoint storage by conversation ID.

        :param conversation_id: The unique identifier for the checkpoint.
        :type conversation_id: str
        :return: The CheckpointStorage if found or created, None otherwise.
        :rtype: Optional[CheckpointStorage]
        """
```

An in-memory checkpoint repository `azure.ai.agentserver.agentframework.persistence.InMemoryCheckpointRepository` and a local file based `azure.ai.agentserver.agentframework.persistence.FileCheckpointRepository(storage_path: str)` are provided.

If checkpoint repository is provided, hosted agent adapter will search for previous checkpoints by `conversation_id`, load the latest checkpoint to `WorkflowAgent`, and then invoke the workflow agent with `CheckpointStorage` instance. Thus, the checkpoint will be updated by agent framework.

In this sample, the workflow persists checkpoints through `FileCheckpointRepository(storage_path="./checkpoints")`, ensuring the pending review queue survives restarts.


### Workflow with HITL

`workflow_as_agent_reflection_pattern.py` defines two executors:

- `Worker` – Generates answers with `AzureOpenAIChatClient`, tracks pending review requests, emits final responses, and implements `on_checkpoint_save` / `on_checkpoint_restore` so pending work can be resumed in multiturn conversions.
- `ReviewerWithHumanInTheLoop` – Always escalates to a human. The `HumanReviewRequest` payload captures the entire conversation so the reviewer can approve or reject the draft. When the reviewer responds, the workflow either emits the answer or regenerates it with the supplied feedback. Hosted agent adapter converts the HITL request to a `function_call` item with `HumanReviewRequest` information as argument. `HumanReviewRequest.convert_to_payload` is used for conversion.
- Human feedback should be provided as a `function_call_output` item with `conversation_id` and `call_id` matching with feedback request. Hosted agent adapter convert the feedback to targeted data instance by calling `ReviewResponse.convert_from_payload`.


### Agent hosting

`main.py` builds the workflow, adapts it with `from_agent_framework`, and starts a local OpenAI Responses-compatible endpoint on `http://localhost:8088`. The endpoint supports both streaming and non-streaming modes and emits `function_call` items whenever the workflow pauses for human feedback.

### Agent deployment

The same container image can be deployed to Microsoft Foundry with the Azure Developer CLI [ai agent](https://aka.ms/azdaiagent/docs) extension, which pushes the image to Azure Container Registry and creates hosted agent versions and deployments.

## Validate the deployed Agent
```python
# Before running the sample:
#    pip install --pre azure-ai-projects>=2.0.0b4

from azure.identity import DefaultAzureCredential
from azure.ai.projects import AIProjectClient
import json

foundry_account = "<Your Foundry Account Name>"
foundry_project = "<Your Foundry Project Name>"
agent_name = "<Your Deployed Agent Name>"

project_endpoint = f"https://{foundry_account}.services.ai.azure.com/api/projects/{foundry_project}"

project_client = AIProjectClient(
    endpoint=project_endpoint,
    credential=DefaultAzureCredential(),
)

# Get an existing agent
agent = project_client.agents.get(agent_name=agent_name)
print(f"Retrieved agent: {agent.name}")

openai_client = project_client.get_openai_client()
conversation = openai_client.conversations.create()

response = openai_client.responses.create(
    input="Draft a launch plan for a sustainable backpack brand",
    conversation=conversation.id,
    extra_body={"agent_reference": {"name": agent.name, "type": "agent_reference"}},
)

call_id = ""
request_id = ""
for item in response.output:
    if item.type == "function_call" and item.name == "__hosted_agent_adapter_hitl__":
        agent_request = json.loads(item.arguments).get("agent_request", {})
        request_id = agent_request.get("request_id", "")
        
        agent_messages = agent_request.get("agent_messages", [])
        agent_messages_str = "\n".join(json.dumps(msg, indent=4) for msg in agent_messages)
        print(f"Agent requests: {agent_messages_str}")
        call_id = item.call_id

if not call_id or not request_id:
    print(f"No human input is required, output: {response.output_text}")
else:
    human_response = {
        "request_id": request_id,
        "feedback": "approve",
        "approved": True,
    }
    response = openai_client.responses.create(
        input=[
            {
                "type": "function_call_output",
                "call_id": call_id,
                "output": json.dumps(human_response)
            }],
        conversation=conversation.id,
        extra_body={"agent_reference": {"name": agent.name, "type": "agent_reference"}},
    )
    print(f"Human response: {human_response['feedback']}")
    print(f"Agent response: {response.output_text}")
```

## Running the Agent Locally

### Prerequisites

1. **Azure OpenAI Service** – An endpoint with a deployed chat model (for example `gpt-4o-mini`). Record the endpoint URL and deployment name.
2. **Azure CLI** – Installed and signed in (`az login`). The workflow uses `AzureCliCredential` for Azure OpenAI authentication.
3. **Python 3.10 or higher** – Verify with `python --version`. Install a newer version if required.
4. **pip** – To install the sample dependencies.

### Environment variables

Set the following variables before running the sample (use a `.env` file or your shell environment):

- `AZURE_OPENAI_ENDPOINT` – Azure OpenAI endpoint URL (required).
- `AZURE_OPENAI_CHAT_DEPLOYMENT_NAME` – Deployment name for your chat model (required).

```powershell
# Replace the placeholder values
$env:AZURE_OPENAI_ENDPOINT="https://your-openai-resource.openai.azure.com/"
$env:AZURE_OPENAI_CHAT_DEPLOYMENT_NAME="gpt-4o-mini"
```

### Using `azd` (Recommended)

The [Azure Developer CLI `ai agent` extension](https://aka.ms/azdaiagent/docs) is the quickest way to run, invoke, and deploy this hosted agent.

**Run locally**

Set the required environment variables (see [Environment Variables](#environment-variables) above) in your `azd` environment so they are injected when running and deploying:

```bash
azd env set AZURE_OPENAI_ENDPOINT "https://your-openai-resource.openai.azure.com/"
```

Start the agent locally:

```bash
azd ai agent run
```

The agent starts on `http://localhost:8088/`.

**Invoke**

**Bash:**
```bash
azd ai agent invoke --local '{"input": "Draft a launch plan for a sustainable backpack brand", "stream": false}'
```

**PowerShell:**
```powershell
azd ai agent invoke --local '{\"input\": \"Draft a launch plan for a sustainable backpack brand\", \"stream\": false}'
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
azd ai agent invoke '{"input": "Draft a launch plan for a sustainable backpack brand", "stream": false}'
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
2. Set the required environment variables (see [Environment Variables](#environment-variables) above).
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

<details>
<summary><h3>Manual setup (without `azd`)</h3></summary>

### Installing dependencies

From the `workflow-agent-with-checkpoint-and-hitl` folder:

```powershell
pip install -r requirements.txt
```

### Running the sample

Start the hosted workflow locally:

```powershell
python main.py
```

The server listens on `http://localhost:8088/` and writes checkpoints to the `./checkpoints` directory.

### Interacting with the agent locally

Send a `POST` request to `http://0.0.0.0:8088/responses`

```json
{
    "agent": {"name": "local_agent", "type": "agent_reference"},
    "stream": false,
    "input": "Draft a launch plan for a sustainable backpack brand",
}
```

A response with human-review request looks like this (formatted for clarity):

```json
{
  "conversation": {"id": "<conversation_id>"},
  "output": [
    {...},
    {
      "type": "function_call",
      "id": "func_xxx",
      "name": "__hosted_agent_adapter_hitl__",
      "call_id": "<call_id>",
      "arguments": "{\"agent_request\":{\"request_id\":\"<request_id>\",...}}"
    }
  ]
}
```

Capture three values from the response:

- `conversation.id`
- The `call_id` of the `__hosted_agent_adapter_hitl__` function call
- The `request_id` inside the serialized `agent_request`

Respond by sending a `CreateResponse` request with `function_call_output` message that carries your review decision. Replace the placeholders before running the command:

```json
{
  "agent": {"name": "local_agent", "type": "agent_reference"},
  "stream": false,
  "convseration": {"id": "<conversation_id>"},
  "input": [
    {
      "call_id": "<call_id>",
      "output": "{\"request_id\":\"<request_id>\",\"approved\":true,\"feedback\":\"approve\"}",
      "type": "function_call_output",
    }
  ]
}
```

## Deploying the agent to Microsoft Foundry

Follow the hosted agent deployment guide at https://aka.ms/azdaiagent/docs to:

1. Configure the Azure Developer CLI and authenticate with your Azure subscription.
2. Build the container image (use `azd` cloud build or `docker build --platform=linux/amd64 ...`).
3. Publish the image to Azure Container Registry.
4. Create a hosted agent version and deployment inside your Azure AI Foundry project.

</details>

## Troubleshooting

### Images built on Apple Silicon or other ARM64 machines do not work on the service

We **recommend using `azd` cloud build**, which always produces a `linux/amd64` image.

If you must build locally on non-`amd64` hardware (for example, Apple Silicon), force the correct architecture:

```bash
docker build --platform=linux/amd64 -t image .
```

This ensures the hosted agent runs correctly in Microsoft Foundry.
