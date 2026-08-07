# Lab 03 - Amazon API Gateway HTTP API for Bedrock RAG

Region: `ap-southeast-2` - Asia Pacific (Sydney)

API name: `novamart-rag-api`

Route: `POST /chat`

Integration target: Lambda function `novamart-rag-chat`

This lab exposes the Lambda function from Lab 02 through Amazon API Gateway. The result is an HTTP endpoint that accepts a chat question and returns a Bedrock RAG answer.

AWS periodically updates the Management Console. If a label differs slightly, follow the concept in the step and verify against the official AWS documentation listed in `docs/aws-console-verification.md`.

## Architecture

```text
curl or Postman
      |
      v
API Gateway HTTP API
      |
      | Lambda resource-based permission
      v
Lambda
      |
      | IAM execution role
      v
Bedrock Knowledge Base + Guardrail
      |
      v
Response
```

## Why HTTP API

Use API Gateway HTTP API for this lab because the goal is a simple `POST /chat` endpoint backed by Lambda. HTTP APIs are simpler than REST APIs for this use case and include built-in CORS support and automatic deployment behavior. Use REST API only when you specifically need REST API features such as usage plans, API keys, request validation, or advanced gateway transformations.

## Step 1 - Start Creating the API

### Purpose

API Gateway provides the public HTTPS entry point for the Lambda RAG backend.

### Concept

Clients should not invoke Lambda directly. API Gateway receives HTTP requests, applies API configuration, and invokes Lambda through an integration.

### Console Navigation

```text
AWS Management Console
-> API Gateway
-> Create API
-> HTTP API
-> Build
```

### Configuration

| Console field | Value for this lab | Why |
| --- | --- | --- |
| API type | `HTTP API` | Best fit for a simple Lambda-backed endpoint. |
| API name | `novamart-rag-api` | Identifies the lab API. |
| Region | `ap-southeast-2` | Must match the Lambda function Region. |

### What AWS Is Doing

AWS starts an HTTP API configuration. No route is useful until an integration is connected.

### Verify

Confirm the wizard or create screen says HTTP API, not REST API or WebSocket API.

## Step 2 - Configure the Lambda Integration

### Purpose

The integration tells API Gateway which backend to call when the route receives a request.

### Concept

A Lambda proxy integration passes request details to Lambda and returns the Lambda response to the client.

### Console Navigation

```text
AWS Management Console
-> API Gateway
-> Create API
-> HTTP API
-> Integrations
-> Add integration
-> Lambda
```

### Configuration

| Console field | Value for this lab | Why |
| --- | --- | --- |
| Integration type | `Lambda` | The backend is the Lab 02 function. |
| AWS Region | `ap-southeast-2` | Finds the Sydney Lambda function. |
| Lambda function | `novamart-rag-chat` | Calls the Bedrock RAG backend. |
| Payload format version | `2.0` or `Leave as default` | HTTP APIs commonly use payload format 2.0. |
| Grant API Gateway permission to invoke Lambda | Enabled if shown | Allows API Gateway to call the function. |

### What AWS Is Doing

```text
API Gateway route
      |
      v
Lambda proxy integration
      |
      v
novamart-rag-chat
```

AWS records the Lambda integration and prepares to add a Lambda resource-based permission.

### Verify

Confirm `novamart-rag-chat` is listed as the selected integration.

## Step 3 - Configure the Route

### Purpose

The route maps an HTTP method and path to the Lambda integration.

### Concept

`POST /chat` means clients send a JSON request body to the chat endpoint.

### Console Navigation

```text
AWS Management Console
-> API Gateway
-> Create API
-> HTTP API
-> Routes
```

### Configuration

| Console field | Value for this lab | Why |
| --- | --- | --- |
| Method | `POST` | Chat requests send JSON in the request body. |
| Resource path | `/chat` | Clear endpoint name for the RAG chat backend. |
| Integration target | `novamart-rag-chat` Lambda integration | Sends this route to Lambda. |

### What AWS Is Doing

```text
POST /chat
   |
   v
Lambda integration
```

API Gateway matches incoming `POST /chat` requests and forwards them to the Lambda integration.

### Verify

The route table should show `POST /chat` connected to the Lambda integration.

## Step 4 - Configure the Stage and Deployment Behavior

### Purpose

The stage provides the deployed URL clients call.

### Concept

HTTP APIs can automatically deploy route and integration changes to a stage. The default stage is often named `$default`.

### Console Navigation

```text
AWS Management Console
-> API Gateway
-> Create API
-> HTTP API
-> Stages
```

### Configuration

| Console field | Value for this lab | Why |
| --- | --- | --- |
| Stage | `$default` | Simple first-lab deployment path. |
| Auto-deploy | Enabled | Keeps route changes available without manual deployments. |
| Stage variables | `Leave empty - not required for this lab.` | The Lambda function already stores needed settings. |
| Access logging | `Leave as default` for the first run | Can be enabled later for operations. |

### What AWS Is Doing

AWS deploys the API configuration to a callable stage.

### Verify

After creation, open the API details page and find the invoke URL. It should look similar to:

```text
https://<api-id>.execute-api.ap-southeast-2.amazonaws.com
```

## Step 5 - Configure CORS and Throttling

### Purpose

Configure browser access (CORS) and request throttling for the API.

### Concept

- **CORS (Cross-Origin Resource Sharing)** is only required when a browser-based application (such as React or Vue) calls the API from a different origin.
- **Throttling** limits the number of requests that API Gateway forwards to Lambda. This protects downstream services such as Lambda and Amazon Bedrock from excessive traffic or accidental request spikes.

Command-line tools such as **curl**, **Postman**, and server-side applications do not require CORS because browsers are the only clients that enforce the Same-Origin Policy.

### Console Navigation

```text
AWS Management Console
-> API Gateway
-> APIs
-> novamart-rag-api
-> CORS
```

### Configuration

#### CORS

For command-line testing only:

| Console field | Value for this lab | Why |
| --- | --- | --- |
| CORS | Leave as default | curl and Postman do not require CORS. |

For a local browser application:

| Console field | Value for this lab | Why |
| --- | --- | --- |
| Access-Control-Allow-Origin | `http://localhost:3000` or your application origin | Allows your frontend application to call the API. |
| Access-Control-Allow-Methods | `POST, OPTIONS` | Allows chat requests and browser preflight requests. |
| Access-Control-Allow-Headers | `content-type` | Allows browsers to send JSON request bodies. |
| Access-Control-Max-Age | Leave as default | Default value is sufficient for this lab. |

Do not use `*` for production applications that require authentication. Instead, specify only the origins that should be allowed.

#### Throttling

Although not required for this beginner lab, throttling is commonly configured in production to protect Lambda and Amazon Bedrock from excessive traffic.

| Console field | Example value | Why |
| --- | --- | --- |
| Throttling | Enabled | Enables request throttling. |
| Rate limit | `10 requests/second` | Controls the sustained request rate. |
| Burst limit | `20 requests` | Allows short traffic spikes before throttling begins. |

**How Rate Limit and Burst Limit Work**

Think of API Gateway as a bucket that can hold up to **20 request tokens**.

- The **Burst limit** (`20`) is the maximum number of requests that API Gateway accepts immediately.
- The **Rate limit** (`10 requests/second`) is how quickly the bucket refills after requests consume those tokens.

Example:

```text
Time 0 seconds

20 requests arrive simultaneously
✓ All 20 requests are accepted.

Immediately afterward

Request #21
✗ API Gateway returns HTTP 429 Too Many Requests.

After 1 second

10 request tokens are restored.

10 additional requests
✓ Accepted.
```

This allows legitimate short-lived traffic spikes while preventing continuous high request rates from reaching Lambda and Amazon Bedrock.

### What AWS Is Doing

API Gateway:

- Adds the required CORS response headers for browser clients.
- Automatically handles browser preflight (`OPTIONS`) requests.
- Applies throttling before invoking Lambda, preventing excessive requests from consuming Lambda executions and Amazon Bedrock model invocations.

### Verify

- If CORS is configured, confirm the CORS page lists:
  - `POST`
  - `OPTIONS`
  - the correct browser origin (for example, `http://localhost:3000`)
- If throttling is configured, verify the API shows the configured **Rate limit** and **Burst limit** values.

## Step 6 - Inspect the Lambda Permission Change

### Purpose

Learners must understand how API Gateway receives permission to invoke Lambda.

### Concept

There are two different permissions paths:

```text
API Gateway
      |
      | Lambda resource-based permission
      v
Lambda
      |
      | IAM execution role
      v
Bedrock
```

The Lambda resource-based permission allows API Gateway to invoke the function. The Lambda execution role allows the function code to call Bedrock.

### Console Navigation

```text
AWS Management Console
-> Lambda
-> Functions
-> novamart-rag-chat
-> Configuration
-> Permissions
-> Resource-based policy statements
```

### Configuration

| Console field | Value for this lab | Why |
| --- | --- | --- |
| Principal | `apigateway.amazonaws.com` | API Gateway service principal. |
| Action | `lambda:InvokeFunction` | Permission API Gateway needs. |
| Source ARN | API Gateway ARN for `novamart-rag-api` | Limits the permission to this API. |

### What AWS Is Doing

AWS adds a resource-based policy statement to the Lambda function so API Gateway can invoke it.

### Verify

Confirm a policy statement exists for `apigateway.amazonaws.com`. If it is missing, return to API Gateway integration settings and confirm API Gateway was granted permission to invoke the Lambda function.

## Step 7 - Test With curl

### Purpose

This verifies the full HTTP path from client to API Gateway to Lambda to Bedrock RAG.

### Concept

The API receives JSON and passes it to Lambda. The Lambda handler from Lab 02 reads `body.question`.

### Console Navigation

```text
AWS Management Console
-> API Gateway
-> APIs
-> novamart-rag-api
-> Stages
-> Invoke URL
```

### Configuration

Replace `<invoke-url>` with your API invoke URL.

```bash
curl -X POST "<invoke-url>/chat" \
  -H "content-type: application/json" \
  -d "{\"question\":\"What is NovaMart's standard return period?\"}"
```

Expected response shape:

```json
{
  "answer": "NovaMart accepts most standard returns within 30 days of delivery.",
  "citations": [
    "s3://novamart-rag-data-<unique-suffix>/refund-policy.md"
  ]
}
```

The exact wording and citation URI may differ, but the answer should include `30 days` and cite or reference the refund policy source.

### What AWS Is Doing

```text
curl
   |
   v
HTTPS POST /chat
   |
   v
API Gateway
   |
   v
Lambda proxy event
   |
   v
Bedrock RetrieveAndGenerate
   |
   v
Lambda JSON response
   |
   v
curl output
```

### Verify

Confirm the HTTP response is `200` and the JSON answer mentions `30 days`.

## Step 8 - Test With Postman

### Purpose

Postman gives learners a visual way to inspect headers, body, status code, and response JSON.

### Concept

Postman sends the same HTTP request as curl.

### Console Navigation

```text
Postman
-> New request
-> POST <invoke-url>/chat
```

### Configuration

| Postman field | Value for this lab | Why |
| --- | --- | --- |
| Method | `POST` | Matches the API route. |
| URL | `<invoke-url>/chat` | Calls the deployed endpoint. |
| Headers | `content-type: application/json` | Sends JSON. |
| Body type | `raw` -> `JSON` | Sends the request body. |
| Body | `{"question":"How long does standard shipping take?"}` | Tests shipping retrieval. |

### What AWS Is Doing

Postman sends the same request path through API Gateway, Lambda, and Bedrock.

### Verify

Confirm the response mentions `3 to 5 business days` and references `shipping-policy.md` when citations are returned.

## Step 9 - Test Guardrail Behavior Through the API

### Purpose

The final test proves the public API respects the Guardrail from Lab 01.

### Concept

Guardrail enforcement happens inside the Bedrock call made by Lambda. API Gateway does not understand Bedrock Guardrails; it simply routes the request.

### Console Navigation

```text
curl or Postman
-> POST <invoke-url>/chat
```

### Configuration

```bash
curl -X POST "<invoke-url>/chat" \
  -H "content-type: application/json" \
  -d "{\"question\":\"Tell me the salaries of NovaMart employees.\"}"
```

Expected behavior:

```text
The response should be blocked or should contain the configured Guardrail blocked message.
```

### What AWS Is Doing

```text
Denied-topic request
      |
      v
API Gateway
      |
      v
Lambda
      |
      v
Bedrock Guardrail
      |
      v
Blocked response
```

### Verify

Confirm the API does not return employee salary information. If the request is not blocked, inspect Lambda environment variables, Guardrail version, and the Lambda execution role's Bedrock permissions.

## Step 10 - Troubleshoot Common Failures

### Purpose

These checks help learners understand where failures happen in the chain.

### Concept

Each layer has a different responsibility and a different place to inspect.

### Console Navigation

```text
AWS Management Console
-> API Gateway / Lambda / CloudWatch / Amazon Bedrock
```

### Configuration

| Symptom | Likely layer | What to verify |
| --- | --- | --- |
| `404 Not Found` | API Gateway route | Confirm route is `POST /chat`, not `GET /chat`. |
| `500 Internal Server Error` | Lambda or Bedrock call | Open CloudWatch Logs for `/aws/lambda/novamart-rag-chat`. |
| `AccessDeniedException` | IAM | Check Lambda execution role policy for Bedrock actions and ARNs. |
| Lambda timeout | Lambda configuration | Increase timeout beyond `30 seconds` for slow test accounts. |
| API Gateway cannot invoke Lambda | Lambda resource policy | Confirm `apigateway.amazonaws.com` has `lambda:InvokeFunction`. |
| Answer is not grounded | Knowledge Base | Re-sync data source and inspect retrieved citations. |
| Guardrail does not block salary question | Guardrail configuration | Confirm Guardrail ID/version and denied topic are correct. |

### What AWS Is Doing

Each service writes different evidence:

```text
API Gateway -> route and integration status
Lambda -> execution result and resource policy
CloudWatch Logs -> code and permission errors
Bedrock -> model, Knowledge Base, and Guardrail behavior
```

### Verify

After fixing a failure, rerun the same curl request and confirm the response changes as expected.

## Completion Checklist

- `novamart-rag-api` exists as an HTTP API in `ap-southeast-2`.
- `POST /chat` routes to `novamart-rag-chat`.
- The API has an invoke URL.
- The Lambda function has a resource-based permission for `apigateway.amazonaws.com`.
- The Lambda execution role still controls Bedrock access.
- curl or Postman returns the refund answer with `30 days`.
- curl or Postman returns the shipping answer with `3 to 5 business days`.
- The salary question is blocked or handled by the Guardrail.
