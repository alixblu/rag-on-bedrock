# Lab 01 - Amazon Bedrock RAG With Knowledge Bases and Guardrails

Region: `ap-southeast-2` - Asia Pacific (Sydney)

Primary model: `Amazon Nova Lite` (`amazon.nova-lite-v1:0`)

Fallback model: `Amazon Nova Micro` (`amazon.nova-micro-v1:0`)

Embeddings model: `Amazon Titan Text Embeddings V2` (`amazon.titan-embed-text-v2:0`)

Dataset: `data/fictional-company/`

This lab builds a single retrieval augmented generation workflow in Amazon Bedrock. You will test a foundation model, upload fictional NovaMart documents to Amazon S3, create a Bedrock Knowledge Base, test retrieval and generation, create a Bedrock Guardrail, and connect that Guardrail to the RAG workflow.

AWS periodically updates the Management Console. If a label differs slightly, follow the concept in the step and verify against the official AWS documentation listed in `docs/aws-console-verification.md`.

## Architecture

```text
User question
     |
     v
Bedrock Guardrail
     |
     v
Bedrock Knowledge Base
     |
     v
Amazon S3 documents -> Parsing -> Chunking -> Titan embeddings -> S3 Vectors
     |
     v
Amazon Nova Lite
     |
     v
Bedrock Guardrail
     |
     v
Grounded answer with citations
```

## Before You Start

Use an IAM identity with permission to use Amazon Bedrock, Amazon S3, IAM role creation for Bedrock service roles, and Bedrock Guardrails. Do not use the root user for this lab.

Create a unique suffix for resources. Example: your initials plus date, such as `pa-20260805`.

Use these names throughout the lab:

| Resource | Value |
| --- | --- |
| S3 bucket | `novamart-rag-data-<unique-suffix>` |
| Knowledge Base | `novamart-kb` |
| Data source | `novamart-policy-docs` |
| Guardrail | `novamart-rag-guardrail` |
| Region | `ap-southeast-2` |

## Part A - Foundation Model

### Step 1 - Select the AWS Region

#### Purpose

The Region controls where Bedrock models, S3 data, vector indexes, guardrails, logs, and API calls are created. Keeping every resource in `ap-southeast-2` avoids cross-Region confusion and keeps the lab reproducible.

#### Concept

Amazon Bedrock model availability is Region-specific. A model that appears in one Region may not appear in another learner's console.

#### Console Navigation

```text
AWS Management Console
-> Region selector
-> Asia Pacific (Sydney) ap-southeast-2
```

#### What AWS Is Doing

The console scope changes to Sydney. Resources you create from this point forward are created in `ap-southeast-2` unless a service is global.

#### Verify

Check the upper-right Region selector. It should show `Sydney` or `ap-southeast-2`.

### Step 2 - Open Amazon Bedrock

#### Purpose

Bedrock provides the foundation model, Knowledge Base, and Guardrail features used in this lab.

#### Concept

Amazon Bedrock is a managed service for foundation models and generative AI application features. In this lab, Bedrock is both the model runtime and the RAG orchestration layer.

#### Console Navigation

```text
AWS Management Console
-> Amazon Bedrock
```

#### Configuration

| Console field | Value for this lab | Why |
| --- | --- | --- |
| Service search | `Amazon Bedrock` | Opens the Bedrock console. |
| Region | `ap-southeast-2` | Must match the lab Region. |

#### What AWS Is Doing

The Bedrock console shows the resources and model catalog available to your IAM identity in Sydney.

#### Verify

The page title should show Amazon Bedrock, and the Region selector should still be Sydney.

### Step 3 - Find Available Foundation Models

#### Purpose

You need to confirm which models can be used before building the RAG workflow.

#### Concept

AWS documentation describes current model access as a mix of default Amazon model access, Marketplace-backed access for some providers, and provider-specific setup for some models. Do not assume every account follows the old universal "Request model access" path.

#### Console Navigation

```text
AWS Management Console
-> Amazon Bedrock
-> Foundation models
-> Model catalog
```

#### Configuration

| Console field | Value for this lab | Why |
| --- | --- | --- |
| Provider filter | `Amazon` | Nova and Titan are Amazon models used in this lab. |
| Modality or task filter | Text or chat, if shown | Narrows the model list to models usable for prompts. |
| Region | `ap-southeast-2` | Confirms Sydney availability. |
| Model | `Amazon Nova Lite` | Primary chat/text generation model for the lab. |
| Fallback model | `Amazon Nova Micro` | Simpler fallback if Nova Lite is unavailable in the learner account. |

#### What AWS Is Doing

The console is listing foundation models that can be discovered in the selected Region. Model access can depend on model provider, Marketplace subscription permissions, and account setup.

#### Verify

Open `Amazon Nova Lite` and confirm it is available or active in `ap-southeast-2`. If the console does not allow Nova Lite, use `Amazon Nova Micro` for every generation step in this lab and record that substitution.

### Step 4 - Open the Bedrock Playground

#### Purpose

The playground proves that the selected foundation model can answer a basic prompt before you connect it to retrieval.

#### Concept

The playground is a direct model test. It does not use your private S3 documents or Knowledge Base yet.

#### Console Navigation

```text
AWS Management Console
-> Amazon Bedrock
-> Playgrounds
-> Chat/Text playground
```

Use the playground label shown in your console. AWS has changed playground names over time, so the important concept is the text or chat playground for foundation model inference.

#### Configuration

| Console field | Value for this lab | Why |
| --- | --- | --- |
| Model provider | `Amazon` | Selects Amazon Nova models. |
| Model | `Amazon Nova Lite` | Primary model for lab generation. |
| Prompt | `Write a two-sentence customer support answer explaining what a refund policy is.` | Simple non-RAG prompt to prove inference works. |
| Temperature | `0.2` | Keeps the answer focused and repeatable. |
| Top P | `Leave as default` | Default is acceptable for this basic test. |
| Maximum length / max tokens | `300` or nearest console equivalent | Gives enough room for a two-sentence answer. |
| Stop sequences | `Leave empty - not required for this lab.` | No custom stopping rule is needed. |

#### What AWS Is Doing

```text
Prompt
   |
   v
Amazon Nova Lite
   |
   v
Generated response
```

Bedrock sends your prompt and inference parameters to the selected model and returns generated text.

#### Verify

Choose the console action that runs the prompt, such as `Run`, `Submit`, or the send button. Confirm the model returns a coherent two-sentence answer. If you receive an access or availability error, return to the model catalog and choose the fallback model.

## Part B - Knowledge Base

### Step 5 - Create an S3 Bucket

#### Purpose

The Knowledge Base needs a document source. Amazon S3 stores the NovaMart markdown files that Bedrock will parse, chunk, embed, and index.

#### Concept

S3 bucket names are globally unique across AWS. The unique suffix prevents a naming conflict with another AWS customer.

#### Console Navigation

```text
AWS Management Console
-> S3
-> Buckets
-> Create bucket
```

#### Configuration

| Console field | Value for this lab | Why |
| --- | --- | --- |
| Bucket type | `General purpose` | Standard S3 bucket type for documents. |
| Bucket name | `novamart-rag-data-<unique-suffix>` | Must be globally unique. |
| AWS Region | `Asia Pacific (Sydney) ap-southeast-2` | Keeps the data in the same Region as Bedrock. |
| Object Ownership | `ACLs disabled` / bucket owner enforced | Recommended default for modern S3 ownership. |
| Block Public Access | `Leave as default` with all public access blocked | The documents are private lab data. |
| Bucket Versioning | `Leave as default` | Versioning is not required for this first lab. |
| Tags | `Leave empty - not required for this lab.` | Tags are useful in production but not needed here. |
| Default encryption | `Leave as default` | S3 server-side encryption defaults are sufficient for the lab. |
| Advanced settings | `Leave as default` | No object lock or advanced feature is required. |

#### What AWS Is Doing

AWS creates a private S3 bucket in Sydney. Bedrock will later receive permission through a service role to read objects from this bucket.

#### Verify

Open the bucket list and confirm `novamart-rag-data-<unique-suffix>` appears with Region `ap-southeast-2`.

### Step 6 - Upload the NovaMart Documents

#### Purpose

The Knowledge Base can only answer from documents that are present in the S3 data source.

#### Concept

These markdown files are the approved support corpus. The lab intentionally excludes the CEO and employee salary information so learners can observe grounded and unsupported answers.

#### Console Navigation

```text
AWS Management Console
-> S3
-> Buckets
-> novamart-rag-data-<unique-suffix>
-> Upload
```

#### Configuration

| Console field | Value for this lab | Why |
| --- | --- | --- |
| Files | Upload all files from `data/fictional-company/` | Provides the retrieval corpus. |
| Destination | Bucket root or `fictional-company/` prefix | Either works. Use one path consistently later. |
| Permissions | `Leave as default` | Objects stay private to the bucket owner. |
| Properties | `Leave as default` | Storage class and encryption defaults are fine. |

Upload these files:

- `company-overview.md`
- `refund-policy.md`
- `shipping-policy.md`
- `product-faq.md`
- `support-policy.md`

#### What AWS Is Doing

```text
Local markdown files
   |
   v
Private S3 objects
   |
   v
Future Bedrock data source
```

AWS stores each markdown file as an S3 object.

#### Verify

Open the bucket or prefix and confirm all five files are visible. Copy the S3 URI for the bucket or prefix. Example: `s3://novamart-rag-data-<unique-suffix>/fictional-company/`.

### Step 7 - Create a Knowledge Base

#### Purpose

The Knowledge Base indexes the NovaMart documents so Bedrock can retrieve relevant chunks for questions.

#### Concept

A Bedrock Knowledge Base connects a data source, parser, chunking strategy, embeddings model, vector store, and service role.

For this lab, create an **Unstructured** Knowledge Base using the **Unmanaged** option.

- **Unstructured** is designed for document-based content such as Markdown, PDF, HTML, Microsoft Word, and plain text files. Since the NovaMart knowledge base consists of Markdown documents stored in Amazon S3, Bedrock must parse, chunk, and embed these documents before they can be searched.

- **Structured** is intended for structured data sources such as relational databases or tables with predefined schemas. This lab does not retrieve information from structured records, so this option is not appropriate.

- **Unmanaged** gives you full control over the retrieval pipeline. In the following steps, you will configure the parser, chunking strategy, embeddings model, and vector store yourself. This helps you understand how each component of a Bedrock RAG pipeline works.

- **Managed** is better suited for production environments where AWS automatically configures much of the retrieval pipeline. While it simplifies deployment, it hides many of the components that this lab is designed to teach.

#### Console Navigation

```text
AWS Management Console
-> Amazon Bedrock
-> Knowledge bases
-> Create
-> Knowledge base with vector store
```

#### Configuration

| Console field | Value for this lab | Why |
| --- | --- | --- |
| Knowledge Base type | `Unstructured` | The data source is a collection of Markdown documents stored in Amazon S3. |
| Management option | `Unmanaged` | Allows you to configure each stage of the RAG pipeline in the following steps. |
| Knowledge Base name | `novamart-kb` | Identifies the RAG index. |
| Description | `NovaMart fictional customer support knowledge base` | Clarifies the purpose. |
| IAM permissions / service role | Create and use a new service role | Best first-lab path because AWS wires required service permissions. |
| Service role name | `Leave as default` or use AWS generated name | AWS-generated role names include the service purpose. |
| Tags | `Leave empty - not required for this lab.` | Tags are not needed for the learning objective. |

#### What AWS Is Doing

```text
Bedrock Knowledge Base
       |
       | assumes
       v
Bedrock service role
       |
       +-- Read S3 documents
       +-- Invoke embeddings model
       +-- Write vectors to vector store
```

AWS prepares a Bedrock-managed RAG resource and a service role that Bedrock can assume.

#### Verify

Before continuing, confirm the wizard shows `novamart-kb` as the Knowledge Base name and the Region remains `ap-southeast-2`.

### Step 8 - Configure the Data Source

#### Purpose

The data source tells Bedrock where the NovaMart documents live.

#### Concept

The S3 URI is the bridge from Bedrock to your private document set.

#### Console Navigation

```text
AWS Management Console
-> Amazon Bedrock
-> Knowledge bases
-> Create
-> Data source configuration
```

#### Configuration

| Console field | Value for this lab | Why |
| --- | --- | --- |
| Data source type | `Amazon S3` | The lab documents are in S3. |
| Data source name | `novamart-policy-docs` | Names the source inside the Knowledge Base. |
| S3 URI | `s3://novamart-rag-data-<unique-suffix>/` or chosen prefix | Points to the uploaded markdown files. |
| Inclusion prefixes | `Leave empty` if the S3 URI points only to the lab files | Not needed if the bucket or prefix contains only lab data. |
| Exclusion patterns | `Leave empty - not required for this lab.` | No files need to be excluded. |
| Data deletion policy | `Retain` if shown | Keeps source files in S3 if the data source is removed. |

#### What AWS Is Doing

Bedrock records the S3 location and prepares to crawl the documents during synchronization. It does not ingest the files until you start sync.

#### Verify

Confirm the S3 URI matches the bucket or prefix where the five markdown files were uploaded.

### Step 9 - Configure Parsing and Chunking

#### Purpose

Parsing converts files into text. Chunking splits that text into pieces that can be embedded and retrieved.

#### Concept

For `ap-southeast-2`, use the default parser as the main path. Do not use Bedrock Data Automation parser as the main lab path because AWS documentation lists that parser path as limited and preview.

#### Console Navigation

```text
AWS Management Console
-> Amazon Bedrock
-> Knowledge bases
-> Create
-> Parsing and chunking
```

#### Configuration

| Console field | Value for this lab | Why |
| --- | --- | --- |
| Parsing strategy | `Default` | Works for simple markdown support documents. |
| Advanced parsing | `Leave as default` | Not needed for this text-only lab. |
| Chunking strategy | `Default` or `Fixed-size chunking` if the wizard requires a choice | Simple policy docs do not need custom semantic chunking. |
| Max tokens | `Leave as default` | Default chunk size is sufficient for short documents. |
| Overlap percentage | `Leave as default` | Default overlap preserves context without over-tuning. |
| Custom transformations | `Leave empty - not required for this lab.` | No transformation Lambda is needed. |

#### What AWS Is Doing

```text
Markdown
   |
   v
Parser
   |
   v
Chunks
```

Bedrock will convert each markdown file into text chunks during ingestion.

#### Verify

Confirm the parser is the default parser and no advanced parser is selected.

### Step 10 - Configure Embeddings and Vector Store

#### Purpose

Embeddings turn text chunks into numeric vectors. The vector store saves those vectors so Bedrock can search for semantically relevant content.

#### Concept

The embeddings model and vector store are the retrieval engine behind the Knowledge Base.

#### Console Navigation

```text
AWS Management Console
-> Amazon Bedrock
-> Knowledge bases
-> Create
-> Embeddings model and vector store
```

#### Configuration

| Console field | Value for this lab | Why |
| --- | --- | --- |
| Embeddings model provider | `Amazon` | Uses Amazon Titan embeddings. |
| Embeddings model | `Amazon Titan Text Embeddings V2` | Current general-purpose text embeddings choice. |
| Vector store | `Quick create a new vector store` | The goal is learning Bedrock RAG, not manually administering a database. |
| Vector store type | `Amazon S3 Vectors` if shown | Simple AWS-managed vector storage path for the lab. |
| Advanced vector settings | `Leave as default` | No custom index tuning is needed. |

If the console shows these choices:

```text
Vector store:

O Quick create a new vector store
O Choose an existing vector store
```

Select `Quick create a new vector store`. For this learning lab, quick create is selected because the goal is to learn Bedrock Knowledge Bases rather than manually administer the vector database.

#### What AWS Is Doing

```text
Chunks
   |
   v
Amazon Titan Text Embeddings V2
   |
   v
Vectors
   |
   v
Amazon S3 Vectors
```

AWS will create or configure vector storage and connect it to the Knowledge Base.

#### Verify

Review the summary page before creating the Knowledge Base. Confirm the model, vector store option, S3 data source, parser, and Region.

### Step 11 - Create and Synchronize the Knowledge Base

#### Purpose

Creation saves the Knowledge Base configuration. Synchronization ingests the S3 documents into the vector store.

#### Concept

A Knowledge Base is not useful until the data source has been synchronized at least once.

#### Console Navigation

```text
AWS Management Console
-> Amazon Bedrock
-> Knowledge bases
-> novamart-kb
-> Data sources
-> novamart-policy-docs
-> Sync / Start ingestion
```

Use the button label shown by the current console, such as `Sync`, `Start sync`, or `Start ingestion job`.

#### Configuration

| Console field | Value for this lab | Why |
| --- | --- | --- |
| Review | Confirm all values | Prevents creating the wrong resource in the wrong Region. |
| Create action | `Create knowledge base` | Saves the RAG configuration. |
| Data source action | `Sync` / `Start ingestion` | Starts parsing, chunking, embedding, and indexing. |

#### What AWS Is Doing

```text
Markdown
   |
   v
Parsing
   |
   v
Chunks
   |
   v
Embedding model
   |
   v
Vectors
   |
   v
Vector store
```

Bedrock reads the S3 objects, parses the markdown, creates chunks, calls Titan embeddings, and stores vectors in the vector store.

#### Verify

Wait for the ingestion job status to show `Complete` or the current console's success status. Check the file or document count if shown. If ingestion fails, open the error details and verify the S3 URI, service role permissions, and model availability.

## Part C - RAG Testing

### Step 12 - Open the Knowledge Base Test Interface

#### Purpose

The test interface lets you verify retrieval before writing any application code.

#### Concept

Retrieval finds relevant chunks. Retrieve and generate uses those chunks as context for a foundation model response.

#### Console Navigation

```text
AWS Management Console
-> Amazon Bedrock
-> Knowledge bases
-> novamart-kb
-> Test knowledge base
```

#### Configuration

| Console field | Value for this lab | Why |
| --- | --- | --- |
| Knowledge Base | `novamart-kb` | Tests the lab RAG index. |
| Generation model | `Amazon Nova Lite` | Generates grounded answers from retrieved chunks. |
| Number of results | `Leave as default` | Defaults are sufficient for five short files. |
| Search type | `Leave as default` | Start with AWS default retrieval behavior. |
| Prompt template | `Leave as default` | Custom prompting is not needed yet. |

#### What AWS Is Doing

```text
Question
   |
   v
Vector search
   |
   v
Relevant chunks
   |
   v
Optional model generation
```

#### Verify

The test panel should let you enter a question and view retrieved sources, generated answers, or both.

### Step 13 - Test a Grounded Refund Answer

#### Purpose

This proves the Knowledge Base can retrieve from `refund-policy.md`.

#### Concept

A grounded answer should cite or display the source document that contains the claim.

#### Console Navigation

```text
AWS Management Console
-> Amazon Bedrock
-> Knowledge bases
-> novamart-kb
-> Test knowledge base
```

#### Configuration

| Console field | Value for this lab | Why |
| --- | --- | --- |
| Mode | `Retrieve and Generate` or generation-enabled query | Tests full RAG. |
| Question | `What is NovaMart's standard return period?` | Expected answer is in `refund-policy.md`. |
| Generation model | `Amazon Nova Lite` | Uses the lab model. |

Expected answer:

```text
NovaMart accepts most standard returns within 30 days of delivery.
```

Expected source:

```text
refund-policy.md
```

#### What AWS Is Doing

Bedrock retrieves chunks related to returns and asks Nova Lite to answer using those chunks.

#### Verify

Inspect the retrieved sources or citations. Confirm `refund-policy.md` is shown and the answer says `30 days`.

### Step 14 - Test a Grounded Shipping Answer

#### Purpose

This proves the Knowledge Base can retrieve a different policy document.

#### Concept

RAG should select context based on the question, not simply summarize the whole corpus.

#### Console Navigation

```text
AWS Management Console
-> Amazon Bedrock
-> Knowledge bases
-> novamart-kb
-> Test knowledge base
```

#### Configuration

| Console field | Value for this lab | Why |
| --- | --- | --- |
| Mode | `Retrieve and Generate` or generation-enabled query | Tests answer generation with citations. |
| Question | `How long does standard shipping take?` | Expected answer is in `shipping-policy.md`. |
| Generation model | `Amazon Nova Lite` | Uses the lab model. |

Expected answer:

```text
Standard shipping usually takes 3 to 5 business days after the order leaves the warehouse.
```

Expected source:

```text
shipping-policy.md
```

#### What AWS Is Doing

Bedrock searches for shipping-related chunks and uses them to generate a concise answer.

#### Verify

Confirm the answer includes `3 to 5 business days` and cites or retrieves `shipping-policy.md`.

### Step 15 - Compare Retrieve and Retrieve and Generate

#### Purpose

Learners need to understand what retrieval does before a model writes the final response.

#### Concept

`Retrieve` returns matching chunks from the Knowledge Base. `Retrieve and Generate` retrieves chunks and then asks a foundation model to write an answer from those chunks.

#### Console Navigation

```text
AWS Management Console
-> Amazon Bedrock
-> Knowledge bases
-> novamart-kb
-> Test knowledge base
-> Configurations
```

#### Configuration

| Console field | Value for this lab | Why |
| --- | --- | --- |
| Query mode | `Retrieve` | Shows raw matching chunks. |
| Query mode | `Retrieve and Generate` | Shows generated answer plus sources. |
| Question | Reuse the refund and shipping questions | Keeps the comparison clear. |

#### What AWS Is Doing

```text
Retrieve:
Question -> Vector search -> Chunks

Retrieve and Generate:
Question -> Vector search -> Chunks -> Foundation model -> Answer
```

#### Verify

Run the same question in both modes if your console exposes both. Confirm `Retrieve` shows source chunks, while `Retrieve and Generate` writes a natural-language answer and shows citations or source references.

### Step 16 - Test an Unsupported Question

#### Purpose

This teaches grounding and hallucination risk.

#### Concept

The dataset intentionally does not include NovaMart's CEO. A grounded system should not invent a CEO.

#### Console Navigation

```text
AWS Management Console
-> Amazon Bedrock
-> Knowledge bases
-> novamart-kb
-> Test knowledge base
```

#### Configuration

| Console field | Value for this lab | Why |
| --- | --- | --- |
| Mode | `Retrieve and Generate` | Tests whether the model stays grounded. |
| Question | `Who is NovaMart's CEO?` | This information is absent from the corpus. |
| Generation model | `Amazon Nova Lite` | Uses the lab model. |

Expected behavior:

```text
The answer should say the information is not available in the knowledge base, or it should avoid naming a CEO.
```

#### What AWS Is Doing

Bedrock may retrieve general company-overview chunks, but those chunks do not include a CEO. The model should not fabricate unsupported details.

#### Verify

Confirm no CEO name is invented. If a name appears, explain that this is a hallucination risk and that production systems need stronger prompting, guardrails, evaluation, and monitoring.

## Part D - Bedrock Guardrails

### Step 17 - Create a Guardrail

#### Purpose

The Guardrail adds policy controls around user input and model output.

#### Concept

Guardrails can filter unsafe content, denied topics, blocked words, sensitive information, and other policy categories. In this lab, the key denied topic is internal employee compensation information.

#### Console Navigation

```text
AWS Management Console
-> Amazon Bedrock
-> Guardrails
-> Create guardrail
```

#### Configuration

| Console field | Value for this lab | Why |
| --- | --- | --- |
| Guardrail name | `novamart-rag-guardrail` | Identifies the lab safety policy. |
| Description | `Blocks internal compensation requests and handles sensitive information for the NovaMart RAG lab.` | Explains the guardrail purpose. |
| Messaging for blocked prompt | `I cannot help with that request. I can help with NovaMart customer support policy questions.` | Learner-friendly blocked response. |
| Messaging for blocked response | `I cannot provide that information. Please ask a customer support policy question.` | Clear fallback response. |

#### What AWS Is Doing

AWS creates a draft Guardrail configuration that can evaluate prompts and model responses.

#### Verify

Confirm the Guardrail name appears and the console shows an editable `DRAFT` configuration.

### Step 18 - Configure Content Filters

#### Purpose

Content filters provide baseline safety handling for harmful content categories.

#### Concept

Content filters are general safety controls. They are different from business-specific denied topics.

#### Console Navigation

```text
AWS Management Console
-> Amazon Bedrock
-> Guardrails
-> novamart-rag-guardrail
-> Content filters
```

#### Configuration

| Console field | Value for this lab | Why |
| --- | --- | --- |
| Hate | `Leave as default` | Not the main teaching focus. |
| Insults | `Leave as default` | Default handling is acceptable for the lab. |
| Sexual | `Leave as default` | Not the main teaching focus. |
| Violence | `Leave as default` | Not the main teaching focus. |
| Misconduct | `Leave as default` | Default handling is acceptable. |
| Prompt attack | `Leave as default` or enable if shown | Keep current AWS recommended behavior. |

#### What AWS Is Doing

Bedrock configures category-level checks that can apply to user prompts and model outputs.

#### Verify

Confirm the content filter section saves without errors.

### Step 19 - Configure a Denied Topic

#### Purpose

Denied topics block a business-specific topic even if the prompt is otherwise ordinary.

#### Concept

Employee salaries are not customer support information. The dataset also states support agents must not disclose employee compensation information.

#### Console Navigation

```text
AWS Management Console
-> Amazon Bedrock
-> Guardrails
-> novamart-rag-guardrail
-> Denied topics
-> Add denied topic
```

#### Configuration

| Console field | Value for this lab | Why |
| --- | --- | --- |
| Topic name | `Internal employee compensation information` | Required denied topic. |
| Definition | `Requests for salaries, wages, bonuses, payroll, compensation bands, or employee pay details for NovaMart employees.` | Defines what to block. |
| Sample phrase | `Tell me the salaries of NovaMart employees.` | Test prompt for the denied topic. |
| Sample phrase | `What are NovaMart's internal pay bands?` | Clarifies the topic boundary. |
| Action | `Block` | The lab should refuse this topic. |

#### What AWS Is Doing

Bedrock adds a business-policy classifier to the Guardrail.

#### Verify

Confirm the denied topic is listed with the block action.

### Step 20 - Configure Word Filters

#### Purpose

Word filters let you block exact words or phrases that should never appear in prompts or responses.

#### Concept

This lab focuses on denied topics and sensitive information, so exact word blocking is not required.

#### Console Navigation

```text
AWS Management Console
-> Amazon Bedrock
-> Guardrails
-> novamart-rag-guardrail
-> Word filters
```

#### Configuration

| Console field | Value for this lab | Why |
| --- | --- | --- |
| Managed word lists | `Leave as default` | Default handling is sufficient. |
| Custom words or phrases | `Leave empty - not required for this lab.` | No exact word list is needed. |

#### What AWS Is Doing

AWS leaves exact word filtering at the default configuration.

#### Verify

Confirm no custom word filter is required to continue.

### Step 21 - Configure Sensitive Information Filters

#### Purpose

Sensitive information filters demonstrate how a system can respond when users provide personal data.

#### Concept

`BLOCK` rejects the input or output. `ANONYMIZE` or `MASK` allows the interaction to continue while replacing sensitive values. Use only fictional test data in this lab.

#### Console Navigation

```text
AWS Management Console
-> Amazon Bedrock
-> Guardrails
-> novamart-rag-guardrail
-> Sensitive information filters
```

#### Configuration

| Console field | Value for this lab | Why |
| --- | --- | --- |
| Email address | `ANONYMIZE` / `MASK` if available | Demonstrates safe handling without failing the whole request. |
| Phone number | `ANONYMIZE` / `MASK` if available | Demonstrates redaction of fictional contact data. |
| Credit card number | `BLOCK` | Payment card data should not be processed in this support chat. |
| US Social Security Number or national ID equivalent | `BLOCK` | Highly sensitive identifiers are not needed. |
| Custom regex | `Leave empty - not required for this lab.` | Managed PII types are enough. |

Use obviously fictional test data only. Do not paste real personal, payment, or identity information.

#### What AWS Is Doing

Bedrock configures detectors for selected sensitive data types and applies the selected action.

#### Verify

Confirm the sensitive information policy saves and that the console shows the configured action for each selected data type.

### Step 22 - Save and Create a Guardrail Version

#### Purpose

Applications should call a stable guardrail version instead of an editable draft.

#### Concept

`DRAFT` is useful for testing. A numbered version, such as `1`, freezes a usable configuration for SDK and API calls.

#### Console Navigation

```text
AWS Management Console
-> Amazon Bedrock
-> Guardrails
-> novamart-rag-guardrail
-> Save
-> Create version
```

#### Configuration

| Console field | Value for this lab | Why |
| --- | --- | --- |
| Save action | Save draft changes | Persists the guardrail policy. |
| Version description | `Initial NovaMart RAG lab guardrail` | Documents what version `1` means. |
| Version | `1` | Used by SDK/API examples. |

#### What AWS Is Doing

AWS stores the draft configuration and creates an immutable numbered version for application use.

#### Verify

Copy the Guardrail ID and version number. You will use them in Lab 02.

### Step 23 - Test the Guardrail

#### Purpose

Testing confirms the Guardrail allows normal support questions and blocks the denied topic.

#### Concept

A Guardrail can evaluate both user prompts and model responses. The test panel shows whether a rule matched.

#### Console Navigation

```text
AWS Management Console
-> Amazon Bedrock
-> Guardrails
-> novamart-rag-guardrail
-> Test
```

#### Configuration

| Test | Prompt | Expected result |
| --- | --- | --- |
| Normal request | `What is NovaMart's refund policy?` | Allowed. |
| Denied topic | `Tell me the salaries of NovaMart employees.` | Blocked by denied topic. |
| Sensitive information | `My fictional email is alex@example.test and my fictional card number is 4111111111111111.` | Email masked or anonymized if configured; card number blocked. |

#### What AWS Is Doing

```text
Prompt
   |
   v
Guardrail policy evaluation
   |
   +-- Allowed
   +-- Blocked
   +-- Masked or anonymized
```

#### Verify

Confirm the normal refund question is allowed, the compensation question is blocked, and the fictional sensitive information behaves according to your selected PII actions.

### Step 24 - Connect the Guardrail to RAG

#### Purpose

The lab must show how the Guardrail participates in the RAG workflow, not only how to create it.

#### Concept

The supported application pattern is to attach guardrail configuration to Bedrock runtime calls that generate the RAG answer. In the Bedrock console, the Knowledge Base test panel may expose Guardrails under `Configurations`. If your console exposes that section, select your Guardrail and version there for testing. If it does not, use the SDK/API pattern shown below.

#### Console Navigation

```text
AWS Management Console
-> Amazon Bedrock
-> Knowledge bases
-> novamart-kb
-> Test knowledge base
-> Configurations
-> Guardrails
```

If the console does not expose Guardrails in the Knowledge Base test configuration, do not invent a console step. Continue with the Lambda and SDK configuration in Lab 02, where the Guardrail is passed through the `RetrieveAndGenerate` API call.

#### Configuration

| Console field | Value for this lab | Why |
| --- | --- | --- |
| Guardrail | `novamart-rag-guardrail` | Applies the lab policy. |
| Guardrail version | `1` | Uses the stable version. |
| Generation model | `Amazon Nova Lite` | Keeps the RAG model consistent. |
| Prompt | `Tell me the salaries of NovaMart employees.` | Verifies the denied topic blocks the RAG request. |

#### What AWS Is Doing

```text
User question
     |
     v
Guardrail evaluation
     |
     v
Knowledge Base retrieval
     |
     v
Foundation Model
     |
     v
Guardrail evaluation
     |
     v
Grounded response
```

The Guardrail can evaluate unsafe or disallowed inputs before retrieval and evaluate generated outputs before returning them to the user.

#### Verify

Run the normal refund question and confirm it remains allowed. Run the salary question and confirm it is blocked. Then record the Knowledge Base ID, model ID, Guardrail ID, and Guardrail version for the Lambda function in Lab 02.

## Completion Checklist

- You selected `ap-southeast-2`.
- You tested `Amazon Nova Lite` or the documented fallback model.
- You created the S3 bucket and uploaded all five NovaMart markdown files.
- You created `novamart-kb`.
- You synchronized `novamart-policy-docs` successfully.
- You verified refund and shipping answers with citations.
- You verified the unsupported CEO question does not receive an invented answer.
- You created `novamart-rag-guardrail`.
- You tested allowed, denied-topic, and sensitive-information prompts.
- You connected the Guardrail to RAG through the console if available, or recorded the values needed for the Lambda `RetrieveAndGenerate` path in Lab 02.
