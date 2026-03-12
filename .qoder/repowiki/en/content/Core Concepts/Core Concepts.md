# Core Concepts

<cite>
**Referenced Files in This Document**
- [src/memu/__init__.py](file://src/memu/__init__.py)
- [src/memu/_core.pyi](file://src/memu/_core.pyi)
- [src/memu/database/models.py](file://src/memu/database/models.py)
- [src/memu/database/interfaces.py](file://src/memu/database/interfaces.py)
- [src/memu/database/factory.py](file://src/memu/database/factory.py)
- [src/memu/app/service.py](file://src/memu/app/service.py)
- [src/memu/app/memorize.py](file://src/memu/app/memorize.py)
- [src/memu/app/retrieve.py](file://src/memu/app/retrieve.py)
- [src/memu/app/settings.py](file://src/memu/app/settings.py)
- [src/memu/workflow/pipeline.py](file://src/memu/workflow/pipeline.py)
- [src/memu/workflow/runner.py](file://src/memu/workflow/runner.py)
- [src/memu/workflow/step.py](file://src/memu/workflow/step.py)
- [src/memu/llm/backends/base.py](file://src/memu/llm/backends/base.py)
- [src/memu/embedding/backends/base.py](file://src/memu/embedding/backends/base.py)
- [examples/proactive/proactive.py](file://examples/proactive/proactive.py)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Project Structure](#project-structure)
3. [Core Components](#core-components)
4. [Architecture Overview](#architecture-overview)
5. [Detailed Component Analysis](#detailed-component-analysis)
6. [Dependency Analysis](#dependency-analysis)
7. [Performance Considerations](#performance-considerations)
8. [Troubleshooting Guide](#troubleshooting-guide)
9. [Conclusion](#conclusion)

## Introduction
This document explains memU’s core architecture and design principles with a focus on:
- The memory layers: Resource, Item, Category
- The memory-as-file-system paradigm and how it relates to the three layers
- Proactive memory lifecycle and the dual-mode retrieval system (RAG vs LLM-based)
- The workflow pipeline architecture and how memory flows from ingestion to retrieval
- Provider abstraction for LLMs and embeddings
- Scope management for user contexts
- Storage backends and their relationships

The goal is to give beginners a clear mental model while providing experienced developers the technical depth to implement custom features and integrate new providers or backends.

## Project Structure
At a high level, memU is organized around:
- Application service and mixins for memory ingestion and retrieval
- A pluggable workflow pipeline engine
- A provider-agnostic database layer with multiple storage backends
- Provider abstractions for LLMs and embeddings
- Settings and configuration models for memory categories, retrieval, and memorization

```mermaid
graph TB
subgraph "Application"
Svc["MemoryService<br/>service.py"]
Mem["MemorizeMixin<br/>memorize.py"]
Ret["RetrieveMixin<br/>retrieve.py"]
Cfg["Settings<br/>settings.py"]
end
subgraph "Workflow Engine"
Pipe["PipelineManager<br/>pipeline.py"]
Step["WorkflowStep<br/>step.py"]
Run["WorkflowRunner<br/>runner.py"]
end
subgraph "Storage Layer"
IFace["Database Protocol<br/>interfaces.py"]
Factory["build_database()<br/>factory.py"]
InMem["In-Memory Impl"]
Postgres["Postgres Impl"]
SQLite["SQLite Impl"]
end
subgraph "Providers"
LLM["LLM Backend Base<br/>llm/backends/base.py"]
Emb["Embedding Backend Base<br/>embedding/backends/base.py"]
end
Svc --> Mem
Svc --> Ret
Svc --> Pipe
Svc --> Run
Svc --> Factory
Factory --> IFace
IFace --> InMem
IFace --> Postgres
IFace --> SQLite
Svc --> LLM
Svc --> Emb
Pipe --> Step
Run --> Step
```

**Diagram sources**
- [src/memu/app/service.py](file://src/memu/app/service.py#L49-L95)
- [src/memu/app/memorize.py](file://src/memu/app/memorize.py#L97-L166)
- [src/memu/app/retrieve.py](file://src/memu/app/retrieve.py#L106-L210)
- [src/memu/workflow/pipeline.py](file://src/memu/workflow/pipeline.py#L21-L49)
- [src/memu/workflow/runner.py](file://src/memu/workflow/runner.py#L28-L39)
- [src/memu/workflow/step.py](file://src/memu/workflow/step.py#L16-L38)
- [src/memu/database/factory.py](file://src/memu/database/factory.py#L15-L43)
- [src/memu/database/interfaces.py](file://src/memu/database/interfaces.py#L12-L26)
- [src/memu/llm/backends/base.py](file://src/memu/llm/backends/base.py#L6-L31)
- [src/memu/embedding/backends/base.py](file://src/memu/embedding/backends/base.py#L6-L17)

**Section sources**
- [src/memu/app/service.py](file://src/memu/app/service.py#L49-L95)
- [src/memu/workflow/pipeline.py](file://src/memu/workflow/pipeline.py#L21-L49)
- [src/memu/database/factory.py](file://src/memu/database/factory.py#L15-L43)

## Core Components
This section introduces the foundational building blocks that underpin memU.

- Memory Layers
  - Resource: Represents a source artifact (e.g., a file or URL) with associated metadata and optional embeddings.
  - MemoryItem: Represents a piece of memory extracted from a Resource, with a memory type (profile, event, knowledge, behavior, skill, tool), summary, and optional embeddings.
  - MemoryCategory: Represents a semantic bucket grouping related MemoryItems, with optional summary and embeddings.

- Memory-as-File-System Paradigm
  - Resources are ingested from URLs or local paths and preprocessed into one or more logical segments.
  - Items are extracted from Resources and linked to Categories.
  - Categories summarize related Items and can reference specific Items for richer recall.

- Dual-Mode Retrieval
  - RAG mode: Uses vector similarity at each tier (Category → Item → Resource) with optional sufficiency checks.
  - LLM mode: Delegates ranking to the LLM at each tier, optionally using category references to narrow recall.

- Workflow Pipeline
  - Steps are composable units with explicit requires/produces sets and capability tags.
  - Pipelines are registered and can be mutated at runtime (insert/replace/remove steps, update configs).
  - Execution is delegated to a runner backend (local/sync supported; extensible).

- Provider Abstraction
  - LLMBackend defines payload construction and response parsing for chat/summary/vision.
  - EmbeddingBackend defines payload construction and response parsing for embeddings.
  - Clients are lazily initialized and cached per profile; wrappers add interception and metadata.

- Scope Management
  - User scope is modeled as a Pydantic model; filters are validated against user-defined fields.
  - Scoped models are built dynamically by merging user scope with core records.

**Section sources**
- [src/memu/database/models.py](file://src/memu/database/models.py#L68-L106)
- [src/memu/app/memorize.py](file://src/memu/app/memorize.py#L97-L166)
- [src/memu/app/retrieve.py](file://src/memu/app/retrieve.py#L106-L210)
- [src/memu/workflow/pipeline.py](file://src/memu/workflow/pipeline.py#L21-L49)
- [src/memu/workflow/step.py](file://src/memu/workflow/step.py#L16-L38)
- [src/memu/llm/backends/base.py](file://src/memu/llm/backends/base.py#L6-L31)
- [src/memu/embedding/backends/base.py](file://src/memu/embedding/backends/base.py#L6-L17)
- [src/memu/app/settings.py](file://src/memu/app/settings.py#L249-L258)

## Architecture Overview
The system orchestrates memory from ingestion to retrieval through a pipeline-driven architecture. MemoryService coordinates:
- LLM and embedding clients (HTTP/SKD/LazyLLM)
- Storage backends (in-memory, PostgreSQL with pgvector, SQLite)
- Workflow pipelines for memorize and retrieve
- Interceptors for observability and policy enforcement

```mermaid
sequenceDiagram
participant Client as "Caller"
participant Service as "MemoryService"
participant FS as "LocalFS"
participant LLM as "LLM Client"
participant DB as "Database"
participant Pipe as "PipelineManager"
participant Runner as "WorkflowRunner"
Client->>Service : memorize(resource_url, modality, user)
Service->>Pipe : register("memorize", steps)
Service->>Runner : run("memorize", initial_state)
Runner->>Service : step "ingest_resource"
Service->>FS : fetch(url, modality)
FS-->>Service : local_path, raw_text
Service->>Service : preprocess_multimodal(...)
Service->>LLM : chat(prompts)
LLM-->>Service : structured entries
Service->>Service : categorize_items(...)
Service->>DB : persist resources/items/relations
DB-->>Service : ids and vectors
Service-->>Client : response (resources, items, categories, relations)
```

**Diagram sources**
- [src/memu/app/service.py](file://src/memu/app/service.py#L315-L348)
- [src/memu/app/memorize.py](file://src/memu/app/memorize.py#L97-L166)
- [src/memu/workflow/runner.py](file://src/memu/workflow/runner.py#L28-L39)
- [src/memu/workflow/step.py](file://src/memu/workflow/step.py#L50-L101)

## Detailed Component Analysis

### Memory Layers and the Memory-as-File-System Paradigm
- Resource
  - Stores the source URL, modality, local path, optional caption, and optional embedding.
  - Used as the atomic unit for retrieval and indexing.
- MemoryItem
  - Captures extracted memory with a memory type, summary, and embedding.
  - Supports reinforcement tracking and metadata for specialized types (e.g., tools).
- MemoryCategory
  - Groups related items and supports optional summaries and embeddings.
  - Summaries can include inline references to items for precise recall.

```mermaid
classDiagram
class BaseRecord {
+string id
+datetime created_at
+datetime updated_at
}
class Resource {
+string url
+string modality
+string local_path
+string caption
+float[] embedding
}
class MemoryItem {
+string resource_id
+string memory_type
+string summary
+float[] embedding
+datetime happened_at
+dict extra
}
class MemoryCategory {
+string name
+string description
+float[] embedding
+string summary
}
class CategoryItem {
+string item_id
+string category_id
}
BaseRecord <|-- Resource
BaseRecord <|-- MemoryItem
BaseRecord <|-- MemoryCategory
BaseRecord <|-- CategoryItem
```

**Diagram sources**
- [src/memu/database/models.py](file://src/memu/database/models.py#L35-L106)

**Section sources**
- [src/memu/database/models.py](file://src/memu/database/models.py#L68-L106)

### Proactive Memory Lifecycle
The proactive lifecycle ties memory capture into ongoing workflows. It demonstrates how a continuous stream of conversation messages is periodically captured and persisted.

```mermaid
flowchart TD
Start(["Start"]) --> Collect["Collect conversation messages"]
Collect --> Threshold{"Reached threshold?"}
Threshold --> |No| LoopBack["Continue collecting"] --> Collect
Threshold --> |Yes| Trigger["Trigger background memorize task"]
Trigger --> Running{"Previous task running?"}
Running --> |Yes| Skip["Skip new task"] --> LoopBack
Running --> |No| Persist["Persist resources/items/relations"]
Persist --> Done(["Done"])
```

**Diagram sources**
- [examples/proactive/proactive.py](file://examples/proactive/proactive.py#L20-L123)

**Section sources**
- [examples/proactive/proactive.py](file://examples/proactive/proactive.py#L20-L123)

### Dual-Mode Retrieval System (RAG vs LLM-Based)
- RAG Mode
  - Routes intent, ranks categories by summary embeddings, checks sufficiency, then recalls items and resources via vector search, optionally rewriting queries between tiers.
- LLM Mode
  - Routes intent, asks the LLM to rank categories, then uses LLM to rank items (optionally using category references), then resources; sufficiency checks can also be LLM-driven.

```mermaid
sequenceDiagram
participant Client as "Caller"
participant Service as "MemoryService"
participant Pipe as "PipelineManager"
participant Runner as "WorkflowRunner"
participant LLM as "LLM Client"
participant DB as "Database"
Client->>Service : retrieve(queries, where)
alt method == "rag"
Service->>Pipe : register("retrieve_rag", steps)
else method == "llm"
Service->>Pipe : register("retrieve_llm", steps)
end
Service->>Runner : run(pipeline, initial_state)
Runner->>Service : route_intention(...)
Service->>LLM : decide sufficiency
LLM-->>Service : needs_retrieval, rewritten_query
Runner->>Service : route_category(...)
Service->>DB : list categories
Service->>LLM/DB : embed or vector search
Runner->>Service : recall_items(...)
Runner->>Service : recall_resources(...)
Runner->>Service : build_context(...)
Service-->>Client : response (categories, items, resources)
```

**Diagram sources**
- [src/memu/app/service.py](file://src/memu/app/service.py#L315-L348)
- [src/memu/app/retrieve.py](file://src/memu/app/retrieve.py#L106-L210)
- [src/memu/workflow/runner.py](file://src/memu/workflow/runner.py#L28-L39)

**Section sources**
- [src/memu/app/retrieve.py](file://src/memu/app/retrieve.py#L42-L85)
- [src/memu/app/retrieve.py](file://src/memu/app/retrieve.py#L106-L210)
- [src/memu/app/retrieve.py](file://src/memu/app/retrieve.py#L454-L536)

### Workflow Pipeline Architecture
- PipelineManager registers named pipelines with ordered steps and validates capabilities, required/produced state keys, and profile availability.
- WorkflowRunner executes steps sequentially, invoking before/after/on-error interceptors.
- Steps declare capabilities (e.g., io, llm, vector, db) enabling runtime routing and backend selection.

```mermaid
classDiagram
class PipelineManager {
+register(name, steps, initial_state_keys)
+build(name) WorkflowStep[]
+config_step(name, step_id, configs) int
+insert_after(name, target, new) int
+insert_before(name, target, new) int
+replace_step(name, target, new) int
+remove_step(name, target) int
}
class WorkflowStep {
+string step_id
+string role
+handler(state, context)
+set~string~ requires
+set~string~ produces
+set~string~ capabilities
+dict config
+copy() WorkflowStep
+run(state, context) WorkflowState
}
class WorkflowRunner {
<<interface>>
+string name
+run(name, steps, initial_state, context, registry) WorkflowState
}
PipelineManager --> WorkflowStep : "manages"
WorkflowRunner --> WorkflowStep : "executes"
```

**Diagram sources**
- [src/memu/workflow/pipeline.py](file://src/memu/workflow/pipeline.py#L21-L171)
- [src/memu/workflow/step.py](file://src/memu/workflow/step.py#L16-L48)
- [src/memu/workflow/runner.py](file://src/memu/workflow/runner.py#L12-L26)

**Section sources**
- [src/memu/workflow/pipeline.py](file://src/memu/workflow/pipeline.py#L21-L171)
- [src/memu/workflow/step.py](file://src/memu/workflow/step.py#L16-L48)
- [src/memu/workflow/runner.py](file://src/memu/workflow/runner.py#L12-L26)

### Provider Abstraction for LLMs and Embeddings
- LLMBackend defines standardized methods for building chat/summary/vision payloads and parsing responses.
- EmbeddingBackend defines standardized methods for building embedding payloads and parsing vectors.
- MemoryService lazily initializes clients per profile and wraps them for interception and metadata.

```mermaid
classDiagram
class LLMBackend {
+string name
+string summary_endpoint
+build_summary_payload(text, system_prompt, chat_model, max_tokens) dict
+parse_summary_response(data) string
+build_vision_payload(prompt, base64_image, mime_type, system_prompt, chat_model, max_tokens) dict
}
class EmbeddingBackend {
+string name
+string embedding_endpoint
+build_embedding_payload(inputs, embed_model) dict
+parse_embedding_response(data) List[]float~~
}
class MemoryService {
+_get_llm_client(profile, step_context)
+_get_step_llm_client(step_context)
+_get_step_embedding_client(step_context)
}
MemoryService --> LLMBackend : "uses via client wrapper"
MemoryService --> EmbeddingBackend : "uses via client wrapper"
```

**Diagram sources**
- [src/memu/llm/backends/base.py](file://src/memu/llm/backends/base.py#L6-L31)
- [src/memu/embedding/backends/base.py](file://src/memu/embedding/backends/base.py#L6-L17)
- [src/memu/app/service.py](file://src/memu/app/service.py#L97-L151)

**Section sources**
- [src/memu/llm/backends/base.py](file://src/memu/llm/backends/base.py#L6-L31)
- [src/memu/embedding/backends/base.py](file://src/memu/embedding/backends/base.py#L6-L17)
- [src/memu/app/service.py](file://src/memu/app/service.py#L97-L151)

### Scope Management for User Contexts
- UserConfig holds a Pydantic model type that defines the user scope schema.
- Filters passed to retrieval are validated against the user model fields to prevent invalid queries.
- Scoped models are dynamically constructed by merging user scope with core records.

```mermaid
flowchart TD
A["User scope dict"] --> B["user_model(**user).model_dump()"]
B --> C["Filter validation against user_model fields"]
C --> D["Normalized where filters"]
D --> E["Query execution with scope"]
```

**Diagram sources**
- [src/memu/app/retrieve.py](file://src/memu/app/retrieve.py#L87-L104)
- [src/memu/database/models.py](file://src/memu/database/models.py#L124-L134)

**Section sources**
- [src/memu/app/retrieve.py](file://src/memu/app/retrieve.py#L87-L104)
- [src/memu/database/models.py](file://src/memu/database/models.py#L124-L134)

### Relationship Between Storage Backends
- build_database selects the backend based on configuration:
  - inmemory: ephemeral, no persistence
  - postgres: persistent with optional pgvector
  - sqlite: lightweight, file-based
- The Database protocol exposes repositories for resources, categories, items, and relations, enabling a uniform API regardless of backend.

```mermaid
graph TB
Factory["build_database(config, user_model)"] --> InMem["In-Memory"]
Factory --> PG["Postgres"]
Factory --> SQL["SQLite"]
InMem --> IFace["Database Protocol"]
PG --> IFace
SQL --> IFace
```

**Diagram sources**
- [src/memu/database/factory.py](file://src/memu/database/factory.py#L15-L43)
- [src/memu/database/interfaces.py](file://src/memu/database/interfaces.py#L12-L26)

**Section sources**
- [src/memu/database/factory.py](file://src/memu/database/factory.py#L15-L43)
- [src/memu/database/interfaces.py](file://src/memu/database/interfaces.py#L12-L26)

### Memory Flow From Ingestion to Retrieval
- Ingestion (Memorize)
  - Fetch resource, preprocess multimodal content, extract structured entries, deduplicate/merge, categorize items, persist and index, summarize categories, and emit a response.
- Retrieval (RAG/LLM)
  - Route intent, optionally rewrite query, rank categories, check sufficiency, recall items and resources, and build a contextualized response.

```mermaid
sequenceDiagram
participant S as "MemoryService"
participant M as "MemorizeMixin"
participant R as "RetrieveMixin"
participant P as "PipelineManager"
participant U as "User"
U->>S : memorize(resource_url, modality, user)
S->>P : register("memorize", steps)
P-->>S : steps
S->>M : run workflow
M-->>S : response
U->>S : retrieve(queries, where)
S->>P : register("retrieve_rag"/"retrieve_llm", steps)
P-->>S : steps
S->>R : run workflow
R-->>S : response
```

**Diagram sources**
- [src/memu/app/memorize.py](file://src/memu/app/memorize.py#L65-L95)
- [src/memu/app/retrieve.py](file://src/memu/app/retrieve.py#L42-L85)
- [src/memu/app/service.py](file://src/memu/app/service.py#L315-L348)

**Section sources**
- [src/memu/app/memorize.py](file://src/memu/app/memorize.py#L65-L95)
- [src/memu/app/retrieve.py](file://src/memu/app/retrieve.py#L42-L85)
- [src/memu/app/service.py](file://src/memu/app/service.py#L315-L348)

## Dependency Analysis
- Coupling and Cohesion
  - MemoryService aggregates concerns (clients, pipelines, runners, databases) but delegates implementation to mixins and protocols, keeping cohesion high within each module.
- External Dependencies
  - LLM and embedding clients are pluggable; database backends are selected at runtime via a factory.
- Potential Circular Dependencies
  - None observed among the analyzed modules; protocol-based interfaces decouple consumers from implementations.

```mermaid
graph LR
Service["MemoryService"] --> MixMem["MemorizeMixin"]
Service --> MixRet["RetrieveMixin"]
Service --> Pipe["PipelineManager"]
Service --> Run["WorkflowRunner"]
Service --> DBF["build_database"]
DBF --> IFace["Database Protocol"]
IFace --> Impl1["In-Memory"]
IFace --> Impl2["Postgres"]
IFace --> Impl3["SQLite"]
Service --> LLM["LLM Backend Base"]
Service --> Emb["Embedding Backend Base"]
```

**Diagram sources**
- [src/memu/app/service.py](file://src/memu/app/service.py#L49-L95)
- [src/memu/database/factory.py](file://src/memu/database/factory.py#L15-L43)
- [src/memu/database/interfaces.py](file://src/memu/database/interfaces.py#L12-L26)
- [src/memu/llm/backends/base.py](file://src/memu/llm/backends/base.py#L6-L31)
- [src/memu/embedding/backends/base.py](file://src/memu/embedding/backends/base.py#L6-L17)

**Section sources**
- [src/memu/app/service.py](file://src/memu/app/service.py#L49-L95)
- [src/memu/database/factory.py](file://src/memu/database/factory.py#L15-L43)
- [src/memu/database/interfaces.py](file://src/memu/database/interfaces.py#L12-L26)

## Performance Considerations
- Vector Search Efficiency
  - Use appropriate top_k per tier to balance recall and latency.
  - Prefer pgvector for production-grade vector search when using PostgreSQL.
- Embedding Batch Size
  - Adjust embed_batch_size for SDK-based clients to optimize throughput.
- Client Caching
  - LLM clients are cached per profile to avoid repeated initialization overhead.
- Interceptors and Workflows
  - Keep interceptors lightweight; heavy logic should be deferred to dedicated steps.

## Troubleshooting Guide
- Unknown filter field for user scope
  - Ensure the where clause keys match the user model fields; otherwise, a validation error is raised.
- Missing required keys in workflow state
  - Verify that prior steps produced the required keys declared by subsequent steps.
- Unknown workflow runner or profile
  - Register custom runners via the provided registration mechanism and ensure profile names exist.
- Unsupported metadata_store provider
  - Only inmemory, postgres, and sqlite are supported; ensure configuration matches one of these.

**Section sources**
- [src/memu/app/retrieve.py](file://src/memu/app/retrieve.py#L87-L104)
- [src/memu/workflow/pipeline.py](file://src/memu/workflow/pipeline.py#L131-L164)
- [src/memu/workflow/runner.py](file://src/memu/workflow/runner.py#L61-L81)
- [src/memu/database/factory.py](file://src/memu/database/factory.py#L42-L43)

## Conclusion
memU’s architecture centers on three memory layers—Resource, Item, and Category—organized as a memory-as-file-system paradigm. The dual-mode retrieval system (RAG and LLM-based) provides flexibility in how relevance is determined, while the workflow pipeline offers composability and runtime configurability. Provider abstractions for LLMs and embeddings, combined with pluggable storage backends, enable extensibility. Scope management ensures safe and predictable filtering of memory across user contexts. Together, these design choices deliver a robust foundation for building proactive, context-aware memory systems.