# Architecture Document — Azure Search OpenAI Demo (C#)

> **RAG (Retrieval-Augmented Generation) application** built with .NET 8, Blazor WebAssembly, and Azure services.  
> This document describes the folder structure, component responsibilities, data flow, and how to add or modify files for RAG scenarios.

---

## Table of Contents

1. [High-Level Architecture](#high-level-architecture)
2. [Azure Services Used](#azure-services-used)
3. [Folder Structure](#folder-structure)
4. [Project Details](#project-details)
   - [MinimalApi (Backend)](#1-minimalapi--backend)
   - [ClientApp (Frontend)](#2-clientapp--frontend)
   - [SharedWebComponents](#3-sharedwebcomponents)
   - [Shared (Models & Services)](#4-shared--models--services)
   - [EmbedFunctions (Azure Functions)](#5-embedfunctions--azure-functions)
   - [PrepareDocs (Document Preparation)](#6-preparedocs--document-preparation)
5. [RAG Data Flow](#rag-data-flow)
6. [Infrastructure as Code](#infrastructure-as-code)
7. [Deployment with AZD](#deployment-with-azd)
8. [How-To Guides](#how-to-guides)
   - [Add New Documents for RAG](#add-new-documents-for-rag)
   - [Change the RAG Prompt / System Message](#change-the-rag-prompt--system-message)
   - [Change the OpenAI Model](#change-the-openai-model)
   - [Add a New API Endpoint](#add-a-new-api-endpoint)
   - [Add a New Blazor Page](#add-a-new-blazor-page)
   - [Modify the Search Index Schema](#modify-the-search-index-schema)
   - [Change Embedding Strategy](#change-embedding-strategy)
   - [Add a New Azure Resource](#add-a-new-azure-resource)

---

## High-Level Architecture

```
???????????????????????????????????????????????????????????????????????
?                         User (Browser)                              ?
?                    Blazor WebAssembly (WASM)                        ?
?              app/frontend + app/SharedWebComponents                 ?
???????????????????????????????????????????????????????????????????????
                           ? HTTP API calls
                           ?
???????????????????????????????????????????????????????????????????????
?                   MinimalApi (Backend)                               ?
?                 app/backend (Container App)                          ?
?                                                                     ?
?  ?????????????????????????????????????????????????????????????      ?
?  ?        ReadRetrieveReadChatService (RAG Engine)           ?      ?
?  ?  1. Generate search query from user question (LLM)       ?      ?
?  ?  2. Search Azure AI Search (text + vector)               ?      ?
?  ?  3. Build prompt with retrieved docs + history            ?      ?
?  ?  4. Generate answer via Azure OpenAI (LLM)               ?      ?
?  ?  5. Optionally generate follow-up questions               ?      ?
?  ?????????????????????????????????????????????????????????????      ?
???????????????????????????????????????????????????????????????????????
       ?              ?              ?               ?
       ?              ?              ?               ?
?????????????? ?????????????? ?????????????? ??????????????????
?  Azure AI  ? ?   Azure    ? ?   Azure    ? ?  Azure Key     ?
?   Search   ? ?  OpenAI    ? ?  Storage   ? ?    Vault       ?
?  (Index)   ? ?  (GPT +    ? ?  (Blob -   ? ?  (Secrets)     ?
?            ? ? Embedding) ? ?  PDFs)     ? ?                ?
?????????????? ?????????????? ?????????????? ??????????????????
                                     ?
                                     ? Blob Trigger
                                     ?
                              ???????????????
                              ?   Azure      ?
                              ?  Functions   ?
                              ? (Embed &     ?
                              ?  Index)      ?
                              ????????????????
                                     ?
                         ?????????????????????????
                         ?           ?           ?
                   ???????????? ?????????? ????????????
                   ? Doc Intel ? ? OpenAI ? ? AI Search?
                   ? (Parse)  ? ?(Embed) ? ? (Index)  ?
                   ???????????? ?????????? ????????????
```

---

## Azure Services Used

| Azure Service | Purpose | Provisioned By |
|---|---|---|
| **Azure Container Apps** | Hosts the backend API + Blazor WASM frontend | `infra/app/web.bicep` |
| **Azure Functions** | Blob-triggered function to embed & index documents | `infra/app/function.bicep` |
| **Azure OpenAI** | Chat completion (GPT-4o-mini) + text embeddings (ada-002) | `infra/core/ai/cognitiveservices.bicep` |
| **Azure AI Search** | Vector + text search index for document retrieval | `infra/core/search/search-services.bicep` |
| **Azure Blob Storage** | Stores uploaded PDF documents in `content` container | `infra/core/storage/storage-account.bicep` |
| **Azure AI Document Intelligence** | Extracts text from PDFs (form recognition / OCR) | `infra/core/ai/cognitiveservices.bicep` |
| **Azure Key Vault** | Stores secrets (endpoints, keys, deployment names) | `infra/core/security/keyvault.bicep` |
| **Azure Container Registry** | Stores Docker images for the backend | `infra/core/host/container-apps.bicep` |
| **Azure Application Insights** | Monitoring & telemetry | `infra/core/monitor/monitoring.bicep` |
| **Azure Computer Vision** *(optional)* | Image embeddings for vision-based RAG | `infra/core/ai/cognitiveservices.bicep` |

---

## Folder Structure

```
azure-search-openai-demo-csharp/
?
??? azure.yaml                      # AZD deployment manifest
??? ARCHITECTURE.md                  # This document
?
??? app/                             # All application code
?   ??? backend/                     # MinimalApi — ASP.NET Core backend
?   ?   ??? Program.cs               #   App startup, DI, middleware
?   ?   ??? Extensions/
?   ?   ?   ??? WebApplicationExtensions.cs  # API endpoint definitions
?   ?   ?   ??? ServiceCollectionExtensions.cs # Azure service DI registration
?   ?   ?   ??? ConfigurationExtensions.cs     # Config helpers
?   ?   ?   ??? KeyVaultConfigurationBuilderExtensions.cs
?   ?   ?   ??? SearchClientExtensions.cs
?   ?   ?   ??? ChatTurnExtensions.cs
?   ?   ??? Services/
?   ?   ?   ??? ReadRetrieveReadChatService.cs  # ? Core RAG logic
?   ?   ?   ??? AzureBlobStorageService.cs      # File upload to Blob Storage
?   ?   ??? Pages/
?   ?   ?   ??? Error.cshtml(.cs)    # Error page (Razor Pages)
?   ?   ??? Properties/
?   ?   ?   ??? launchSettings.json  # Local dev launch settings
?   ?   ??? appsettings.json         # App configuration
?   ?   ??? CorpusRecord.cs          # Record types for corpus data
?   ?   ??? GlobalUsings.cs          # Global using directives
?   ?   ??? MinimalApi.csproj        # Project file
?   ?
?   ??? frontend/                    # ClientApp — Blazor WebAssembly
?   ?   ??? Program.cs               #   WASM startup
?   ?   ??? Components/
?   ?   ?   ??? PdfViewerDialog.razor(.cs)   # PDF viewer component
?   ?   ?   ??? ImageViewerDialog.razor      # Image viewer component
?   ?   ??? Services/
?   ?   ?   ??? WebPdfViewer.cs              # PDF rendering service
?   ?   ?   ??? TextToSpeechPreferencesListenerService.cs
?   ?   ??? Extensions/
?   ?   ?   ??? StringExtensions.cs
?   ?   ??? Interop/
?   ?   ?   ??? JavaScriptModule.cs          # JS interop
?   ?   ??? Options/
?   ?   ?   ??? AppSettings.cs
?   ?   ??? wwwroot/                         # Static assets
?   ?   ?   ??? index.html                   # WASM entry point
?   ?   ?   ??? js/                          # JavaScript files
?   ?   ?   ??? favicon.png
?   ?   ?   ??? manifest.json
?   ?   ??? _Imports.razor
?   ?   ??? GlobalUsings.cs
?   ?   ??? ClientApp.csproj
?   ?
?   ??? SharedWebComponents/         # Shared Blazor components & pages
?   ?   ??? Pages/
?   ?   ?   ??? Chat.razor(.cs)      #   Chat page (RAG conversation UI)
?   ?   ?   ??? Docs.razor(.cs)      #   Document upload/management page
?   ?   ?   ??? Index.razor(.cs)     #   Home/landing page (Ask question)
?   ?   ?   ??? VoiceChat.razor(.cs) #   Voice chat page
?   ?   ??? Components/
?   ?   ?   ??? Answer.razor(.cs)    #   Renders AI answers with citations
?   ?   ?   ??? SupportingContent.razor(.cs) # Shows retrieved doc chunks
?   ?   ?   ??? VoiceDialog.razor(.cs)
?   ?   ?   ??? VoiceTextInput.razor(.cs)
?   ?   ??? Shared/
?   ?   ?   ??? MainLayout.razor(.cs) #  App layout
?   ?   ?   ??? NavMenu.razor         #  Navigation menu
?   ?   ?   ??? LogoutDisplay.razor
?   ?   ??? Services/
?   ?   ?   ??? ApiClient.cs          #  HTTP client to call backend API
?   ?   ?   ??? OpenAIPromptQueue.cs  #  Queue for streaming prompts
?   ?   ?   ??? IPdfViewer.cs
?   ?   ?   ??? ITextToSpeechPreferencesListener.cs
?   ?   ??? Models/
?   ?   ?   ??? AnswerResult.cs
?   ?   ?   ??? CitationDetails.cs
?   ?   ?   ??? RequestSettingsOverrides.cs
?   ?   ?   ??? UserQuestion.cs
?   ?   ?   ??? VoicePreferences.cs
?   ?   ?   ??? ...
?   ?   ??? wwwroot/css/app.css       # Application styles
?   ?   ??? App.razor                 # Root Blazor component
?   ?   ??? _Imports.razor
?   ?   ??? SharedWebComponents.csproj
?   ?
?   ??? shared/Shared/               # Shared library (models & services)
?   ?   ??? Models/
?   ?   ?   ??? ChatRequest.cs        #   Chat API request model
?   ?   ?   ??? ChatMessage.cs        #   Chat message model
?   ?   ?   ??? ChatChunkResponse.cs  #   Streaming response chunk
?   ?   ?   ??? PromptRequest.cs      #   Simple prompt request
?   ?   ?   ??? PromptResponse.cs     #   Prompt response
?   ?   ?   ??? RequestOverrides.cs   #   RAG settings (top, temperature, etc.)
?   ?   ?   ??? ResponseChoice.cs     #   Answer + context
?   ?   ?   ??? DocumentResponse.cs   #   Document list response
?   ?   ?   ??? UploadDocumentsResponse.cs
?   ?   ?   ??? Section.cs            #   Document section/chunk
?   ?   ?   ??? Approach.cs           #   RAG approach enum
?   ?   ?   ??? RetrievalMode.cs      #   Text / Vector / Hybrid
?   ?   ?   ??? ...
?   ?   ??? Services/
?   ?   ?   ??? AzureSearchEmbedService.cs  # Embeds & indexes doc chunks
?   ?   ?   ??? AzureSearchService.cs       # Queries the search index
?   ?   ?   ??? ISearchService.cs           # Search service interface
?   ?   ?   ??? IEmbedService.cs            # Embed service interface
?   ?   ?   ??? AzureComputerVisionService.cs
?   ?   ?   ??? IComputerVisionService.cs
?   ?   ??? Json/
?   ?   ?   ??? SerializerOptions.cs
?   ?   ??? EmbeddingType.cs
?   ?   ??? Shared.csproj
?   ?
?   ??? functions/EmbedFunctions/     # Azure Functions — document processing
?   ?   ??? EmbeddingFunction.cs      #   Blob trigger ? embed document
?   ?   ??? Program.cs               #   Function app startup & DI
?   ?   ??? Services/
?   ?   ?   ??? EmbeddingAggregateService.cs  # Orchestrates embedding
?   ?   ?   ??? EmbedServiceFactory.cs        # Factory for embed providers
?   ?   ?   ??? MilvusEmbedService.cs         # Milvus vector DB support
?   ?   ?   ??? PineconeEmbedService.cs       # Pinecone vector DB support
?   ?   ?   ??? QdrantEmbedService.cs         # Qdrant vector DB support
?   ?   ??? host.json
?   ?   ??? EmbedFunctions.csproj
?   ?
?   ??? prepdocs/PrepareDocs/         # CLI tool — initial document ingestion
?   ?   ??? PrepareDocs.csproj
?   ?
?   ??? tests/
?   ?   ??? ClientApp.Tests/          # Frontend unit tests
?   ?   ??? MinimalApi.Tests/         # Backend API tests
?   ?
?   ??? Dockerfile                    # Docker image for backend + frontend
?
??? data/                             # ?? Sample documents for RAG
?   ??? Benefit_Options.pdf
?   ??? employee_handbook.pdf
?   ??? Northwind_Health_Plus_Benefits_Details.pdf
?   ??? Northwind_Standard_Benefits_Details.pdf
?   ??? PerksPlus.pdf
?   ??? role_library.pdf
?   ??? imgs/                         # Images for vision-based RAG
?
??? infra/                            # ??? Bicep infrastructure-as-code
?   ??? main.bicep                    #   Main deployment template
?   ??? main.parameters.json          #   Deployment parameters
?   ??? abbreviations.json            #   Resource naming abbreviations
?   ??? app/
?   ?   ??? web.bicep                 #   Container App (backend)
?   ?   ??? function.bicep            #   Azure Function App
?   ??? core/
?       ??? ai/                       #   OpenAI, Doc Intelligence, Vision
?       ??? host/                     #   Container Apps, App Service Plan
?       ??? monitor/                  #   Application Insights, Log Analytics
?       ??? search/                   #   Azure AI Search
?       ??? security/                 #   Key Vault, RBAC roles
?       ??? storage/                  #   Storage Account
?
??? scripts/                          # ?? Deployment scripts
?   ??? prepdocs.ps1                  #   Post-provision: ingest documents
?   ??? prepdocs.sh                   #   Same, for Linux/macOS
?   ??? roles.ps1                     #   Assign Azure RBAC roles
?   ??? roles.sh
?
??? .github/                          # GitHub Actions workflows
??? .devcontainer/                    # Dev container configuration
??? .vscode/                          # VS Code settings
??? docs/                             # Additional documentation
```

---

## Project Details

### 1. MinimalApi — Backend

**Path:** `app/backend/`  
**Hosted on:** Azure Container Apps  
**Framework:** ASP.NET Core 8 Minimal API + Razor Pages

| File | Responsibility |
|---|---|
| `Program.cs` | App startup: configures DI, middleware, Key Vault, Redis cache, App Insights |
| `Extensions/WebApplicationExtensions.cs` | Defines all API endpoints (`/api/chat`, `/api/documents`, `/api/openai/chat`, `/api/images`) |
| `Extensions/ServiceCollectionExtensions.cs` | Registers Azure services (OpenAI, Search, Storage, etc.) in DI |
| `Services/ReadRetrieveReadChatService.cs` | **? Core RAG engine** — implements the Read-Retrieve-Read pattern |
| `Services/AzureBlobStorageService.cs` | Handles document upload to Azure Blob Storage |

**API Endpoints:**

| Method | Route | Description |
|---|---|---|
| POST | `/api/chat` | RAG chat — sends question, retrieves docs, generates answer |
| POST | `/api/openai/chat` | Streaming chat with "Blazor Clippy" assistant |
| POST | `/api/documents` | Upload documents to Blob Storage |
| GET | `/api/documents` | List all documents in Blob Storage |
| POST | `/api/images` | Generate images via DALL-E |
| GET | `/api/enableLogout` | Check if logout is enabled |

### 2. ClientApp — Frontend

**Path:** `app/frontend/`  
**Framework:** Blazor WebAssembly (.NET 8)

The WASM app runs entirely in the browser. It communicates with the backend via HTTP API calls through `ApiClient.cs`. The main UI pages and components live in `SharedWebComponents`.

### 3. SharedWebComponents

**Path:** `app/SharedWebComponents/`  
**Framework:** Razor Class Library (Blazor)

Contains all shared UI pages, components, and services used by the frontend:

| Page | Route | Description |
|---|---|---|
| `Index.razor` | `/` | Home page — single question/answer (Ask) |
| `Chat.razor` | `/chat` | Multi-turn conversation with chat history |
| `Docs.razor` | `/docs` | Upload & manage documents |
| `VoiceChat.razor` | `/voicechat` | Voice-based chat interface |

| Key Component | Description |
|---|---|
| `Answer.razor` | Renders AI answers with citation links |
| `SupportingContent.razor` | Displays retrieved document chunks |
| `ApiClient.cs` | HTTP client that calls backend API endpoints |

### 4. Shared — Models & Services

**Path:** `app/shared/Shared/`

Contains shared models and service interfaces used by both the backend and functions:

| Category | Key Files | Description |
|---|---|---|
| **Models** | `ChatRequest.cs`, `ChatMessage.cs`, `RequestOverrides.cs`, `ResponseChoice.cs` | Request/response DTOs for the API |
| **Services** | `AzureSearchEmbedService.cs` | Chunks documents, generates embeddings, indexes to AI Search |
| **Services** | `AzureSearchService.cs` | Queries the search index (text + vector + hybrid) |
| **Interfaces** | `ISearchService.cs`, `IEmbedService.cs` | Abstractions for search and embedding |

### 5. EmbedFunctions — Azure Functions

**Path:** `app/functions/EmbedFunctions/`  
**Hosted on:** Azure Functions (Consumption plan)

**Trigger:** When a blob is uploaded to the `content` container in Azure Storage, the function automatically:
1. Reads the blob (PDF)
2. Extracts text using Azure AI Document Intelligence
3. Chunks the text into sections
4. Generates embeddings using Azure OpenAI
5. Indexes the chunks into Azure AI Search

| File | Description |
|---|---|
| `EmbeddingFunction.cs` | Blob trigger function — entry point |
| `Services/EmbeddingAggregateService.cs` | Orchestrates the embedding pipeline |
| `Services/EmbedServiceFactory.cs` | Factory pattern for different vector stores |

### 6. PrepareDocs — Document Preparation

**Path:** `app/prepdocs/PrepareDocs/`

CLI tool that runs as a **post-provision hook** (`scripts/prepdocs.ps1`). It processes all documents in the `data/` folder and uploads them to Azure Blob Storage, which then triggers the embedding function.

---

## RAG Data Flow

### Document Ingestion Flow

```
1. PDF placed in data/ folder (or uploaded via /api/documents)
        ?
        ?
2. Uploaded to Azure Blob Storage ("content" container)
        ?
        ?  (Blob Trigger)
3. Azure Function (EmbeddingFunction) fires
        ?
        ?
4. Azure AI Document Intelligence extracts text from PDF
        ?
        ?
5. Text is chunked into sections (app/shared/Shared/Services/AzureSearchEmbedService.cs)
        ?
        ?
6. Azure OpenAI generates vector embeddings for each chunk
        ?
        ?
7. Chunks + embeddings are indexed in Azure AI Search
```

### Query Flow (Read-Retrieve-Read)

```
1. User types a question in the Blazor UI
        ?
        ?
2. Frontend sends POST /api/chat with question + history
        ?
        ?
3. ReadRetrieveReadChatService processes the request:
        ?
        ??? Step 1: LLM generates an optimized search query from the user question
        ?
        ??? Step 2: Azure AI Search retrieves relevant document chunks
        ?           (supports Text, Vector, or Hybrid retrieval modes)
        ?
        ??? Step 2.5: (Optional) If vision enabled, retrieve relevant images
        ?
        ??? Step 3: Build prompt with retrieved sources + conversation history
        ?           ? LLM generates answer as JSON { answer, thoughts }
        ?
        ??? Step 4: (Optional) LLM generates follow-up questions
        ?
        ?
4. Response returned with answer, citations, thoughts, and follow-up questions
```

---

## Infrastructure as Code

All Azure resources are defined in `infra/` using **Bicep**:

| File | Resources |
|---|---|
| `infra/main.bicep` | Main orchestrator — defines all parameters and modules |
| `infra/main.parameters.json` | Default parameter values |
| `infra/app/web.bicep` | Container App for the backend |
| `infra/app/function.bicep` | Azure Function App |
| `infra/core/ai/cognitiveservices.bicep` | Azure OpenAI, Document Intelligence, Computer Vision |
| `infra/core/host/container-apps.bicep` | Container Apps Environment + Container Registry |
| `infra/core/host/appserviceplan.bicep` | App Service Plan (for Functions) |
| `infra/core/search/search-services.bicep` | Azure AI Search |
| `infra/core/storage/storage-account.bicep` | Azure Storage Account |
| `infra/core/security/keyvault.bicep` | Azure Key Vault |
| `infra/core/security/keyvault-secrets.bicep` | Key Vault secrets |
| `infra/core/security/role.bicep` | RBAC role assignments |
| `infra/core/monitor/monitoring.bicep` | App Insights + Log Analytics |

---

## Deployment with AZD

This project uses **Azure Developer CLI (azd)** for deployment, configured in `azure.yaml`:

```bash
# Authenticate
azd auth login

# Create an environment
azd env new <environment-name>

# Provision Azure resources + deploy application
azd up
```

**What happens during `azd up`:**

1. **Provision** — Creates all Azure resources defined in `infra/main.bicep`
2. **Deploy** — Builds and deploys:
   - `web` service ? Container App (Docker build from `app/Dockerfile`)
   - `function` service ? Azure Functions
3. **Post-provision hook** — Runs `scripts/prepdocs.ps1` which:
   - Processes all PDFs in `data/` folder
   - Uploads them to Azure Blob Storage
   - Triggers the embedding function to index them

---

## How-To Guides

### Add New Documents for RAG

**To add documents that get indexed during initial deployment:**

1. Place your PDF files in the `data/` folder:
   ```
   data/
   ??? your-new-document.pdf    ? Add here
   ??? Benefit_Options.pdf
   ??? ...
   ```

2. Re-run the document preparation:
   ```bash
   azd env set AZD_PREPDOCS_RAN "false"
   azd provision
   ```
   This re-triggers `scripts/prepdocs.ps1` which processes all files in `data/`.

**To add documents at runtime (after deployment):**

1. Use the **Docs page** in the web UI (`/docs`) to upload PDF files, **OR**
2. Upload files directly to the `content` container in Azure Blob Storage
3. The Azure Function blob trigger will automatically embed and index the new document

**Supported formats:** PDF files (processed by Azure AI Document Intelligence)

---

### Change the RAG Prompt / System Message

Edit `app/backend/Services/ReadRetrieveReadChatService.cs`:

**System message** (line ~108):
```csharp
var answerChat = new ChatHistory(
    "You are a system assistant who helps the company employees with their questions. Be brief in your answers");
```
Change this string to customize the AI's persona and behavior.

**Answer format prompt** (line ~140):
```csharp
var prompt = @$" ## Source ##
{documentContents}
## End ##

You answer needs to be a json object with the following format.
...
```
Modify this to change how the AI uses the retrieved documents.

**Search query generation prompt** (line ~87):
```csharp
var getQueryChat = new ChatHistory(@"You are a helpful AI assistant, generate search query for followup question.
Make your respond simple and precise. Return the query only, do not return any other text.
...
```

---

### Change the OpenAI Model

**Option A: Change deployment parameters** (recommended)

Edit `infra/main.bicep` to change the model:
```bicep
param azureOpenAIChatGptModelName string = 'gpt-4o-mini'    // Change model
param azureOpenAIChatGptModelVersion string = '2024-07-18'   // Change version
param azureEmbeddingModelName string = 'text-embedding-ada-002' // Change embedding model
```

Then re-provision:
```bash
azd provision
```

**Option B: Set via azd environment variables**

```bash
azd env set AZURE_OPENAI_CHATGPT_MODEL_NAME "gpt-4o"
azd env set AZURE_OPENAI_CHATGPT_MODEL_VERSION "2024-05-13"
azd provision
```

---

### Add a New API Endpoint

1. **Define the endpoint** in `app/backend/Extensions/WebApplicationExtensions.cs`:

```csharp
internal static WebApplication MapApi(this WebApplication app)
{
    var api = app.MapGroup("api");

    // ... existing endpoints ...

    // Add your new endpoint
    api.MapGet("my-new-endpoint", OnGetMyNewEndpointAsync);

    return app;
}

private static async Task<IResult> OnGetMyNewEndpointAsync(
    [FromServices] MyService myService,
    CancellationToken cancellationToken)
{
    // Your logic here
    return TypedResults.Ok(result);
}
```

2. **Register any new services** in `app/backend/Extensions/ServiceCollectionExtensions.cs`

3. **Add the client-side API call** in `app/SharedWebComponents/Services/ApiClient.cs`

---

### Add a New Blazor Page

1. **Create the page** in `app/SharedWebComponents/Pages/`:

   - `MyPage.razor` — Razor markup
   - `MyPage.razor.cs` — Code-behind (optional)

```razor
@page "/mypage"

<PageTitle>My Page</PageTitle>

<h1>My Custom Page</h1>
```

2. **Add navigation** in `app/SharedWebComponents/Shared/NavMenu.razor`:

```razor
<MudNavLink Href="mypage" Icon="@Icons.Material.Filled.Star">
    My Page
</MudNavLink>
```

---

### Modify the Search Index Schema

The search index schema is defined in `app/shared/Shared/Services/AzureSearchEmbedService.cs`. This file controls:

- How documents are chunked into sections
- What fields are stored in the search index
- How embeddings are generated and associated with chunks

To add a new field to the index, update the index creation logic in this file and update `AzureSearchService.cs` to query the new field.

---

### Change Embedding Strategy

The project supports multiple vector store backends:

| Provider | File | Description |
|---|---|---|
| **Azure AI Search** (default) | `app/shared/Shared/Services/AzureSearchEmbedService.cs` | Uses Azure AI Search as vector store |
| Milvus | `app/functions/EmbedFunctions/Services/MilvusEmbedService.cs` | Milvus vector DB |
| Pinecone | `app/functions/EmbedFunctions/Services/PineconeEmbedService.cs` | Pinecone vector DB |
| Qdrant | `app/functions/EmbedFunctions/Services/QdrantEmbedService.cs` | Qdrant vector DB |

To switch providers, update `app/functions/EmbedFunctions/Services/EmbedServiceFactory.cs`.

---

### Add a New Azure Resource

1. **Create a Bicep module** in `infra/core/<category>/`:
   ```
   infra/core/database/cosmosdb.bicep
   ```

2. **Reference it** in `infra/main.bicep`:
   ```bicep
   module cosmosDb 'core/database/cosmosdb.bicep' = {
     name: 'cosmosdb'
     scope: resourceGroup
     params: { ... }
   }
   ```

3. **Add secrets** to Key Vault in `infra/main.bicep` under `keyVaultSecrets`

4. **Add role assignments** if using managed identity authentication

5. **Re-provision**:
   ```bash
   azd provision
   ```

---

## Key Configuration & Environment Variables

| Variable | Description | Set In |
|---|---|---|
| `UseAOAI` | Use Azure OpenAI (`true`) vs OpenAI API (`false`) | Key Vault |
| `AzureOpenAiServiceEndpoint` | Azure OpenAI endpoint URL | Key Vault |
| `AzureOpenAiChatGptDeployment` | Chat model deployment name | Key Vault |
| `AzureOpenAiEmbeddingDeployment` | Embedding model deployment name | Key Vault |
| `AzureSearchServiceEndpoint` | Azure AI Search endpoint | Key Vault |
| `AzureSearchIndex` | Search index name (default: `gptkbindex`) | Key Vault |
| `AzureStorageAccountEndpoint` | Blob storage endpoint | Key Vault |
| `AzureStorageContainer` | Blob container name (default: `content`) | Key Vault |
| `UseVision` | Enable vision/image RAG | Key Vault |

---

## Testing

| Project | Path | Description |
|---|---|---|
| `ClientApp.Tests` | `app/tests/ClientApp.Tests/` | Unit tests for Blazor frontend components |
| `MinimalApi.Tests` | `app/tests/MinimalApi.Tests/` | Unit tests for backend API logic |

Run tests:
```bash
dotnet test app/tests/ClientApp.Tests/
dotnet test app/tests/MinimalApi.Tests/
```

---

## Quick Reference: Where to Make Changes

| I want to... | Edit this file/folder |
|---|---|
| Change the AI's behavior/persona | `app/backend/Services/ReadRetrieveReadChatService.cs` |
| Add new documents | `data/` folder or upload via `/docs` page |
| Change the UI layout | `app/SharedWebComponents/Shared/MainLayout.razor` |
| Add a new page | `app/SharedWebComponents/Pages/` |
| Add a new API endpoint | `app/backend/Extensions/WebApplicationExtensions.cs` |
| Change how documents are chunked | `app/shared/Shared/Services/AzureSearchEmbedService.cs` |
| Change how documents are searched | `app/shared/Shared/Services/AzureSearchService.cs` |
| Change Azure infrastructure | `infra/main.bicep` and `infra/core/` or `infra/app/` |
| Change the OpenAI model | `infra/main.bicep` (parameters) |
| Modify the embedding pipeline | `app/functions/EmbedFunctions/` |
| Change deployment config | `azure.yaml` |
| Add CSS styles | `app/SharedWebComponents/wwwroot/css/app.css` |
| Add shared models/DTOs | `app/shared/Shared/Models/` |
