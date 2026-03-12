# Architecture Decision Records

<cite>
**Referenced Files in This Document**
- [0001-workflow-pipeline-architecture.md](file://docs/adr/0001-workflow-pipeline-architecture.md)
- [0002-pluggable-storage-and-vector-strategy.md](file://docs/adr/0002-pluggable-storage-and-vector-strategy.md)
- [0003-user-scope-in-data-model.md](file://docs/adr/0003-user-scope-in-data-model.md)
- [README.md](file://docs/adr/README.md)
- [architecture.md](file://docs/architecture.md)
- [pipeline.py](file://src/memu/workflow/pipeline.py)
- [runner.py](file://src/memu/workflow/runner.py)
- [step.py](file://src/memu/workflow/step.py)
- [interceptor.py](file://src/memu/workflow/interceptor.py)
- [factory.py](file://src/memu/database/factory.py)
- [interfaces.py](file://src/memu/database/interfaces.py)
- [models.py](file://src/memu/database/models.py)
- [memory_item.py](file://src/memu/database/repositories/memory_item.py)
- [memory_category.py](file://src/memu/database/repositories/memory_category.py)
- [category_item.py](file://src/memu/database/repositories/category_item.py)
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
10. [Appendices](#appendices)

## Introduction
This document presents the Architecture Decision Records (ADRs) that define memU’s foundational design choices. It explains the rationale, evaluation criteria, and trade-offs for:
- Workflow pipeline architecture for core operations
- Pluggable storage and vector strategy
- User scope modeling embedded in data models

It also documents how these decisions influence system behavior, extensibility, and maintenance, and provides guidance for evolving the architecture over time.

## Project Structure
The ADRs are organized under docs/adr and complement the architecture overview in docs/architecture.md. The implementation spans workflow orchestration, database abstraction, and data models.

```mermaid
graph TB
A["docs/adr/README.md"] --> B["docs/adr/0001-..."]
A --> C["docs/adr/0002-..."]
A --> D["docs/adr/0003-..."]
E["docs/architecture.md"] --> F["src/memu/workflow/*"]
E --> G["src/memu/database/*"]
E --> H["src/memu/app/service.py"]
```

**Diagram sources**
- [README.md](file://docs/adr/README.md#L1-L6)
- [architecture.md](file://docs/architecture.md#L1-L170)

**Section sources**
- [README.md](file://docs/adr/README.md#L1-L6)
- [architecture.md](file://docs/architecture.md#L1-L170)

## Core Components
- Workflow engine: pipelines, steps, runners, and interceptors enable staged execution with observability and customization.
- Database abstraction: a protocol-backed repository layer supports pluggable backends with backend-aware vector behavior.
- Data models: scope fields are merged into core records to enforce consistent scoping across APIs.

**Section sources**
- [architecture.md](file://docs/architecture.md#L32-L170)

## Architecture Overview
The system orchestrates ingestion and retrieval through named pipelines, backed by a repository abstraction and LLM clients. The architecture supports local-first development and production-grade vector search.

```mermaid
flowchart TD
A["Input Resource or Query"] --> B["MemoryService"]
B --> C["Workflow Pipelines"]
C --> D["LLM Clients"]
C --> E["Database Repositories"]
E --> F["Resources"]
E --> G["Memory Items"]
E --> H["Memory Categories"]
E --> I["Category Relations"]
```

**Diagram sources**
- [architecture.md](file://docs/architecture.md#L20-L30)

**Section sources**
- [architecture.md](file://docs/architecture.md#L9-L170)

## Detailed Component Analysis

### ADR 0001: Use Workflow Pipelines for Core Operations
- Decision: Model operations as named pipelines of ordered steps with explicit state contracts and capability tags.
- Evaluation criteria: Extensibility, observability, runtime customization, and uniformity across memorize/retrieve/CRUD.
- Alternatives considered: Monolithic functions per operation (rejected due to reduced extensibility and observability).
- Trade-offs:
  - Positive: Uniform execution model, explicit stage boundaries, extension points, interception and observability.
  - Negative: Dict-based state relies on key discipline, pipeline mutation can vary behavior across deployments, more framework code than direct calls.
- Implementation highlights:
  - PipelineManager registers pipelines, validates dependencies, and supports runtime mutation (insert/replace/remove).
  - WorkflowRunner is a protocol with a default LocalWorkflowRunner.
  - Interceptors provide before/after/on_error hooks for instrumentation and control.

```mermaid
classDiagram
class PipelineManager {
+register(name, steps, initial_state_keys)
+build(name)
+config_step(name, step_id, configs)
+insert_after(name, target, step)
+insert_before(name, target, step)
+replace_step(name, target, step)
+remove_step(name, target)
+revision_token()
}
class WorkflowStep {
+step_id : str
+role : str
+requires : set[str]
+produces : set[str]
+capabilities : set[str]
+config : dict
+run(state, context)
+copy()
}
class WorkflowRunner {
<<protocol>>
+name : str
+run(workflow_name, steps, initial_state, context, interceptors)
}
class LocalWorkflowRunner {
+name : "local"
+run(...)
}
PipelineManager --> WorkflowStep : "manages"
WorkflowRunner <|.. LocalWorkflowRunner : "implements"
```

**Diagram sources**
- [pipeline.py](file://src/memu/workflow/pipeline.py#L21-L171)
- [step.py](file://src/memu/workflow/step.py#L16-L102)
- [runner.py](file://src/memu/workflow/runner.py#L12-L82)

**Section sources**
- [0001-workflow-pipeline-architecture.md](file://docs/adr/0001-workflow-pipeline-architecture.md#L1-L36)
- [pipeline.py](file://src/memu/workflow/pipeline.py#L21-L171)
- [step.py](file://src/memu/workflow/step.py#L16-L102)
- [runner.py](file://src/memu/workflow/runner.py#L12-L82)
- [interceptor.py](file://src/memu/workflow/interceptor.py#L56-L219)

### ADR 0002: Use Pluggable Storage with Backend-Specific Vector Search
- Decision: Adopt a repository-based abstraction behind a Database protocol with selectable providers: inmemory, sqlite, postgres.
- Evaluation criteria: Zero-setup local development, lightweight persistence, and scalable vector similarity in production.
- Alternatives considered: Single monolithic storage engine (rejected due to mismatched needs across local and production).
- Trade-offs:
  - Positive: One service API across environments, clear backend contracts, predictable fallback behavior.
  - Negative: Duplicate repository logic across backends, performance differences, brute-force search on SQLite/inmemory.
- Implementation highlights:
  - Factory builds provider-specific databases.
  - Vector behavior is backend-aware: brute-force cosine search for portability; pgvector distance queries when enabled.

```mermaid
classDiagram
class Database {
<<protocol>>
+resource_repo
+memory_category_repo
+memory_item_repo
+category_item_repo
+resources
+items
+categories
+relations
+close()
}
class ResourceRepo
class MemoryItemRepo
class MemoryCategoryRepo
class CategoryItemRepo
class DatabaseFactory {
+build_database(config, user_model) Database
}
Database <|.. DatabaseFactory : "builds"
Database --> ResourceRepo
Database --> MemoryItemRepo
Database --> MemoryCategoryRepo
Database --> CategoryItemRepo
```

**Diagram sources**
- [interfaces.py](file://src/memu/database/interfaces.py#L12-L36)
- [factory.py](file://src/memu/database/factory.py#L15-L44)
- [memory_item.py](file://src/memu/database/repositories/memory_item.py#L9-L55)
- [memory_category.py](file://src/memu/database/repositories/memory_category.py#L9-L34)
- [category_item.py](file://src/memu/database/repositories/category_item.py#L9-L24)

**Section sources**
- [0002-pluggable-storage-and-vector-strategy.md](file://docs/adr/0002-pluggable-storage-and-vector-strategy.md#L1-L43)
- [factory.py](file://src/memu/database/factory.py#L15-L44)
- [interfaces.py](file://src/memu/database/interfaces.py#L12-L36)
- [architecture.md](file://docs/architecture.md#L111-L170)

### ADR 0003: Model User Scope as First-Class Fields on Memory Records
- Decision: Embed scope directly into persisted entities by merging a configurable UserConfig model with core record models.
- Evaluation criteria: Consistent filtering across APIs, backend independence, and support for multi-tenant and multi-agent patterns.
- Alternatives considered: Keeping scope external (rejected due to ad-hoc filtering and weakened isolation).
- Trade-offs:
  - Positive: Consistent filtering model, backend-independent semantics, multi-tenant/multi-agent support.
  - Negative: Increased schema/model complexity, varying shapes by scope model, caller alignment requirements.
- Implementation highlights:
  - Scope fields are part of resource/category/item/relation models.
  - Repositories accept user_data on writes and where filters on reads.
  - API-level where filters are validated against configured scope fields.

```mermaid
classDiagram
class BaseRecord {
+id : str
+created_at : datetime
+updated_at : datetime
}
class Resource {
+url : str
+modality : str
+local_path : str
+caption : str
+embedding : float[]
}
class MemoryItem {
+resource_id : str
+memory_type : str
+summary : str
+embedding : float[]
+happened_at : datetime
+extra : dict
}
class MemoryCategory {
+name : str
+description : str
+embedding : float[]
+summary : str
}
class CategoryItem {
+item_id : str
+category_id : str
}
class ScopedModels {
+build_scoped_models(user_model)
+merge_scope_model(user_model, core_model, name_suffix)
}
BaseRecord <|-- Resource
BaseRecord <|-- MemoryItem
BaseRecord <|-- MemoryCategory
BaseRecord <|-- CategoryItem
ScopedModels --> Resource
ScopedModels --> MemoryItem
ScopedModels --> MemoryCategory
ScopedModels --> CategoryItem
```

**Diagram sources**
- [models.py](file://src/memu/database/models.py#L35-L149)

**Section sources**
- [0003-user-scope-in-data-model.md](file://docs/adr/0003-user-scope-in-data-model.md#L1-L33)
- [models.py](file://src/memu/database/models.py#L108-L149)
- [architecture.md](file://docs/architecture.md#L132-L137)

## Dependency Analysis
The workflow and database layers are decoupled from concrete implementations via protocols and factories, enabling runtime selection and extension.

```mermaid
graph TB
WF_Pipeline["src/memu/workflow/pipeline.py"] --> WF_Step["src/memu/workflow/step.py"]
WF_Runner["src/memu/workflow/runner.py"] --> WF_Step
WF_Interceptor["src/memu/workflow/interceptor.py"] --> WF_Runner
DB_Factory["src/memu/database/factory.py"] --> DB_Interfaces["src/memu/database/interfaces.py"]
DB_Factory --> Repo_Item["src/memu/database/repositories/memory_item.py"]
DB_Factory --> Repo_Category["src/memu/database/repositories/memory_category.py"]
DB_Factory --> Repo_Relation["src/memu/database/repositories/category_item.py"]
DB_Models["src/memu/database/models.py"] --> Repo_Item
DB_Models --> Repo_Category
DB_Models --> Repo_Relation
```

**Diagram sources**
- [pipeline.py](file://src/memu/workflow/pipeline.py#L1-L171)
- [step.py](file://src/memu/workflow/step.py#L1-L102)
- [runner.py](file://src/memu/workflow/runner.py#L1-L82)
- [interceptor.py](file://src/memu/workflow/interceptor.py#L1-L219)
- [factory.py](file://src/memu/database/factory.py#L1-L44)
- [interfaces.py](file://src/memu/database/interfaces.py#L1-L36)
- [memory_item.py](file://src/memu/database/repositories/memory_item.py#L1-L55)
- [memory_category.py](file://src/memu/database/repositories/memory_category.py#L1-L34)
- [category_item.py](file://src/memu/database/repositories/category_item.py#L1-L24)
- [models.py](file://src/memu/database/models.py#L1-L149)

**Section sources**
- [pipeline.py](file://src/memu/workflow/pipeline.py#L21-L171)
- [runner.py](file://src/memu/workflow/runner.py#L46-L82)
- [factory.py](file://src/memu/database/factory.py#L15-L44)
- [interfaces.py](file://src/memu/database/interfaces.py#L12-L36)

## Performance Considerations
- Workflow state is dict-based, validated by key names rather than static types; ensure disciplined naming and schema hygiene.
- SQLite and inmemory vector search rely on brute-force cosine similarity; expect lower scalability compared to pgvector-enabled Postgres.
- Category and extraction quality depend on prompts and LLMs; consider prompt engineering and quality gates.
- Some extension hooks are placeholders (e.g., dedupe/merge stage); treat as temporary and revisit during refinement.

**Section sources**
- [architecture.md](file://docs/architecture.md#L158-L164)

## Troubleshooting Guide
- Pipeline errors:
  - Missing required state keys or unknown capabilities/validation failures occur during registration/mutation; review step contracts and capability tags.
  - Use revision tokens to track pipeline changes across deployments.
- Runner resolution:
  - Unknown runner names or factories returning non-compliant runners cause errors; register via the runner factory and ensure protocol compliance.
- Interceptors:
  - Strict mode propagates interceptor exceptions; otherwise failures are logged. Verify interceptor signatures and error handling.
- Database provider selection:
  - Unsupported provider or misconfiguration raises errors; confirm provider string and backend availability.

**Section sources**
- [pipeline.py](file://src/memu/workflow/pipeline.py#L131-L171)
- [runner.py](file://src/memu/workflow/runner.py#L61-L82)
- [interceptor.py](file://src/memu/workflow/interceptor.py#L163-L219)
- [factory.py](file://src/memu/database/factory.py#L15-L44)

## Conclusion
These ADRs establish a flexible, observable, and extensible foundation for memU:
- Workflows provide uniform, customizable execution with strong observability.
- Storage abstraction enables seamless transitions from local to production-grade vector search.
- User scope embedded in models ensures consistent isolation and multi-tenant/multi-agent support.

Future evolution should focus on reducing duplication across backends, refining placeholder extension hooks, and enhancing schema stability while preserving backward compatibility.

## Appendices
- Related ADRs and architecture overview are linked below for cross-reference.

**Section sources**
- [README.md](file://docs/adr/README.md#L1-L6)
- [architecture.md](file://docs/architecture.md#L165-L170)