# NEUTRON
### Azure-native Enterprise Generative AI & Knowledge Intelligence Platform

**NEUTRON** is an enterprise AI knowledge assistant that connects an organization's existing information, turns documents and structured records into searchable knowledge, and answers questions through a secure, citation-grounded conversational interface. It is designed around **Microsoft Foundry, Azure OpenAI, Azure AI Search, Azure Functions, Azure Databricks, Python/FastAPI, and React/TypeScript**, with evaluation, observability, security, cost controls, and cloud operations built into the architecture.

> **Product goal:** Give employees reliable answers from the information they are authorized to use, and give administrators and AI engineers the tools to manage data, evaluate answer quality, understand failures, and control operating costs.

![NEUTRON AI Chat interface](docs/screens/01-ai-chat.svg)

**Interface previews:** [AI Chat](docs/screens/01-ai-chat.svg) · [Data Sources](docs/screens/02-data-sources.svg) · [Evaluation & Observability](docs/screens/03-evaluation-observability.svg)

---

## Table of contents

1. [Product overview](#product-overview)
2. [Users and core experiences](#users-and-core-experiences)
3. [Technology stack](#technology-stack)
4. [System architecture](#system-architecture)
5. [Data sources and connectors](#data-sources-and-connectors)
6. [Document ingestion and indexing](#document-ingestion-and-indexing)
7. [RAG and chat workflow](#rag-and-chat-workflow)
8. [Identity, privacy, and document security](#identity-privacy-and-document-security)
9. [AI evaluation and answer quality](#ai-evaluation-and-answer-quality)
10. [Observability, reliability, and cost optimization](#observability-reliability-and-cost-optimization)
11. [Application screens](#application-screens)
12. [Repository organization](#repository-organization)
13. [Developer prerequisites](#developer-prerequisites)
14. [Local development](#local-development)
15. [Azure deployment](#azure-deployment)
16. [API design](#api-design)
17. [Testing and release criteria](#testing-and-release-criteria)
18. [Project scope and boundaries](#project-scope-and-boundaries)
19. [Delivery milestones](#delivery-milestones)
20. [Microsoft documentation](#microsoft-documentation)

## Product overview

A typical organization stores its knowledge across PDF files, SharePoint libraries, Microsoft 365 documents, Azure Blob Storage, databases, and internal applications. Employees often cannot find the relevant information quickly, and a conventional LLM cannot safely answer questions about data it has never seen.

NEUTRON connects **approved organizational sources**, processes their content, creates a permission-aware search index, retrieves relevant passages, and uses a deployed Azure OpenAI model to produce an answer with verifiable source citations.

**Examples**

- An employee asks, “What is the annual leave policy?” and receives an answer citing the correct HR handbook section.
- A finance analyst asks, “What changed between Q2 and Q3 revenue?” and sees source passages from authorized financial reports.
- An administrator connects a SharePoint document library, chooses the approved scope, and schedules incremental synchronization.
- An AI engineer investigates an inaccurate answer by opening its request trace, inspecting retrieved chunks, and running the evaluation dataset.
- An operator compares models, chunk counts, retrieval settings, latency, and estimated inference cost before changing the production configuration.

### Design principles

- **Security before similarity:** no document is shown to a user—or passed to the LLM—unless access is authorized.
- **Citations over unsupported claims:** answers are grounded in accessible sources; insufficient evidence leads to an explicit no-answer response.
- **Observable from end to end:** the chat, retrieval, ingestion, and model calls share trace context.
- **Measurable quality:** a golden dataset and evaluation runs determine whether a change improves or damages output quality.
- **Cloud-native, but cost-aware:** long-lived chat services are separated from scheduled or event-driven processing jobs.
- **Focused scope:** one organization with multiple users; complex cross-organization multi-tenancy belongs in NEXUS.
- **Human-controlled operations:** connection authorization, sensitive documents, access changes, and releases are governed by people.

## Users and core experiences

| Role | Primary experience | Permissions |
| --- | --- | --- |
| **Employee** | Chat with approved knowledge, inspect citations, view own conversation history, submit answer feedback | Read/search only authorized information |
| **Knowledge Administrator** | Manage document collections, connect approved sources, run sync jobs, assign document groups | Manage sources and content within granted scope |
| **AI Engineer** | Review traces, retrieval metrics, evaluation runs, prompt/model configuration, and optimization experiments | Engineering diagnostics without blanket access to private content |
| **Platform Administrator** | Manage users, roles, service configuration, monitoring, retention, and security policy | Privileged administration with auditable actions |

A role does not automatically grant unlimited document access. Source-level and document-level authorization remain separate checks.

### Main experiences

1. **Secure sign-in** through Microsoft Entra ID.
2. **AI Chat** with streaming, conversation history, citations, and source previews.
3. **Knowledge Base** with documents, collections, classification, indexing state, and access assignments.
4. **Data Sources** to connect organization-managed systems and inspect sync history.
5. **Ingestion Jobs** with status, retry reasons, processing stages, and provenance.
6. **Evaluation Lab** for test datasets, evaluation runs, failure analysis, and comparison.
7. **Observability & Costs** for requests, traces, failures, usage, latency, model costs, and quality trends.
8. **Administration** for identity, settings, audit trails, retention policies, and cost controls.

## Technology stack

| Layer | Selected technology | Responsibility |
| --- | --- | --- |
| Frontend | **React + TypeScript + Vite** | Chat interface and admin/engineering dashboards |
| Backend | **Python 3.12+ + FastAPI** | APIs, orchestration, authorization, server-sent events |
| Application database | **PostgreSQL** | Users, roles, conversations, sources, sync jobs, settings, evaluations |
| Identity | **Microsoft Entra ID** | Organization sign-in, claims, application identities |
| Models and AI workbench | **Microsoft Foundry / Azure OpenAI** | Chat and embedding deployments, model experimentation, evaluation |
| Search | **Azure AI Search** | Keyword + vector + hybrid search, filters, ranking |
| Raw document storage | **Azure Blob Storage** | Private original files and ingestion artifacts |
| Event processing | **Azure Functions + Azure queueing** | Asynchronous ingestion, retries, event-driven jobs |
| Data engineering | **Azure Databricks + PySpark + Delta Lake** | Batch ETL, normalization, large-data workflows |
| Observability | **OpenTelemetry, Azure Monitor, Application Insights** | Distributed tracing, metrics, failures, dashboards, alerts |
| Secrets | **Azure Key Vault + managed identities** | Centralized credentials and least-privilege cloud access |
| CI/CD | **GitHub Actions + Docker** | Automated tests, builds, releases, environment promotion |
| Local storage emulator | **Azurite** | Azure Blob-compatible local development |
| Load and integration testing | **pytest, httpx, Playwright, Locust** | Backend tests, browser tests, load and resilience checks |

**Important separation:** Microsoft Foundry manages AI assets and model-related workflows; Azure AI Search performs retrieval; Azure Databricks performs data engineering. They are not interchangeable services.

## System architecture

~~~mermaid
flowchart TB
    U[Employee / Administrator] --> UI[React + TypeScript]
    UI --> API[FastAPI API + RAG Orchestrator]
    API --> AUTH[Microsoft Entra ID + Authorization]
    API <--> PG[(PostgreSQL)]
    API --> SEARCH[Azure AI Search]
    SEARCH --> API
    API --> LLM[Azure OpenAI Model Deployment]
    LLM --> API
    API --> UI

    DS[Uploads / Blob / SharePoint / Azure SQL] --> ING[Connectors + Source Scoping]
    ING --> BLOB[(Private Azure Blob Storage)]
    ING --> QUEUE[Queue + Azure Functions]
    QUEUE --> PARSE[Parse / Normalize / Chunk]
    QUEUE --> DBX[Databricks + PySpark Batch ETL]
    DBX --> PARSE
    PARSE --> EMB[Azure OpenAI Embeddings]
    EMB --> SEARCH

    API --> O11Y[OpenTelemetry + Application Insights]
    QUEUE --> O11Y
    DBX --> O11Y
    API --> EVAL[Evaluation Lab + Golden Datasets]
    O11Y --> DASH[Observability + Cost Dashboard]
    EVAL --> DASH
~~~

### Storage boundaries

- **PostgreSQL:** application metadata, identities and role mappings, conversations, ingestion status, evaluation runs, and audits.
- **Blob Storage:** originals, derived artifacts, and encrypted storage as configured.
- **Azure AI Search:** searchable text, embeddings, citations/provenance, and authorization metadata; it is not the source of truth for permissions.
- **Databricks/Delta Lake:** batch transformation datasets and operationally governed ETL outputs; not the chat request path.
- **Application Insights:** privacy-filtered telemetry rather than an unrestricted archive of prompts or document content.

## Data sources and connectors

NEUTRON supports **manual ingestion** and **organization-approved source connections**. Connecting a source never implies unrestricted access to all content it contains.

| Source | Integration path | Main use |
| --- | --- | --- |
| Direct file upload | FastAPI validation → private Blob Storage → queue | Small document collections and ad hoc uploads |
| Azure Blob Storage | Native Azure AI Search indexing or custom event-driven processing | Cloud documents, reports, archives |
| Microsoft SharePoint | Microsoft Graph and/or supported Azure AI Search knowledge-source capabilities, according to service maturity | Microsoft 365 enterprise libraries |
| Azure SQL Database | Query-scoped extraction and supported Azure AI Search indexing | Authorized structured business records |
| OneDrive | Microsoft Graph, approved folder scope | Personal/team Microsoft 365 files |
| Google Drive | OAuth + Google Drive API adapter | External cloud document integration |
| Confluence | Scoped Atlassian API adapter | Team and product knowledge |

### Connector administration

Each connector has:

- Approved source identity and authentication method (prefer **managed identities** or OAuth; avoid standing shared passwords).
- Explicit site, library, container, table, folder, or query scope.
- Connection test and permission validation.
- Full first synchronization; incremental sync where supported.
- Update detection, deletion propagation, and orphaned-chunk cleanup.
- Sync frequency, last run, processed/failed counters, retry controls.
- Mapping of source provenance, document owners, and access-control metadata.
- Audit trail for creation, credential rotation, rescope, and disconnection.

**SharePoint security:** an indexer or application credential may read content on behalf of a service, but that alone does **not** preserve every end-user permission. NEUTRON must propagate or validate source ACLs, enforce them at query time, and address permission revocation. Preview capabilities should be evaluated before depending on them for sensitive production workloads.

**Synchronization semantics:** indexed sources may be temporarily stale. The UI should expose sync freshness. High-sensitivity content can require live source authorization or a fail-closed policy until permissions are verified.

## Document ingestion and indexing

~~~mermaid
flowchart LR
    A[Source detected or file uploaded] --> B[Validate type / size / source scope]
    B --> C[Store original and register job]
    C --> D[Queue + Azure Functions]
    D --> E{Processing path}
    E -->|Individual documents| F[Python parser]
    E -->|Batch / large datasets| G[Databricks PySpark / Delta]
    G --> F
    F --> H[Text / tables / structure extraction]
    H --> I[Chunking + metadata + ACLs]
    I --> J[Content hash / deduplicate]
    J --> K[Embedding model]
    K --> L[Azure AI Search index]
    L --> M[Ready for authorized retrieval]
~~~

### Parsing and chunking contract

- Start with PDF, DOCX, TXT, and Markdown; treat XLSX/CSV as structured data with a separate mapping.
- Preserve headings, section paths, page numbers, table boundaries, document version, source ID, and source URL.
- Use structure-aware chunking with approximately **512 tokens and 64-token overlap** as an initial experiment, not a universal optimum.
- Extract tabular content with column headers; preserve table provenance and avoid splitting a related row without context.
- Add scanned PDF/OCR handling as needed; distinguish OCR-derived text and confidence from native text.
- Track checksum, content version, parser version, embedding model, dimensions, and indexing generation.
- Upsert changed chunks idempotently and delete stale chunks when content or permissions change.
- Send only the text needed for embeddings; sanitize and reject unsupported files.
- Treat retrieved documents and their embedded instructions as **untrusted data**, not system instructions.

### Databricks responsibility

Databricks provides hands-on enterprise data engineering without putting Spark in the critical path of every chat request.

- PySpark notebooks and jobs ingest larger approved batches.
- Delta Lake tables hold normalized, versioned intermediate datasets.
- Data-quality checks validate schema, missing fields, duplicate records, and corrupted rows.
- Incremental processing uses change tracking or upstream watermarks where available.
- A Databricks job emits validated outputs that NEUTRON's indexing adapter converts into search documents.
- Jobs are scheduled or event-triggered; compute auto-termination and job-based execution control costs.

## RAG and chat workflow

1. Entra ID authenticates the user; the backend resolves application roles and authorized document scope.
2. The user selects a knowledge collection and sends a question.
3. FastAPI validates the request and records a trace and conversation reference.
4. The query is embedded using the compatible deployed embedding model.
5. Azure AI Search runs **hybrid retrieval** (keyword + vector), applying permitted-source and document filters.
6. Optional semantic reranking and result filtering choose the most useful passages.
7. The backend **rechecks authorization** before including passages in the LLM context, especially when permissions might have changed.
8. The prompt includes vetted context, provenance identifiers, and instructions to abstain when evidence is insufficient.
9. Azure OpenAI streams the answer through FastAPI to React.
10. NEUTRON displays source citations, document/page links, and a feedback action.
11. Telemetry records latency, retrieval choices, token counts, model/version, errors, and safe evaluation metadata.

~~~mermaid
sequenceDiagram
    participant User
    participant UI as React
    participant API as FastAPI
    participant Search as Azure AI Search
    participant LLM as Azure OpenAI
    User->>UI: Ask a question
    UI->>API: Authorized chat request
    API->>API: Resolve document permissions
    API->>Search: Hybrid search + access filters
    Search-->>API: Candidate chunks and citations
    API->>API: Recheck access + assemble context
    API->>LLM: Prompt with authorized context
    LLM-->>API: Streamed answer
    API-->>UI: SSE tokens + citations
    UI-->>User: Answer and source previews
~~~

### Retrieval configuration

- Configurable top-K, vector weights, semantic ranker, content budget, and minimum quality checks.
- Search index stores document ID, chunk ID, source URL, page/section, classification, ACL metadata, embedding vector, and version.
- Relevance experiments compare vector-only, keyword-only, hybrid search, and optional reranking.
- Exact filenames, numeric identifiers, and critical keywords should benefit from lexical matching.
- Searches and citations must respect document permission and collection boundaries.
- No-answer behavior is preferred to fabricating unsupported details.

## Identity, privacy, and document security

NEUTRON is a **single-organization** deployment with multiple authorized users and access groups. It deliberately avoids a full multi-tenant platform.

### Mandatory controls

- Microsoft Entra ID sign-in; server-side token validation, audience/issuer checks, expiry, and secure session handling.
- RBAC for application functions, combined with **document-level access control**.
- Least-privilege Azure managed identities; Azure Key Vault for secrets and credential rotation.
- Private Blob containers; encryption in transit and at rest; signed or backend-mediated document access.
- Permission-scoped Azure AI Search queries **plus independent source/document authorization checks** as required.
- Strict separation between admin tools, engineering diagnostics, and employee chat access.
- Secure citations: no direct public Blob URLs, index bypass, unauthorized previews, or cross-user conversation access.
- Protection against prompt injection, untrusted tool output, sensitive prompt logging, and insecure file parsing.
- Auditable connector changes, document access, permission changes, and administrative actions.
- Retention and deletion rules for documents, conversations, indexing artifacts, backups, and traces.
- Fail-closed behavior when required permission state cannot be verified.

### Privacy and AI data handling

- Minimize context sent to Azure OpenAI.
- Never include raw passwords, API keys, bearer tokens, or connection strings in a prompt or telemetry.
- Redact or mask personal/confidential fields in logs; configure tracing content capture carefully.
- Review Azure model processing/retention, service region, and data residency against organization policy.
- Never assume Azure subscription RBAC alone implements document-level permissions for chatbot users.
- Do not use organizational content to train a custom model by default.

## AI evaluation and answer quality

NEUTRON includes an **Evaluation Lab**, not just a thumbs-up/down button.

### Evaluation dataset

A versioned golden dataset stores:

- User question and authorized user persona.
- Reference answer and verifiable source document/page/chunk.
- Expected relevant document IDs.
- Allowed/denied classification.
- Expected behavior for impossible, stale, and sensitive queries.

### Evaluation dimensions

| Measure | Question answered |
| --- | --- |
| Recall@K | Did retrieval find the expected evidence in the top K? |
| MRR / NDCG | Was relevant evidence ranked appropriately? |
| Groundedness / faithfulness | Is the answer supported by the retrieved context? |
| Answer correctness | Does it agree with a verified reference answer? |
| Citation precision | Do citations genuinely support the statements? |
| Answer relevance / completeness | Did the assistant answer the user's actual question? |
| Abstention quality | Does the assistant refuse to invent an answer when evidence is absent? |
| Security regression | Does any restricted information reach retrieval results or prompts? |

Use deterministic assertions where possible and AI-assisted judges where useful. Human-reviewed test data remains essential: model-as-judge scores are estimates, not proof of factual correctness.

### Quality gates

- Run test suites on retrieval algorithm, index schema, prompt, and model changes.
- Compare against a stored baseline with the same dataset and controlled settings.
- Fail deployments on critical permission leaks and agreed quality-regression thresholds.
- Keep evaluator/model/prompt version, cost, and execution metadata per run.
- Provide drill-down from a failed evaluation to the underlying retrieval trace.
- Sample production behavior within privacy policy rather than running expensive judges on every request.

## Observability, reliability, and cost optimization

### Distributed tracing

**OpenTelemetry** carries correlation IDs across FastAPI, search, Azure OpenAI, queues, functions, and ingestion workers. **Application Insights** provides Azure-native telemetry, monitoring, and alerts.

A trace should reveal:

- Request identifier, user/role pseudonym, resource and model version.
- Authentication and authorization outcome (without revealing secrets).
- Query embedding time and search time.
- Candidate counts, index version, retrieval mode and scores.
- Time to first token, full response latency, input/output/cached tokens.
- Error category, retry count, timeout, upstream dependency status.
- Safe references to feedback and evaluation results.

**Privacy rule:** full prompts, model responses, document content, or source URLs containing secrets are not collected by default. Telemetry content capture requires a specific, access-controlled policy.

### Operational dashboards

| Dashboard | Metrics |
| --- | --- |
| **Quality** | Groundedness, citation correctness, Recall@K, failed evaluations |
| **Chat performance** | Request rate, P50/P95 latency, time to first token, error rate |
| **Search** | Search latency, zero-result rate, retrieved chunk count, filter performance |
| **Ingestion** | Files processed, processing time, failures, queue depth, job retries |
| **Model usage** | Requests by deployment, input/output tokens, cached tokens, per-request estimate |
| **Azure spend** | Service spend, model estimates, cost per 1,000 answers, budget utilization |
| **Security** | Denied access, suspicious attempts, permission-sync failures, audit events |

### Optimization experiments

- **Retrieval:** compare top-K, hybrid weights, semantic reranker, and chunk sizes.
- **Context:** reduce irrelevant chunks and excessive conversation history.
- **Model choice:** benchmark smaller/cheaper deployments for simpler tasks; use quality gates before routing.
- **Embedding reuse:** hash unchanged chunks and avoid repeated processing.
- **Caching:** use supported prompt caching and safe caching of non-sensitive outputs where policy allows.
- **Infrastructure:** auto-terminate Databricks jobs, use event-driven functions, and monitor idle resources.
- **Reliability:** backoff and bounded retries for throttling, dead-letter queues, idempotent ingestion, timeout budgets, cancellation and graceful degradation.

Optimization claims must be supported by measured **quality, latency, and cost** comparisons. Azure budget alerts are notifications and not automatic hard spending caps.

## Application screens

### 1. AI Chat

![AI Chat](docs/screens/01-ai-chat.svg)

- Sidebar navigation and conversation history.
- Streaming messages and evidence-backed answers.
- Source citation panel with page/section details.
- Knowledge collection selector, feedback, and permissions-aware previews.

### 2. Data Sources and synchronization

![Data Sources](docs/screens/02-data-sources.svg)

- Connected enterprise sources, connection health, credential/permission scope, and latest sync.
- Add Source flow for upload, Azure Blob, SharePoint, Azure SQL, and later connectors.
- Job progress, incremental sync, error reasons, and retry controls.

### 3. Evaluation and Observability

![Evaluation and Observability](docs/screens/03-evaluation-observability.svg)

- Quality scorecards, evaluation runs, and retrieval regressions.
- Trace explorer and latency breakdowns.
- Azure model/token usage and cost comparisons.
- Alerts for security, quality, ingestion, and operational failure.

Additional application areas: **Knowledge Base**, **Ingestion Jobs**, **Administration**, and user **Settings**.

## Repository organization

~~~text
neutron/
├── README.md
├── .env.example
├── docker-compose.yml
├── Makefile
├── frontend/
│   ├── src/
│   │   ├── app/
│   │   ├── components/
│   │   ├── features/
│   │   │   ├── chat/
│   │   │   ├── knowledge-base/
│   │   │   ├── data-sources/
│   │   │   ├── ingestion/
│   │   │   ├── evaluation/
│   │   │   └── observability/
│   │   └── lib/
│   └── package.json
├── backend/
│   ├── app/
│   │   ├── api/
│   │   ├── auth/
│   │   ├── chat/
│   │   ├── connectors/
│   │   ├── ingestion/
│   │   ├── retrieval/
│   │   ├── evaluation/
│   │   ├── observability/
│   │   ├── infrastructure/
│   │   └── main.py
│   ├── tests/
│   └── requirements.txt
├── functions/
│   ├── ingestion_trigger/
│   └── host.json
├── databricks/
│   ├── notebooks/
│   ├── jobs/
│   └── schemas/
├── infra/
│   ├── bicep/
│   └── environments/
├── evaluations/
│   ├── datasets/
│   └── baselines/
├── docs/
│   ├── screens/
│   ├── architecture/
│   ├── decisions/
│   └── runbooks/
└── .github/
    └── workflows/
~~~

Organize features by business capability. Keep Azure SDK-specific code behind adapters so retrieval and model operations can be tested with fakes locally.

## Developer prerequisites

- **Python 3.12+**
- **Node.js 22 LTS+** and npm
- **Docker Desktop** and Docker Compose
- **Azure CLI** with access to an Azure subscription
- Microsoft Entra tenant permissions appropriate for registering or using an application
- Azure OpenAI chat and embedding model deployments
- Azure AI Search, Storage, and other resources created for the chosen environment
- **Azure Databricks** workspace or learning environment for PySpark/Delta exercises
- GitHub repository access for CI/CD

### Configuration

Use an ignored **.env** in local development and environment-specific Azure configuration/Key Vault in deployed environments.

~~~dotenv
APP_ENV=local
APP_NAME=neutron
DATABASE_URL=postgresql+psycopg://neutron:neutron@localhost:5432/neutron

# Entra authentication: register separate SPA and API application settings.
AZURE_TENANT_ID=
AZURE_API_CLIENT_ID=
VITE_ENTRA_CLIENT_ID=
VITE_ENTRA_AUTHORITY=
VITE_API_BASE_URL=http://localhost:8000

# Model deployment names and endpoints
AZURE_OPENAI_ENDPOINT=
AZURE_OPENAI_CHAT_DEPLOYMENT=
AZURE_OPENAI_EMBEDDING_DEPLOYMENT=
AZURE_OPENAI_API_VERSION=

# Azure AI Search
AZURE_SEARCH_ENDPOINT=
AZURE_SEARCH_INDEX_NAME=neutron-chunks

# Blob storage (use local Azurite or Azure credentials as appropriate)
AZURE_STORAGE_ACCOUNT_URL=
AZURE_STORAGE_CONTAINER=knowledge

# Observability
APPLICATIONINSIGHTS_CONNECTION_STRING=
OTEL_SERVICE_NAME=neutron-api

# Databricks batch processing
DATABRICKS_HOST=
DATABRICKS_JOB_ID=
~~~

Do not commit real credentials. Prefer **DefaultAzureCredential / managed identity** for backend Azure calls and service-specific least-privilege permissions. The frontend must never contain Azure OpenAI or Azure AI Search secrets.

## Local development

### Option A — Docker Compose

The repository's local execution contract is:

~~~bash
git clone https://github.com/summiya/neutron.git
cd neutron
cp .env.example .env
docker compose up --build
~~~

Local endpoints:

| Component | URL |
| --- | --- |
| React application | http://localhost:5173 |
| FastAPI API | http://localhost:8000 |
| FastAPI OpenAPI documentation | http://localhost:8000/docs |
| PostgreSQL | localhost:5432 |
| Azurite (Blob emulator) | http://localhost:10000 |

React and FastAPI run locally; Azure model and search calls use real configured Azure resources unless tests substitute fakes. The local container setup does not emulate Foundry, Azure OpenAI, Azure AI Search, or Databricks.

### Option B — Run application processes separately

~~~bash
# Infrastructure
docker compose up -d postgres azurite

# Backend
cd backend
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000

# In a separate terminal: frontend
cd frontend
npm install
npm run dev
~~~

On Windows, activate the Python virtual environment using the PowerShell activation script instead of the macOS/Linux command.

### Development checks

~~~bash
# From backend, with the environment active
pytest

# From frontend
npm run lint
npm run build

# Whole local stack
docker compose ps
~~~

The repository structure and commands above form the implementation contract for the project scaffold and developer environment.

## Azure deployment

### Resource topology

- **Microsoft Foundry project:** selected chat and embedding model deployments and evaluation configuration.
- **Azure AI Search:** searchable index with vector fields, text fields, provenance, and document security metadata.
- **Storage account:** private Blob containers for originals and processing outputs.
- **Azure Functions / Queue Storage:** ingestion triggers and bounded retries.
- **Azure Databricks:** on-demand batch transformation jobs with auto-termination.
- **API hosting:** Azure Container Apps (or equivalent managed Azure container hosting).
- **Frontend hosting:** Azure Static Web Apps (or equivalent static hosting).
- **PostgreSQL:** managed Azure Database for PostgreSQL or a secure equivalent.
- **Key Vault:** secrets, keys and external connector credentials.
- **Azure Monitor + Application Insights:** traces, metrics, alert rules, and dashboards.
- **Entra ID:** workforce authentication, group-based access, and managed identities.

### Release workflow

1. Run tests, formatting, type checking, and security/dependency scans in GitHub Actions.
2. Build immutable frontend and backend artifacts.
3. Provision or update Azure resources using infrastructure as code (Bicep).
4. Apply database migrations and controlled search-index schema changes.
5. Deploy to a restricted **staging** environment.
6. Run smoke tests, access-control regressions, RAG evaluations, and performance checks.
7. Promote the validated release to production with an explicit rollback path.
8. Track request quality, failures, spending, and permission-sync health after release.

**Cloud security:** public access should terminate at authenticated app entry points. Protect storage, index, model endpoints, databases, and admin actions via network isolation and identity controls according to chosen Azure tier and budget.

## API design

Illustrative REST/SSE contract:

| Endpoint | Behavior |
| --- | --- |
| GET /health | Liveness and basic readiness information |
| GET /api/v1/me | Current user, roles, and available collections |
| POST /api/v1/chat/stream | SSE conversational answer with citations |
| GET /api/v1/conversations | User-scoped conversation history |
| POST /api/v1/documents | Authorized upload and ingestion registration |
| GET /api/v1/documents | Permission-filtered document listing |
| GET /api/v1/documents/{id}/preview | Authorized, non-public source preview |
| GET /api/v1/data-sources | Authorized connector inventory |
| POST /api/v1/data-sources | Create an approved connector |
| POST /api/v1/data-sources/{id}/sync | Initiate a controlled sync |
| GET /api/v1/ingestion-jobs | Job status, progress, failures |
| POST /api/v1/evaluations/runs | Execute an evaluation dataset |
| GET /api/v1/evaluations/runs/{id} | Metrics and failed-case details |
| GET /api/v1/observability/summary | Engineering metrics and cost summary |

All endpoints require server-side authorization appropriate to the operation. Request, response, and event schemas are versioned and documented in OpenAPI.

## Testing and release criteria

### Automated test strategy

| Layer | Examples |
| --- | --- |
| Unit | Chunking, source metadata, token estimates, auth policies, citation formatting |
| Integration | PostgreSQL, Blob/Azurite, Azure AI Search adapter, model adapter, queue handlers |
| End-to-end | Entra login, upload, indexing, chat, citation preview, connector sync |
| Security | Unauthorized document retrieval, stale ACLs, revoked access, injection tests, credential leaks |
| Retrieval | Recall@K, ranking quality, no-evidence behavior, authorization-filter regressions |
| Evaluation | Golden reference answers, groundedness, citation precision, model comparison |
| Load | Chat throughput, P50/P95 latency, concurrency, rate limiting, saturation |
| Resilience | Upstream timeout, 429 retry, queue poison message, duplicate event, partial processing |
| Operations | Logs and traces, audit events, budget alerts, rollback, backup/restore exercises |

### Release acceptance criteria

- An authorized user can ask a question and see an answer with clickable source evidence.
- An unauthorized user cannot retrieve, cite, preview, or download restricted documents.
- Changes, deletions, and access revocations are propagated according to explicit freshness policy.
- Documents can be indexed through upload and at least one connected source.
- The ingestion pipeline is idempotent and exposes failures and retries.
- The evaluation suite produces repeatable quality results with versioned baselines.
- End-to-end traces and token/cost metrics are visible without dumping private content.
- CI/CD, smoke tests, load tests, backups, and a documented rollback method are exercised.
- Azure budget notifications and least-privilege service identities are configured.

## Project scope and boundaries

### NEUTRON is designed to do

- Answer questions using **authorized organizational knowledge**.
- Connect selected internal data sources with administrator approval.
- Produce citations and allow users to inspect supporting document passages.
- Evaluate retrieval and generation quality against verified test data.
- Track latency, errors, traces, token usage, and Azure service spending.
- Support multiple employees and roles inside **one organization**.
- Demonstrate Azure-native AI architecture, PySpark/Databricks processing, and DevOps engineering.

### NEUTRON is deliberately not designed to do

- **Full multi-tenancy / SaaS organization isolation.** NEXUS covers this use case.
- **Unrestricted access to SharePoint, OneDrive, email, or databases.** An organization administrator must approve scope and access.
- **Bypass source ACLs.** Search relevance is never permission.
- **Guarantee every generated answer is correct.** Evaluation, citations, and abstention reduce risk, but human review remains necessary for high-stakes decisions.
- **Automatically know every source change instantly.** Indexed data has sync latency; sensitive access checks may require live verification.
- **Run Databricks for each chat message.** Databricks is for data engineering and batch processing.
- **Execute autonomous business transactions or unrestricted agents.** Tool-driven multi-agent workflows belong in STRIDER.
- **Train foundation models on employee documents by default.**
- **Expose raw enterprise data or full prompts in general-purpose observability dashboards.**
- **Provide financial, medical, or legal decisions without appropriate human oversight.**

## Delivery milestones

| Milestone | Outcome |
| --- | --- |
| **M1 — Foundation** | React shell, FastAPI API, Entra integration, PostgreSQL, Foundry chat deployment |
| **M2 — Searchable knowledge** | File upload, Blob Storage, parsing/chunking, embeddings, Azure AI Search index |
| **M3 — Secure RAG** | Hybrid retrieval, citations, streaming, document authorization, chat history |
| **M4 — Enterprise ingestion** | Azure Functions, queueing, Blob/SharePoint connectors, sync controls |
| **M5 — Databricks** | PySpark batch transformations, Delta tables, data quality, index handoff |
| **M6 — Quality and operations** | Golden dataset, evaluations, OpenTelemetry, Application Insights, cost dashboards |
| **M7 — Cloud deployment** | CI/CD, staging, IaC, security tests, load tests, production release procedures |

## Microsoft documentation

- [Microsoft Foundry](https://learn.microsoft.com/en-us/azure/foundry/)
- [Azure AI Search overview](https://learn.microsoft.com/en-us/azure/search/search-what-is-azure-search)
- [Azure AI Search hybrid retrieval](https://learn.microsoft.com/en-us/azure/search/hybrid-search-overview)
- [Azure AI Search vector search](https://learn.microsoft.com/en-us/azure/search/vector-search-overview)
- [Observability in Microsoft Foundry](https://learn.microsoft.com/en-us/azure/foundry/concepts/observability)
- [Microsoft Foundry tracing](https://learn.microsoft.com/en-us/azure/foundry/observability/how-to/trace-agent-setup)
- [Azure Databricks documentation](https://learn.microsoft.com/en-us/azure/databricks/)
- [Azure Functions documentation](https://learn.microsoft.com/en-us/azure/azure-functions/)
- [Azure identity documentation](https://learn.microsoft.com/en-us/entra/identity/)

---

**NEUTRON** — *Enterprise knowledge, securely connected and intelligently understood.*
