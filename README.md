# Microsoft Agent Framework: Hosting Agentic Workflows in Azure AI Foundry
This repo provides a practical implementation of _Hosted Workflows_ using the **Microsoft Agent Framework**'s Python SDK. Unlike local workflows, hosted workflows are deployed directly to Azure AI Foundry, allowing them to be managed, versioned and executed within the Azure ecosystem.

> [!WARNING]
> To successfully run these notebooks, you must have an **Azure AI Foundry** project and **AI model deployment**. Please ensure you have the following environment variables set up in your system:
> | Environment Variable             | Description                                                                     |
> | -------------------------------- | ------------------------------------------------------------------------------- |
> | `AZURE_FOUNDRY_PROJECT_ENDPOINT` | The endpoint URL for your Azure AI Foundry project.                             |
> | `AZURE_FOUNDRY_GPT_MODEL`        | The name of the model deployment to be used by the agent, e.g., _gpt-4.1-mini_. |

## 📑 Table of Contents
- [Use-Case Scenario: Code Review](#use-case-scenario-code-review)
- [Code Sample: YAML Definition](#code-sample-yaml-definition)
- [Code Sample: Python Script]()
- [Deployment of Hosted Workflow]()
- [Execution in Azure AI Foundry]()

## Use-Case Scenario: Code Review
This sample demonstrates a _Code Review_ workflow involving two specialised agents:
- **Developer Agent**: writes code to solve problems and may occasionally introduce bugs,
- **Reviewer Agent**: checks the code for style and correctness.

The workflow continues in a loop until the Reviewer provides the `approved` keyword, at which point the final solution is delivered then to the user.

## Code Sample: YAML Definition
The workflow logic is defined in `CodeReview.yaml` file, that describes the triggers, variables and the sequence of agent invocations.

### 1.1 Workflow Trigger
The workflow is initialised using the `OnConversationStart` trigger, that captures the user's input into a local variable (`Local.LatestMessage`) as the starting context for the agents.

``` YAML
kind: workflow
trigger:
  kind: OnConversationStart
  id: trigger_wf
  actions:
    - kind: SetVariable
      id: init_latest
      variable: Local.LatestMessage
      value: =UserMessage(System.LastMessageText)
```

### 1.2 Agent Invocation
The logic utilises the `InvokeAzureAgent` action to call specific agents registered in Azure AI Foundry.
- **DeveloperAgent**: receives the initial problem or previous feedback to generate code,
- **ReviewerAgent**: analyses the developer's output to provide feedback or approval,

Both agents map their outputs back to `Local.LatestMessage`, for the conversation state to persist across turns.

``` YAML
- kind: InvokeAzureAgent
  id: invoke_developer
  agent:
    name: DeveloperAgent
  conversationId: =System.ConversationId
  input:
    messages: =Local.LatestMessage
  output:
    messages: Local.LatestMessage
    autoSend: true
```

### 1.3 Conditions and Looping
To manage the _code review_ interactions, the workflow utilises a `ConditionGroup` logic.
- **Approval Check**: A condition searches for the string "_approved_" within the reviewer's last message.
- **Termination**: If approved, the workflow sends the final activity and ends the conversation.
- **Looping**: If not approved, a `GotoAction` redirects the flow back to the `invoke_developer` step.

``` YAML
- kind: ConditionGroup
  id: check_approved
  conditions:
    - id: if_approved
      condition: =!IsBlank(Find("approved", Lower(Last(Local.LatestMessage).Text)))
      actions:
        - kind: EndConversation
  elseActions:
    - kind: GotoAction
      id: loop_back
      actionId: invoke_developer
```

## Code Sample: Python Script
The hosted_workflow.py script acts as the deployment engine. It uses the AIProjectClient to register the agents and the workflow within your Azure project.

2.1 Registering Agents
Agents are created as versions within the project, allowing you to define their persona and underlying model deployment once and reuse them across different workflows.

Python

# Create Developer Agent
developer_agent = project_client.agents.create_version(
    agent_name="DeveloperAgent",
    definition=PromptAgentDefinition(
        model=model,
        instructions="You are a junior developer practicing code writing..."
    ),
)
2.2 Registering the Workflow
The YAML definition is uploaded to Azure AI Foundry as a WorkflowAgentDefinition. This makes the workflow "hosted," meaning it can be triggered via API calls to the project endpoint.

Python

workflow_agent = project_client.agents.create_version(
    agent_name=workflow_definition["name"],
    definition=WorkflowAgentDefinition(workflow=workflow_yaml),
)

## Deployment of Hosted Workflow
To deploy the workflow to your Azure AI Foundry project, run the following command. Ensure you have authenticated via the Azure CLI (az login) first.

Bash

# Install dependencies
pip install azure-ai-projects azure-identity pyyaml

# Run the deployment script
python hosted_workflow.py CodeReview.yaml

## Execution in Azure AI Foundry
Once deployed, you can manage and visualize your agents and workflows directly in the Azure AI Foundry portal:

Agents Section: View the DeveloperAgent and ReviewerAgent, inspect their instructions, and test them individually.

Workflows Section: Find the CodeReviewWorkflow. You can view the YAML structure and monitor execution logs.

Traceability: Each run generates a unique ConversationId, allowing you to trace the multi-turn interaction between the developer and reviewer agents.

Example Execution Trace:
Plaintext

Creating agents in Azure AI Foundry...
Created DeveloperAgent: agent_abc123
Created ReviewerAgent: agent_xyz789

Creating workflow in Azure AI Foundry...
Created workflow: wf_987654

Hosted Workflow Created Successfully!
Workflow ID: wf_987654
