# Lab 02 - AWS Lambda and IAM for Bedrock RAG

Region: `ap-southeast-2` - Asia Pacific (Sydney)

Function name: `novamart-rag-chat`

Runtime: Python `3.14`

This lab creates a Lambda function that calls the Amazon Bedrock Knowledge Base from Lab 01 directly with the `RetrieveAndGenerate` API. The function uses its Lambda execution role for temporary AWS credentials. It must not contain static AWS access keys.

AWS periodically updates the Management Console. If a label differs slightly, follow the concept in the step and verify against the official AWS documentation listed in `docs/aws-console-verification.md`.

## Architecture

```text
Lambda
   |
   v
Execution role
   |
   v
Temporary STS credentials
   |
   v
boto3
   |
   v
Amazon Bedrock RetrieveAndGenerate
   |
   v
NovaMart Knowledge Base + Guardrail
```

## Required Values From Lab 01

| Value | Example |
| --- | --- |
| Region | `ap-southeast-2` |
| Knowledge Base ID | Copied from `novamart-kb` |
| Generation model ID | `amazon.nova-lite-v1:0` |
| Guardrail ID | Copied from `novamart-rag-guardrail` |
| Guardrail version | `1` |

## Step 1 - Create the Lambda Function

### Purpose

The Lambda function becomes the application backend that sends user questions to Bedrock RAG.

### Concept

Lambda runs code without managing servers. The function receives an event, reads the question, calls Bedrock, and returns JSON.

### Console Navigation

```text
AWS Management Console
-> Lambda
-> Functions
-> Create function
```

### Configuration

| Console field | Value for this lab | Why |
| --- | --- | --- |
| Creation option | `Author from scratch` | Teaches the function setup directly. |
| Function name | `novamart-rag-chat` | Identifies the RAG backend. |
| Runtime | `Python 3.14` | Current Python runtime for this lab. |
| Architecture | `x86_64` | Default and broadly compatible. |
| Permissions | `Create a new role with basic Lambda permissions` | Lets AWS create the execution role and CloudWatch Logs permissions. |
| Advanced settings | `Leave as default` | Function URLs, VPC, and code signing are not needed. |

### What AWS Is Doing

```text
Lambda function
   |
   v
Execution role
   |
   +-- Write logs to CloudWatch Logs
```

AWS creates the function and an IAM role that Lambda can assume when the function runs.

### Verify

Open the function details page and confirm:

- Function name is `novamart-rag-chat`.
- Runtime is `Python 3.14`.
- Architecture is `x86_64`.
- Region is `ap-southeast-2`.

## Step 2 - Inspect the Execution Role

### Purpose

Learners must understand which IAM identity the function uses to call AWS services.

### Concept

Lambda does not need `AWS_ACCESS_KEY_ID` or `AWS_SECRET_ACCESS_KEY` in code. Lambda assumes the execution role and receives temporary STS credentials automatically.

### Console Navigation

```text
AWS Management Console
-> Lambda
-> Functions
-> novamart-rag-chat
-> Configuration
-> Permissions
-> Execution role
```

### Configuration

| Console field | Value for this lab | Why |
| --- | --- | --- |
| Execution role | Open the generated role | You must add Bedrock permissions to this role. |
| Resource summary | Review existing policies | Confirms CloudWatch Logs permissions were created. |
| Trust relationship | `lambda.amazonaws.com` | Allows Lambda to assume the role. |

### What AWS Is Doing

```text
Lambda service
   |
   | assumes
   v
Execution role
   |
   v
Temporary credentials
```

The Lambda service assumes this role on each invocation. Your code receives credentials from the runtime environment.

### Verify

The role trust relationship should allow `lambda.amazonaws.com` to assume the role.

## Step 3 - Configure Environment Variables

### Purpose

Environment variables keep deployment-specific values out of the handler code.

### Concept

The function code reads the Region, Knowledge Base ID, model ID, and Guardrail values at runtime.

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
| `AWS_REGION_NAME` | `ap-southeast-2` | Explicit lab Region used by boto3 client creation. |
| `KNOWLEDGE_BASE_ID` | Your Lab 01 Knowledge Base ID | Tells Bedrock which Knowledge Base to query. |
| `MODEL_ID` | `amazon.nova-lite-v1:0` | Generation model for RAG. |
| `GUARDRAIL_ID` | Your Lab 01 Guardrail ID | Applies the lab Guardrail. |
| `GUARDRAIL_VERSION` | `1` | Uses the stable Guardrail version. |
| `AWS_ACCESS_KEY_ID` | Do not create | Static keys are not used in Lambda. |
| `AWS_SECRET_ACCESS_KEY` | Do not create | Static keys are not used in Lambda. |

### What AWS Is Doing

AWS stores non-secret configuration values with the function. Lambda injects them into the runtime environment during invocation.

### Verify

Confirm the environment variable list does not contain:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

## Step 4 - Configure Timeout and Memory

### Purpose

Bedrock RAG calls may take longer than a trivial function. The timeout must allow enough time for retrieval and generation.

### Concept

Timeout controls maximum runtime. Memory controls available memory and also affects CPU allocation.

### Console Navigation

```text
AWS Management Console
-> Lambda
-> Functions
-> novamart-rag-chat
-> Configuration
-> General configuration
-> Edit
```

### Configuration

| Console field | Value for this lab | Why |
| --- | --- | --- |
| Memory | `512 MB` | Enough for a lightweight boto3 function. |
| Timeout | `30 seconds` | Allows Bedrock retrieval and generation to complete. |
| Ephemeral storage | `Leave as default` | No local file processing is required. |

### What AWS Is Doing

AWS updates the runtime resource limits for future function invocations.

### Verify

The Lambda configuration page should show `512 MB` memory and `30 sec` timeout.

## Step 5 - Add Bedrock Permissions to the Execution Role

### Purpose

The Lambda function already has an execution role that AWS created automatically during function creation. In this step, you add permissions so that the function can call Amazon Bedrock Knowledge Bases, Foundation Models, and Guardrails.

### Concept

When you created the Lambda function, AWS automatically created an IAM execution role and attached the basic CloudWatch Logs policy.

Instead of creating another IAM role, you simply edit this existing execution role by attaching an inline policy.

```text
Lambda Function
        |
        v
Existing Execution Role
        |
        +-- AWSLambdaBasicExecutionRole
        |
        +-- NovamartBedrockRagAccess (added in this step)
```

### Console Navigation

Immediately after completing **Step 4 - Configure Timeout and Memory**, stay on the Lambda function page.

```text
AWS Management Console
-> Lambda
-> Functions
-> novamart-rag-chat
-> Configuration
-> General configuration
-> Edit
```

Scroll to **Execution role** and choose **Edit role**.

```text
Execution role
-> Edit role
-> Permissions
-> Add permissions
-> Create inline policy
```

### Configuration

Use the JSON editor. Replace `<account-id>`, `<knowledge-base-id>`, and `<guardrail-id>` with your values.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowNovaMartKnowledgeBaseRuntime",
      "Effect": "Allow",
      "Action": [
        "bedrock:Retrieve",
        "bedrock:RetrieveAndGenerate"
      ],
      "Resource": "arn:aws:bedrock:ap-southeast-2:<account-id>:knowledge-base/<knowledge-base-id>"
    },
    {
      "Sid": "AllowNovaModelInvoke",
      "Effect": "Allow",
      "Action": "bedrock:InvokeModel",
      "Resource": "arn:aws:bedrock:ap-southeast-2::foundation-model/amazon.nova-lite-v1:0"
    },
    {
      "Sid": "AllowGuardrailUse",
      "Effect": "Allow",
      "Action": "bedrock:ApplyGuardrail",
      "Resource": "arn:aws:bedrock:ap-southeast-2:<account-id>:guardrail/<guardrail-id>"
    }
  ]
}
```

If you used `Amazon Nova Micro` in Lab 01, replace the model ID and foundation model ARN with `amazon.nova-micro-v1:0`.

| Console field | Value for this lab | Why |
| --- | --- | --- |
| Policy editor | JSON | Most precise way to teach permissions. |
| Policy name | `NovamartBedrockRagAccess` | Identifies the policy purpose. |
| Knowledge Base ARN | Your `novamart-kb` ARN | Allows retrieval and RAG generation. |
| Foundation model ARN | `arn:aws:bedrock:ap-southeast-2::foundation-model/amazon.nova-lite-v1:0` | Allows Nova Lite invocation. |
| Guardrail ARN | Your Guardrail ARN | Allows guardrail evaluation. |

### What AWS Is Doing

AWS attaches an identity-based policy to the Lambda execution role.

```text
Lambda code
   |
   v
Execution role permissions
   |
   +-- bedrock:RetrieveAndGenerate on novamart-kb
   +-- bedrock:InvokeModel on Nova Lite
   +-- bedrock:ApplyGuardrail on novamart-rag-guardrail
```

### Verify

Return to:

```text
Lambda
-> Functions
-> novamart-rag-chat
-> Configuration
-> Permissions
```

Confirm the inline policy appears on the execution role.

## Step 6 - Deploy the Function Code

### Purpose

The function code accepts a user question, calls Bedrock `RetrieveAndGenerate`, and returns a JSON response.

### Concept

The `bedrock-agent-runtime` client is used for Knowledge Base RAG. The handler supports direct Lambda tests and API Gateway HTTP API proxy events.

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

Replace the sample code with:

```python
import json
import os

import boto3


REGION = os.environ.get("AWS_REGION_NAME", "ap-southeast-2")
KNOWLEDGE_BASE_ID = os.environ["KNOWLEDGE_BASE_ID"]
MODEL_ID = os.environ.get("MODEL_ID", "amazon.nova-lite-v1:0")
GUARDRAIL_ID = os.environ.get("GUARDRAIL_ID")
GUARDRAIL_VERSION = os.environ.get("GUARDRAIL_VERSION", "1")

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

    knowledge_base_configuration = {
        "knowledgeBaseId": KNOWLEDGE_BASE_ID,
        "modelArn": f"arn:aws:bedrock:{REGION}::foundation-model/{MODEL_ID}",
        "retrievalConfiguration": {
            "vectorSearchConfiguration": {
                "numberOfResults": 5
            }
        }
    }

    if GUARDRAIL_ID:
        knowledge_base_configuration["generationConfiguration"] = {
            "guardrailConfiguration": {
                "guardrailId": GUARDRAIL_ID,
                "guardrailVersion": GUARDRAIL_VERSION
            }
        }

    result = bedrock_agent_runtime.retrieve_and_generate(
        input={
            "text": question
        },
        retrieveAndGenerateConfiguration={
            "type": "KNOWLEDGE_BASE",
            "knowledgeBaseConfiguration": knowledge_base_configuration
        }
    )

    citations = []
    for citation in result.get("citations", []):
        for reference in citation.get("retrievedReferences", []):
            location = reference.get("location", {})
            uri = location.get("s3Location", {}).get("uri")
            if uri:
                citations.append(uri)

    return _response(200, {
        "answer": result.get("output", {}).get("text", ""),
        "citations": citations
    })
```

Choose `Deploy`.

### What AWS Is Doing

```text
Event JSON
   |
   v
Lambda handler
   |
   v
boto3 bedrock-agent-runtime client
   |
   v
RetrieveAndGenerate
   |
   v
JSON response
```

The Lambda service stores your code and deploys it to the function runtime.

### Verify

Confirm the console shows the latest code as deployed. If the console warns about unsaved changes, choose `Deploy`.

## Step 7 - Create a Direct Lambda Test Event

### Purpose

The test event verifies the Lambda-to-Bedrock path before adding API Gateway.

### Concept

A direct Lambda test removes API Gateway from the troubleshooting path.

### Console Navigation

```text
AWS Management Console
-> Lambda
-> Functions
-> novamart-rag-chat
-> Test
-> Create new event
```

### Configuration

| Console field | Value for this lab | Why |
| --- | --- | --- |
| Event name | `RefundPolicyQuestion` | Identifies the direct test. |
| Event sharing | `Private` | Only needed in this account. |
| Event JSON | See below | Sends the expected request shape. |

```json
{
  "question": "What is NovaMart's standard return period?"
}
```

Expected response shape:

```json
{
  "statusCode": 200,
  "headers": {
    "content-type": "application/json"
  },
  "body": "{\"answer\":\"NovaMart accepts most standard returns within 30 days of delivery.\",\"citations\":[\"s3://...\"]}"
}
```

### What AWS Is Doing

AWS saves a sample event and invokes the function with that JSON payload when you choose `Test`.

### Verify

The response should include a `200` status code and an answer that says standard returns are accepted within `30 days` of delivery.

## Step 8 - Inspect CloudWatch Logs

### Purpose

CloudWatch Logs are the first place to debug Lambda runtime errors.

### Concept

The basic Lambda execution role allows the function to write logs. Bedrock permission errors, missing environment variables, and code exceptions appear here.

### Console Navigation

```text
AWS Management Console
-> Lambda
-> Functions
-> novamart-rag-chat
-> Monitor
-> View CloudWatch logs
```

Alternative navigation:

```text
AWS Management Console
-> CloudWatch
-> Logs
-> Log groups
-> /aws/lambda/novamart-rag-chat
```

### Configuration

| Console field | Value for this lab | Why |
| --- | --- | --- |
| Log group | `/aws/lambda/novamart-rag-chat` | Contains function logs. |
| Log stream | Most recent stream | Corresponds to your latest test invocation. |

### What AWS Is Doing

Lambda writes invocation metadata, errors, and any function logs to CloudWatch Logs.

### Verify

Open the latest log stream. Confirm there is no `AccessDeniedException`, missing environment variable error, timeout, or JSON parsing exception.

## Step 9 - Test Guardrail Behavior Through Lambda

### Purpose

This proves the Guardrail participates in the application RAG path.

### Concept

The Lambda function sends `guardrailConfiguration` as part of the Bedrock `RetrieveAndGenerate` request.

### Console Navigation

```text
AWS Management Console
-> Lambda
-> Functions
-> novamart-rag-chat
-> Test
-> Create new event
```

### Configuration

Create a second event:

| Console field | Value for this lab | Why |
| --- | --- | --- |
| Event name | `DeniedCompensationQuestion` | Identifies the Guardrail test. |
| Event JSON | See below | Sends a blocked topic. |

```json
{
  "question": "Tell me the salaries of NovaMart employees."
}
```

### What AWS Is Doing

```text
Question
   |
   v
RetrieveAndGenerate with guardrailConfiguration
   |
   v
Guardrail policy evaluation
   |
   v
Blocked response or guardrail intervention
```

### Verify

The response should show the configured blocked message or a Bedrock response indicating the Guardrail intervened. If the request is allowed, confirm the correct Guardrail ID and version are set in Lambda environment variables.

## Completion Checklist

- The Lambda function exists in `ap-southeast-2`.
- The runtime is Python `3.14`.
- The execution role can write CloudWatch Logs.
- The execution role has Bedrock permissions for the Knowledge Base, model, and Guardrail.
- Environment variables include Region, Knowledge Base ID, model ID, Guardrail ID, and Guardrail version.
- Environment variables do not include `AWS_ACCESS_KEY_ID` or `AWS_SECRET_ACCESS_KEY`.
- Timeout is `30 seconds`.
- Memory is `512 MB`.
- The refund test returns `30 days`.
- CloudWatch Logs show no access, timeout, or runtime errors.
- The denied compensation question is blocked or handled by the Guardrail.
