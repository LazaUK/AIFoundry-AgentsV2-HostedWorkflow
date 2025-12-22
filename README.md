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
- [Code Sample: YAML Definition]()
- [Code Sample: Python Script]()
- [Deployment of Hosted Workflow]()
- [Execution in Azure AI Foundry]()

## Use-Case Scenario: Code Review
This sample demonstrates a _Code Review_ workflow involving two specialised agents:
- **Developer Agent**: writes code to solve problems and may occasionally introduce bugs,
- **Reviewer Agent**: checks the code for style and correctness.

The workflow continues in a loop until the Reviewer provides the `approved` keyword, at which point the final solution is delivered then to the user.

## Code Sample: YAML Definition
The workflow logic is defined declaratively in CodeReview.yaml. This file describes the triggers, variables, and the sequence of agent invocations using the InvokeAzureAgent action.

``` YAML
kind: workflow
trigger:
  kind: OnConversationStart
  actions:
    - kind: InvokeAzureAgent
      id: invoke_developer
      agent: { name: DeveloperAgent }
      # ... input/output mapping ...
    - kind: InvokeAzureAgent
      id: invoke_reviewer
      agent: { name: ReviewerAgent }
      # ... feedback loop logic ...
    - kind: ConditionGroup
      id: check_approved
      conditions:
        - condition: =!IsBlank(Find("approved", Lower(Last(Local.LatestMessage).Text)))
          actions:
            - kind: EndConversation
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
