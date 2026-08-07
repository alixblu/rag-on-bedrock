# AWS Console Verification Notes

Verification date: 2026-08-05

Primary lab Region: `ap-southeast-2` - Asia Pacific (Sydney)

These labs are written from official AWS documentation and should be checked against the live AWS Management Console before classroom delivery. AWS periodically updates console labels and navigation. If a label differs slightly, follow the concept in the step and confirm against the linked AWS documentation.

## Core Defaults

| Area | Lab default | Notes |
| --- | --- | --- |
| AWS Region | `ap-southeast-2` | All resources should be created in Sydney unless a step explicitly says otherwise. |
| Text/chat model | `Amazon Nova Lite` / `amazon.nova-lite-v1:0` | Use this model where available in the Sydney Bedrock model catalog. |
| Text/chat fallback | `Amazon Nova Micro` / `amazon.nova-micro-v1:0` | Use only if Nova Lite is not active or available in the learner account. |
| Embeddings model | `Amazon Titan Text Embeddings V2` / `amazon.titan-embed-text-v2:0` | Used by the knowledge base ingestion pipeline. |
| Parser | Default parser | Bedrock Data Automation parser is not the main path for this Sydney lab. |
| Vector store | Quick create with Amazon S3 Vectors | Keeps the first lab focused on Bedrock Knowledge Bases rather than manual vector database administration. |
| Guardrail version | `DRAFT`, then version `1` | Test while drafting, then use a numbered version for application calls. |
| Lambda function | `novamart-rag-chat` | Directly calls Bedrock Knowledge Bases in Lab 02. |
| API Gateway API | `novamart-rag-api` | Exposes Lambda through `POST /chat` in Lab 03. |
| Optional Agent | `novamart-rag-agent` | Optional Lab 04 replacement for direct Lambda `RetrieveAndGenerate` calls. |

## Official AWS Documentation Checked

Core Bedrock and S3:

- Amazon Bedrock supported foundation models: https://docs.aws.amazon.com/bedrock/latest/userguide/models-supported.html
- Amazon Bedrock model availability and compatibility: https://docs.aws.amazon.com/bedrock/latest/userguide/models.html
- Amazon Bedrock model access behavior: https://docs.aws.amazon.com/bedrock/latest/userguide/model-access.html
- Amazon Bedrock foundation model usage reference: https://docs.aws.amazon.com/bedrock/latest/userguide/foundation-models-reference.html
- Create a Bedrock knowledge base with a vector store: https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base-create.html
- Knowledge base prerequisites: https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base-prereq.html
- Retrieve data and generate responses with Knowledge Bases: https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html
- Query a knowledge base and generate responses: https://docs.aws.amazon.com/bedrock/latest/userguide/kb-test-retrieve-generate.html
- Retrieve and Generate API: https://docs.aws.amazon.com/bedrock/latest/APIReference/API_agent-runtime_RetrieveAndGenerate.html
- Boto3 `retrieve_and_generate`: https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/bedrock-agent-runtime/client/retrieve_and_generate.html
- S3 create bucket user guide: https://docs.aws.amazon.com/AmazonS3/latest/userguide/create-bucket-overview.html
- S3 upload objects user guide: https://docs.aws.amazon.com/AmazonS3/latest/userguide/upload-objects.html

Core Lambda and API Gateway:

- Create your first Lambda function: https://docs.aws.amazon.com/lambda/latest/dg/getting-started.html
- Lambda Python functions: https://docs.aws.amazon.com/lambda/latest/dg/lambda-python.html
- Lambda execution roles: https://docs.aws.amazon.com/lambda/latest/dg/lambda-intro-execution-role.html
- Lambda environment variables: https://docs.aws.amazon.com/lambda/latest/dg/configuration-envvars.html
- Lambda logs in CloudWatch: https://docs.aws.amazon.com/lambda/latest/dg/monitoring-cloudwatchlogs.html
- View Lambda CloudWatch logs: https://docs.aws.amazon.com/lambda/latest/dg/monitoring-cloudwatchlogs-view.html
- Develop HTTP APIs in API Gateway: https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-develop.html
- HTTP API Lambda integrations: https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-develop-integrations-lambda.html
- HTTP API routes: https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-develop-routes.html
- HTTP API CORS: https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-cors.html

Optional Agent Lab 04:

- Bedrock Agents overview: https://docs.aws.amazon.com/bedrock/latest/userguide/agents.html
- Bedrock Agents Classic maintenance mode: https://docs.aws.amazon.com/bedrock/latest/userguide/agents-classic-maintenance-mode.html
- Create and configure an Amazon Bedrock Agent: https://docs.aws.amazon.com/bedrock/latest/userguide/agents-create.html
- Add a Knowledge Base to an Agent: https://docs.aws.amazon.com/bedrock/latest/userguide/agents-kb-add.html
- Associate a Guardrail with an Agent: https://docs.aws.amazon.com/bedrock/latest/userguide/agents-guardrail.html
- Test and troubleshoot an Agent: https://docs.aws.amazon.com/bedrock/latest/userguide/agents-test.html
- Deploy an Agent with an alias: https://docs.aws.amazon.com/bedrock/latest/userguide/agents-deploy.html
- InvokeAgent API reference: https://docs.aws.amazon.com/bedrock/latest/APIReference/API_agent-runtime_InvokeAgent.html
- Boto3 `invoke_agent`: https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/bedrock-agent-runtime/client/invoke_agent.html

## Current Console Behavior Notes

- Bedrock model access is no longer a single universal "Request model access" workflow. AWS documentation distinguishes default Amazon model access, Marketplace-backed model permissions, and provider-specific setup for some third-party models.
- In Bedrock Knowledge Bases, the current console path documented by AWS is `Amazon Bedrock` -> `Knowledge bases` -> `Create` -> `Knowledge base with vector store`.
- Knowledge base testing can retrieve source chunks and can also generate a response from retrieved chunks when a generation model is configured.
- The `RetrieveAndGenerate` API queries a Knowledge Base and returns generated responses with citations to relevant source data.
- Lambda automatically sends logs to CloudWatch Logs when the execution role has the required logging permissions.
- HTTP APIs require at least one route, integration, stage, and deployment. The lab uses a Lambda integration and `POST /chat`.
- Guardrails can be tested while in `DRAFT`. For application calls, the labs use a numbered Guardrail version.
- Optional Lab 04 must be treated as optional because AWS documentation states Bedrock Agents Classic entered maintenance mode and is not open to new customers after July 30, 2026.

## Verification Checklist Before Learning

- Confirm `ap-southeast-2` is selected in the AWS console Region selector before creating any resource.
- Confirm `Amazon Nova Lite` is active and usable in the Bedrock model catalog for the learner account. If not, use `Amazon Nova Micro` consistently.
- Confirm `Amazon Titan Text Embeddings V2` is available for Knowledge Bases in `ap-southeast-2`.
- Confirm the Knowledge Base creation wizard offers the selected S3 data source, default parser, chunking controls, embeddings model selection, and quick vector store creation path.
- Confirm the Guardrails console exposes content filters, denied topics, word filters, sensitive information filters, test, and version creation.
- Confirm the Lambda console exposes function creation, runtime selection, execution role inspection, environment variables, timeout/memory configuration, code deployment, test events, and CloudWatch logs.
- Confirm the API Gateway console exposes HTTP API creation, Lambda integration, route configuration, stage/invoke URL, CORS, and Lambda invoke permission behavior.
- For optional Lab 04, confirm Bedrock Agents Classic is available in the learner account before learning the Agent replacement path.
