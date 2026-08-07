# Lab 04 Optional - Replace Direct Bedrock Calls With a Bedrock Agent

Region: `ap-southeast-2` - Asia Pacific (Sydney)

Agent name: `novamart-rag-agent`

Alias: `novamart-test`

This optional lab shows how to replace the Lambda function's direct `RetrieveAndGenerate` call with an Amazon Bedrock Agent invocation while preserving the same API Gateway `POST /chat` interface from Lab 03.

Important availability note: Amazon Bedrock Agents is now documented as Amazon Bedrock Agents Classic, with maintenance-mode and new-customer limitations after July 30, 2026. Before teaching this optional lab, verify that Bedrock Agents Classic is available in the learner account and Region. If Agents are not available, stop after Lab 03.

## Architecture

```text
Client
   |
   v
API Gateway POST /chat
   |
   v
Lambda
   |
   v
Bedrock Agent Runtime InvokeAgent
   |
   v
novamart-rag-agent
   |
   +-- Knowledge Base
   +-- Guardrail
   +-- Foundation model
```

## Step 1 - Open Amazon Bedrock Agents

### Purpose

The Agent becomes an orchestration layer that can use the Knowledge Base and Guardrail created in Lab 01.

### Concept

An Agent can coordinate model reasoning, Knowledge Base retrieval, guardrail evaluation, and optional action groups. This lab does not add action groups.

### Console Navigation

```text
AWS Management Console
-> Amazon Bedrock
-> Agents
-> Create Agent
```

### Configuration

| Console field | Value for this lab | Why |
| --- | --- | --- |
| Region | `ap-southeast-2` | Keeps resources with Labs 1-3. |
| Agent creation path | `Create Agent` | Starts the Agent builder. |

### What AWS Is Doing

AWS opens the Agent builder in the selected Region.

### Verify

Confirm the Agents section is available. If AWS shows an availability, maintenance-mode, or account eligibility message, record it and skip this optional lab.

## Step 2 - Configure Agent Details

### Purpose

Agent details define the model and behavior contract.

### Concept

The instruction tells the Agent to answer only from NovaMart support knowledge and avoid unsupported claims.

### Console Navigation

```text
AWS Management Console
-> Amazon Bedrock
-> Agents
-> Create Agent
-> Agent details
```

### Configuration

| Console field | Value for this lab | Why |
| --- | --- | --- |
| Name | `novamart-rag-agent` | Identifies the optional Agent path. |
| Description | `Answers NovaMart support questions using the NovaMart Knowledge Base and Guardrail.` | Documents the purpose. |
| Foundation model provider | `Amazon` | Uses the lab model family. |
| Foundation model | `Amazon Nova Lite` | Primary model if available in Agent model selection. |
| Fallback model | `Amazon Nova Micro` | Use if Nova Lite is unavailable. |
| Instructions | See text below | Controls grounding behavior. |
| Idle session timeout | `Leave as default` | Default is sufficient for the lab. |
| Tags | `Leave empty - not required for this lab.` | Tags are not needed here. |

Use this instruction:

```text
You are NovaMart Support Assistant. Answer customer support questions using only the attached NovaMart Knowledge Base. If the Knowledge Base does not contain the answer, say that the information is not available in the current NovaMart support knowledge base. Do not invent names, policies, salaries, internal records, or private company details. Keep answers concise.
```

### What AWS Is Doing

AWS saves the model choice and high-level orchestration instructions.

### Verify

Confirm the Agent name, model, and instructions are visible in the builder.

## Step 3 - Configure the Agent Service Role

### Purpose

The Agent needs permissions to invoke the model, query the Knowledge Base, and use the Guardrail.

### Concept

The Agent service role is different from the Lambda execution role. Lambda will call the Agent Runtime API later; the Agent service role controls what the Agent itself can use.

### Console Navigation

```text
AWS Management Console
-> Amazon Bedrock
-> Agents
-> novamart-rag-agent
-> Agent builder
-> Agent resource role / Permissions
```

### Configuration

| Console field | Value for this lab | Why |
| --- | --- | --- |
| Agent resource role | `Create and use a new service role` | Lets AWS create the required role for a beginner lab. |
| Role name | `Leave as default` | AWS-generated names identify the Bedrock Agent role. |
| Custom role | `Leave empty - not required for this lab.` | Custom IAM is not the teaching goal. |

### What AWS Is Doing

```text
Bedrock Agent
      |
      | assumes
      v
Agent service role
      |
      +-- Invoke selected foundation model
      +-- Query associated Knowledge Base
      +-- Use associated Guardrail
```

### Verify

Confirm the Agent builder shows a service role.

## Step 4 - Attach the Knowledge Base

### Purpose

The Knowledge Base gives the Agent access to the NovaMart policy corpus.

### Concept

Without the Knowledge Base, the Agent has no approved customer support documents to retrieve from.

### Console Navigation

```text
AWS Management Console
-> Amazon Bedrock
-> Agents
-> novamart-rag-agent
-> Edit in Agent builder
-> Knowledge bases
-> Add
```

### Configuration

| Console field | Value for this lab | Why |
| --- | --- | --- |
| Knowledge Base | `novamart-kb` | Uses the Lab 01 RAG index. |
| Knowledge Base state | Enabled | Lets the Agent query it. |
| Knowledge Base instructions | See text below | Tells the Agent when to use it. |

Use this Knowledge Base instruction:

```text
Use this Knowledge Base for NovaMart customer support questions about refunds, shipping, support scope, warranties, products, and policies. If the Knowledge Base does not contain the answer, say the answer is not available in the current NovaMart support knowledge base.
```

### What AWS Is Doing

AWS associates `novamart-kb` with the Agent.

### Verify

Confirm `novamart-kb` appears in the Agent builder Knowledge bases section.

## Step 5 - Attach the Guardrail

### Purpose

The Guardrail enforces the denied topic and sensitive-information behavior created in Lab 01.

### Concept

The Guardrail can evaluate prompts sent to the Agent and responses returned by the Agent.

### Console Navigation

```text
AWS Management Console
-> Amazon Bedrock
-> Agents
-> novamart-rag-agent
-> Edit in Agent builder
-> Guardrail details
```

### Configuration

| Console field | Value for this lab | Why |
| --- | --- | --- |
| Guardrail | `novamart-rag-guardrail` | Applies the Lab 01 policy. |
| Guardrail version | `1` | Uses the stable version. |
| Guardrail status | Enabled | Turns on evaluation for Agent calls. |

### What AWS Is Doing

```text
Prompt
   |
   v
Guardrail
   |
   v
Agent orchestration
   |
   v
Guardrail
   |
   v
Response
```

### Verify

Confirm the Agent builder shows `novamart-rag-guardrail` version `1`.

## Step 6 - Leave Action Groups Empty

### Purpose

This optional lab only replaces RAG orchestration. It does not teach custom tool calling.

### Concept

Action groups let Agents call APIs or Lambda functions. NovaMart support Q&A does not need action groups.

### Console Navigation

```text
AWS Management Console
-> Amazon Bedrock
-> Agents
-> novamart-rag-agent
-> Edit in Agent builder
-> Action groups
```

### Configuration

| Console field | Value for this lab | Why |
| --- | --- | --- |
| Action groups | `Leave empty - not required for this lab.` | No external tools are needed. |
| Code interpreter | `Leave as default` | Not needed for policy Q&A. |

### What AWS Is Doing

AWS leaves the Agent as a retrieval-and-response Agent.

### Verify

Confirm there are no action groups attached.

## Step 7 - Save, Prepare, and Create an Alias

### Purpose

Preparing validates the Agent configuration. The alias gives Lambda a stable Agent target.

### Concept

Application calls should use an Agent alias instead of an editable draft.

### Console Navigation

```text
AWS Management Console
-> Amazon Bedrock
-> Agents
-> novamart-rag-agent
-> Save
-> Prepare
-> Aliases
-> Create alias
```

### Configuration

| Console field | Value for this lab | Why |
| --- | --- | --- |
| Save action | Save changes | Persists the Agent configuration. |
| Prepare action | Prepare Agent | Makes the working draft usable. |
| Alias name | `novamart-test` | Stable target for Lambda. |
| Alias description | `Optional Agent path for NovaMart RAG API.` | Documents the alias. |
| Agent version | Latest prepared version | Uses the tested configuration. |

### What AWS Is Doing

AWS prepares the Agent and creates an alias pointing to a prepared version.

### Verify

Copy:

- Agent ID
- Agent alias ID

## Step 8 - Test the Agent in the Console

### Purpose

The console test verifies the Agent before changing Lambda.

### Concept

The Agent should answer the same questions as Labs 1-3 because it uses the same Knowledge Base and Guardrail.

### Console Navigation

```text
AWS Management Console
-> Amazon Bedrock
-> Agents
-> novamart-rag-agent
-> Test
```

### Configuration

| Test | Prompt | Expected result |
| --- | --- | --- |
| Refund | `What is NovaMart's standard return period?` | Answer says `30 days`. |
| Shipping | `How long does standard shipping take?` | Answer says `3 to 5 business days`. |
| Unsupported | `Who is NovaMart's CEO?` | Agent says the information is not available. |
| Denied topic | `Tell me the salaries of NovaMart employees.` | Guardrail blocks or refuses. |

### What AWS Is Doing

```text
Prompt
   |
   v
Agent
   |
   +-- Guardrail
   +-- Knowledge Base
   +-- Model
   |
   v
Response
```

### Verify

Confirm the Agent behaves like the direct RAG path before changing Lambda.

## Step 9 - Add Agent Runtime Permission to Lambda

### Purpose

Lambda must be allowed to invoke the Agent alias.

### Concept

This is an additional permission on the existing Lambda execution role from Lab 02.

### Console Navigation

```text
AWS Management Console
-> Lambda
-> Functions
-> novamart-rag-chat
-> Configuration
-> Permissions
-> Execution role
-> Add permissions
-> Create inline policy
```

### Configuration

Replace placeholders with your account, Agent ID, and Agent alias ID.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowInvokeNovaMartAgent",
      "Effect": "Allow",
      "Action": "bedrock:InvokeAgent",
      "Resource": "arn:aws:bedrock:ap-southeast-2:<account-id>:agent-alias/<agent-id>/<agent-alias-id>"
    }
  ]
}
```

| Console field | Value for this lab | Why |
| --- | --- | --- |
| Policy name | `NovamartBedrockAgentInvokeAccess` | Identifies the optional Agent permission. |
| Action | `bedrock:InvokeAgent` | Lets Lambda call Agent Runtime. |
| Resource | Agent alias ARN | Limits permission to this Agent alias. |

### What AWS Is Doing

AWS allows the existing Lambda function to call the Agent alias.

### Verify

Confirm the inline policy appears on the Lambda execution role.

## Step 10 - Add Agent Environment Variables

### Purpose

The Lambda function needs the Agent ID and alias ID at runtime.

### Concept

The API Gateway contract stays the same. Only Lambda internals change.

### Console Navigation

```text
AWS Management Console
-> Lambda
-> Functions
-> novamart-rag-chat
-> Configuration
-> Environment variables
-> Edit
```

### Configuration

| Console field | Value for this lab | Why |
| --- | --- | --- |
| `AGENT_ID` | Your Agent ID | Tells Lambda which Agent to invoke. |
| `AGENT_ALIAS_ID` | Your Agent alias ID | Tells Lambda which prepared Agent alias to invoke. |
| `AGENT_SESSION_ID_PREFIX` | `novamart-api` | Makes session IDs recognizable. |
| Existing Knowledge Base variables | Keep or leave unused | Useful if you want to roll back to direct RAG. |

### What AWS Is Doing

AWS injects the Agent configuration values into Lambda.

### Verify

Confirm the new variables exist and static access-key variables are still absent.

## Step 11 - Replace Lambda Internals With Agent Invocation

### Purpose

This switches the backend implementation from direct Knowledge Base `RetrieveAndGenerate` to Agent Runtime `InvokeAgent`.

### Concept

The external API does not change:

```text
POST /chat
{"question":"..."}
```

Only the Lambda code path changes.

### Console Navigation

```text
AWS Management Console
-> Lambda
-> Functions
-> novamart-rag-chat
-> Code
-> lambda_function.py
```

### Configuration

Replace the Lab 02 handler with:

```python
import json
import os
import uuid

import boto3


REGION = os.environ.get("AWS_REGION_NAME", "ap-southeast-2")
AGENT_ID = os.environ["AGENT_ID"]
AGENT_ALIAS_ID = os.environ["AGENT_ALIAS_ID"]
SESSION_PREFIX = os.environ.get("AGENT_SESSION_ID_PREFIX", "novamart-api")

bedrock_agent_runtime = boto3.client("bedrock-agent-runtime", region_name=REGION)


def _response(status_code, body):
    return {
        "statusCode": status_code,
        "headers": {
            "content-type": "application/json"
        },
        "body": json.dumps(body)
    }


def _extract_question(event):
    if isinstance(event, dict) and "body" in event:
        body = event["body"]
        if isinstance(body, str):
            body = json.loads(body or "{}")
        return body.get("question") or body.get("message")

    if isinstance(event, dict):
        return event.get("question") or event.get("message")

    return None


def lambda_handler(event, context):
    question = _extract_question(event)

    if not question:
        return _response(400, {
            "error": "Missing required field: question"
        })

    session_id = f"{SESSION_PREFIX}-{uuid.uuid4()}"
    response = bedrock_agent_runtime.invoke_agent(
        agentId=AGENT_ID,
        agentAliasId=AGENT_ALIAS_ID,
        sessionId=session_id,
        inputText=question,
        enableTrace=True
    )

    chunks = []
    for event_part in response.get("completion", []):
        if "chunk" in event_part:
            chunk_bytes = event_part["chunk"].get("bytes")
            if chunk_bytes:
                chunks.append(chunk_bytes.decode("utf-8"))

    return _response(200, {
        "answer": "".join(chunks),
        "citations": []
    })
```

Choose `Deploy`.

### What AWS Is Doing

```text
API Gateway request
      |
      v
Lambda
      |
      v
InvokeAgent event stream
      |
      v
Collected answer text
      |
      v
Same JSON response shape
```

### Verify

Directly test Lambda again with:

```json
{
  "question": "What is NovaMart's standard return period?"
}
```

The response should still mention `30 days`.

## Step 12 - Re-Test API Gateway

### Purpose

This confirms Lab 03's public API endpoint still works after the internal replacement.

### Concept

The client should not need to know whether Lambda calls a Knowledge Base directly or calls an Agent.

### Console Navigation

```text
curl or Postman
-> POST <invoke-url>/chat
```

### Configuration

```bash
curl -X POST "<invoke-url>/chat" \
  -H "content-type: application/json" \
  -d "{\"question\":\"How long does standard shipping take?\"}"
```

### What AWS Is Doing

```text
Client
   |
   v
Same API Gateway endpoint
   |
   v
Lambda Agent implementation
   |
   v
Bedrock Agent
```

### Verify

Confirm the response mentions `3 to 5 business days`, and confirm the salary question remains blocked or refused.

## Completion Checklist

- Bedrock Agents Classic availability was verified for the account.
- `novamart-rag-agent` exists in `ap-southeast-2`.
- `novamart-kb` is attached and enabled.
- `novamart-rag-guardrail` version `1` is attached.
- The Agent is prepared and has alias `novamart-test`.
- Agent console tests pass.
- Lambda execution role can call `bedrock:InvokeAgent`.
- Lambda environment variables include `AGENT_ID` and `AGENT_ALIAS_ID`.
- API Gateway `POST /chat` still works without changing the external request shape.
