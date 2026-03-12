# Database Architecture

<cite>
**Referenced Files in This Document**
- [factory.py](file://src/memu/database/factory.py)
- [__init__.py](file://src/memu/database/__init__.py)
- [interfaces.py](file://src/memu/database/interfaces.py)
- [models.py](file://src/memu/models.py)
- [state.py](file://src/memu/database/state.py)
- [repositories/__init__.py](file://src/memu/database/repositories/__init__.py)
- [memory_item.py](file://src/memu/database/repositories/memory_item.py)
- [memory_category.py](file://src/memu/database/repositories/memory_category.py)
- [resource.py](file://src/memu/database/repositories/resource.py)
- [category_item.py](file://src/memu/database/repositories/category_item.py)
- [inmemory/__init__.py](file://src/memu/database/inmemory/__init__.py)
- [inmemory/models.py](file://src/memu/database/inmemory/models.py)
- [inmemory/repo.py](file://src/memu/database/inmemory/repo.py)
- [inmemory/vector.py](file://src/memu/database/inmemory/vector.py)
- [sqlite/__init__.py](file://src/memu/database/sqlite/__init__.py)
- [sqlite/models.py](file://src/memu/database/sqlite/models.py)
- [sqlite/schema.py](file://src/memu/database/sqlite/schema.py)
- [sqlite/session.py](file://src/memu/database/sqlite/session.py)
- [sqlite/sqlite.py](file://src/memu/database/sqlite/sqlite.py)
- [postgres/__init__.py](file://src/memu/database/postgres/__init__.py)
- [postgres/models.py](file://src/memu/database/postgres/models.py)
- [postgres/schema.py](file://src/memu/database/postgres/schema.py)
- [postgres/session.py](file://src/memu/database/postgres/session.py)
- [postgres/postgres.py](file://src/memu/database/postgres/postgres.py)
- [postgres/migration.py](file://src/memu/database/postgres/migration.py)
- [postgres/migrations/env.py](file://src/memu/database/postgres/migrations/env.py)
- [settings.py](file://src/memu/app/settings.py)
- [crud.py](file://src/memu/app/crud.py)
- [retrieve.py](file://src/memu/app/retrieve.py)
- [service.py](file://src/memu/app/service.py)
- [workflow/pipeline.py](file://src/memu/workflow/pipeline.py)
- [workflow/runner.py](file://src/memu/workflow/runner.py)
- [workflow/step.py](file://src/memu/workflow/step.py)
- [workflow/interceptor.py](file://src/memu/workflow/interceptor.py)
- [prompts/retrieve/query_rewriter.py](file://src/memu/prompts/retrieve/query_rewriter.py)
- [prompts/retrieve/llm_category_ranker.py](file://src/memu/prompts/retrieve/llm_category_ranker.py)
- [prompts/retrieve/llm_item_ranker.py](file://src/memu/prompts/retrieve/llm_item_ranker.py)
- [prompts/retrieve/llm_resource_ranker.py](file://src/memu/prompts/retrieve/llm_resource_ranker.py)
- [prompts/retrieve/judger.py](file://src/memu/prompts/retrieve/judger.py)
- [embedding/backends/openai.py](file://src/memu/embedding/backends/openai.py)
- [embedding/backends/doubao.py](file://src/memu/embedding/backends/doubao.py)
- [llm/backends/openai.py](file://src/memu/llm/backends/openai.py)
- [llm/backends/grok.py](file://src/memu/llm/backends/grok.py)
- [llm/backends/openrouter.py](file://src/memu/llm/backends/openrouter.py)
- [llm/backends/doubao.py](file://src/memu/llm/backends/doubao.py)
- [test_inmemory.py](file://tests/test_inmemory.py)
- [test_sqlite.py](file://tests/test_sqlite.py)
- [test_postgres.py](file://tests/test_postgres.py)
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
This document describes the database layer architecture of memU with a focus on the pluggable storage backend system. It explains the factory pattern for backend selection, repository contracts, ORM-like models, and vector search utilities. It documents three storage backends:
- In-memory for development and testing
- SQLite for lightweight deployments
- PostgreSQL with optional pgvector extension for production

It also covers repository pattern implementation, migration strategies, infrastructure requirements, scalability considerations, deployment topology, and cross-cutting concerns such as data validation, transactions, and backups. Finally, it illustrates system context diagrams showing how the database layer integrates with the workflow engine and LLM providers.

## Project Structure
The database subsystem is organized around a backend-agnostic interface and a factory that selects a concrete implementation at runtime. The repository pattern isolates data access logic, while models define the persisted entities. Vector search utilities are provided for in-memory implementations.

```mermaid
graph TB
subgraph "Database Layer"
F["factory.py<br/>build_database()"]
IF["interfaces.py<br/>Database Protocol"]
ST["state.py<br/>DatabaseState"]
RE["repositories/*<br/>Contracts"]
MD["models.py<br/>Pydantic Records"]
IM["inmemory/*<br/>InMemoryStore"]
SQ["sqlite/*<br/>SQLite Backend"]
PG["postgres/*<br/>PostgreSQL Backend"]
end
subgraph "App Layer"
APP["app/crud.py<br/>CRUD"]
SRV["app/service.py<br/>Service"]
RET["app/retrieve.py<br/>Retrieve"]
end
subgraph "Workflow Engine"
WF["workflow/pipeline.py"]
RUN["workflow/runner.py"]
STEP["workflow/step.py"]
INT["workflow/interceptor.py"]
end
subgraph "LLM Providers"
EMB["embedding/backends/*"]
LLM["llm/backends/*"]
end
F --> IF
F --> IM
F --> SQ
F --> PG
IF --> RE
RE --> MD
IM --> ST
SQ --> ST
PG --> ST
APP --> F
SRV --> F
RET --> F
WF --> APP
EMB --> SRV
LLM --> SRV
```

**Diagram sources**
- [factory.py](file://src/memu/database/factory.py#L15-L44)
- [interfaces.py](file://src/memu/database/interfaces.py#L12-L27)
- [state.py](file://src/memu/database/state.py#L8-L14)
- [repositories/__init__.py](file://src/memu/database/repositories/__init__.py#L1-L7)
- [models.py](file://src/memu/database/models.py#L35-L148)
- [inmemory/repo.py](file://src/memu/database/inmemory/repo.py#L20-L61)
- [sqlite/__init__.py](file://src/memu/database/sqlite/__init__.py#L1-L26)
- [postgres/__init__.py](file://src/memu/database/postgres/__init__.py#L1-L26)
- [app/crud.py](file://src/memu/app/crud.py#L1-L200)
- [app/service.py](file://src/memu/app/service.py#L1-L200)
- [app/retrieve.py](file://src/memu/app/retrieve.py#L1-L200)
- [workflow/pipeline.py](file://src/memu/workflow/pipeline.py#L1-L200)
- [workflow/runner.py](file://src/memu/workflow/runner.py#L1-L200)
- [workflow/step.py](file://src/memu/workflow/step.py#L1-L200)
- [workflow/interceptor.py](file://src/memu/workflow/interceptor.py#L1-L200)
- [embedding/backends/openai.py](file://src/memu/embedding/backends/openai.py#L1-L200)
- [embedding/backends/doubao.py](file://src/memu/embedding/backends/doubao.py#L1-L200)
- [llm/backends/openai.py](file://src/memu/llm/backends/openai.py#L1-L200)
- [llm/backends/grok.py](file://src/memu/llm/backends/grok.py#L1-L200)
- [llm/backends/openrouter.py](file://src/memu/llm/backends/openrouter.py#L1-L200)
- [llm/backends/doubao.py](file://src/memu/llm/backends/doubao.py#L1-L200)

**Section sources**
- [factory.py](file://src/memu/database/factory.py#L1-L44)
- [__init__.py](file://src/memu/database/__init__.py#L1-L29)

## Core Components
- Factory pattern: The build_database function selects a backend based on configuration and returns a Database protocol-compliant object.
- Database Protocol: Defines repositories and in-memory caches for resources, items, categories, and relations.
- Repository Contracts: Typed protocols for CRUD and vector search operations.
- ORM-like Models: Pydantic models representing Resource, MemoryItem, MemoryCategory, and CategoryItem with computed hashes and timestamps.
- State Management: In-memory state container shared across repositories for in-memory backend.
- Vector Utilities: Cosine similarity and salience-aware scoring for retrieval ranking.

Key implementation references:
- Factory: [build_database](file://src/memu/database/factory.py#L15-L44)
- Database Protocol: [Database](file://src/memu/database/interfaces.py#L12-L27)
- Repositories: [ResourceRepo](file://src/memu/database/repositories/resource.py#L9-L31), [MemoryCategoryRepo](file://src/memu/database/repositories/memory_category.py#L9-L34), [MemoryItemRepo](file://src/memu/database/repositories/memory_item.py#L9-L55), [CategoryItemRepo](file://src/memu/database/repositories/category_item.py#L9-L24)
- Models: [Resource](file://src/memu/database/models.py#L68-L74), [MemoryItem](file://src/memu/database/models.py#L76-L94), [MemoryCategory](file://src/memu/database/models.py#L96-L101), [CategoryItem](file://src/memu/database/models.py#L103-L106), [compute_content_hash](file://src/memu/database/models.py#L15-L32)
- State: [DatabaseState](file://src/memu/database/state.py#L8-L14)
- Vector utilities: [cosine_topk](file://src/memu/database/inmemory/vector.py#L56-L92), [salience_score](file://src/memu/database/inmemory/vector.py#L16-L53)

**Section sources**
- [factory.py](file://src/memu/database/factory.py#L15-L44)
- [interfaces.py](file://src/memu/database/interfaces.py#L12-L27)
- [repositories/__init__.py](file://src/memu/database/repositories/__init__.py#L1-L7)
- [models.py](file://src/memu/database/models.py#L15-L148)
- [state.py](file://src/memu/database/state.py#L8-L14)
- [inmemory/vector.py](file://src/memu/database/inmemory/vector.py#L16-L138)

## Architecture Overview
The database layer follows a layered architecture:
- Application layer (app/*): orchestrates CRUD and retrieval operations.
- Database abstraction (database/*): provides a unified interface via the Database protocol and repository contracts.
- Backend implementations: inmemory, sqlite, postgres.
- Workflow engine: invokes app services that depend on the database abstraction.
- LLM providers: supply embeddings and generation results used by the app layer.

```mermaid
graph TB
subgraph "Application"
CRUD["app/crud.py"]
SVC["app/service.py"]
RET["app/retrieve.py"]
end
subgraph "Database Abstraction"
DBP["interfaces.py<br/>Database Protocol"]
REPOS["repositories/*<br/>Contracts"]
MODELS["models.py<br/>Records"]
end
subgraph "Backends"
IMPL["inmemory/repo.py<br/>InMemoryStore"]
SQL["sqlite/*"]
PGSQL["postgres/*"]
end
subgraph "Workflow"
PIPE["workflow/pipeline.py"]
RUNR["workflow/runner.py"]
STEP["workflow/step.py"]
INTX["workflow/interceptor.py"]
end
subgraph "LLM Providers"
EMB["embedding/backends/*"]
LLM["llm/backends/*"]
end
CRUD --> SVC
SVC --> DBP
DBP --> REPOS
REPOS --> MODELS
DBP --> IMPL
DBP --> SQL
DBP --> PGSQL
PIPE --> CRUD
RUNR --> PIPE
STEP --> RUNR
INTX --> STEP
EMB --> SVC
LLM --> SVC
```

**Diagram sources**
- [app/crud.py](file://src/memu/app/crud.py#L1-L200)
- [app/service.py](file://src/memu/app/service.py#L1-L200)
- [app/retrieve.py](file://src/memu/app/retrieve.py#L1-L200)
- [interfaces.py](file://src/memu/database/interfaces.py#L12-L27)
- [repositories/__init__.py](file://src/memu/database/repositories/__init__.py#L1-L7)
- [models.py](file://src/memu/database/models.py#L35-L148)
- [inmemory/repo.py](file://src/memu/database/inmemory/repo.py#L20-L61)
- [sqlite/__init__.py](file://src/memu/database/sqlite/__init__.py#L1-L26)
- [postgres/__init__.py](file://src/memu/database/postgres/__init__.py#L1-L26)
- [workflow/pipeline.py](file://src/memu/workflow/pipeline.py#L1-L200)
- [workflow/runner.py](file://src/memu/workflow/runner.py#L1-L200)
- [workflow/step.py](file://src/memu/workflow/step.py#L1-L200)
- [workflow/interceptor.py](file://src/memu/workflow/interceptor.py#L1-L200)
- [embedding/backends/openai.py](file://src/memu/embedding/backends/openai.py#L1-L200)
- [embedding/backends/doubao.py](file://src/memu/embedding/backends/doubao.py#L1-L200)
- [llm/backends/openai.py](file://src/memu/llm/backends/openai.py#L1-L200)
- [llm/backends/grok.py](file://src/memu/llm/backends/grok.py#L1-L200)
- [llm/backends/openrouter.py](file://src/memu/llm/backends/openrouter.py#L1-L200)
- [llm/backends/doubao.py](file://src/memu/llm/backends/doubao.py#L1-L200)

## Detailed Component Analysis

### Factory Pattern and Backend Selection
The factory builds a Database implementation based on configuration. It supports:
- inmemory: No persistence, suitable for development and tests.
- postgres: Production-grade with optional pgvector.
- sqlite: Lightweight, file-based storage.

```mermaid
flowchart TD
Start(["build_database(config, user_model)"]) --> ReadProvider["Read provider from config"]
ReadProvider --> CheckIM{"Provider == 'inmemory'?"}
CheckIM --> |Yes| BuildIM["Import inmemory and build InMemoryStore"]
CheckIM --> |No| CheckPG{"Provider == 'postgres'?"}
CheckPG --> |Yes| ImportPG["Lazy import postgres backend"] --> BuildPG["Build PostgreSQL database"]
CheckPG --> |No| CheckSQ{"Provider == 'sqlite'?"}
CheckSQ --> |Yes| ImportSQ["Lazy import sqlite backend"] --> BuildSQ["Build SQLite database"]
CheckSQ --> |No| Error["Raise ValueError: unsupported provider"]
BuildIM --> Return(["Return Database"])
BuildPG --> Return
BuildSQ --> Return
Error --> End(["Exit"])
Return --> End
```

**Diagram sources**
- [factory.py](file://src/memu/database/factory.py#L15-L44)

**Section sources**
- [factory.py](file://src/memu/database/factory.py#L15-L44)

### Database Protocol and Repository Contracts
The Database protocol exposes typed repositories and in-memory caches. Repository contracts define CRUD and vector search operations.

```mermaid
classDiagram
class Database {
+resource_repo
+memory_category_repo
+memory_item_repo
+category_item_repo
+resources : dict
+items : dict
+categories : dict
+relations : list
+close()
}
class ResourceRepo {
+list_resources(where) dict
+clear_resources(where) dict
+create_resource(...)
+load_existing()
}
class MemoryCategoryRepo {
+list_categories(where) dict
+clear_categories(where) dict
+get_or_create_category(...)
+update_category(...)
+load_existing()
}
class MemoryItemRepo {
+get_item(id) MemoryItem
+list_items(where) dict
+clear_items(where) dict
+create_item(...)
+update_item(...)
+delete_item(id)
+list_items_by_ref_ids(ref_ids, where) dict
+vector_search_items(query_vec, top_k, where) list
+load_existing()
}
class CategoryItemRepo {
+list_relations(where) list
+link_item_category(item_id, cat_id, user_data)
+unlink_item_category(item_id, cat_id)
+get_item_categories(item_id) list
+load_existing()
}
Database --> ResourceRepo
Database --> MemoryCategoryRepo
Database --> MemoryItemRepo
Database --> CategoryItemRepo
```

**Diagram sources**
- [interfaces.py](file://src/memu/database/interfaces.py#L12-L27)
- [repositories/resource.py](file://src/memu/database/repositories/resource.py#L9-L31)
- [repositories/memory_category.py](file://src/memu/database/repositories/memory_category.py#L9-L34)
- [repositories/memory_item.py](file://src/memu/database/repositories/memory_item.py#L9-L55)
- [repositories/category_item.py](file://src/memu/database/repositories/category_item.py#L9-L24)

**Section sources**
- [interfaces.py](file://src/memu/database/interfaces.py#L12-L27)
- [repositories/resource.py](file://src/memu/database/repositories/resource.py#L9-L31)
- [repositories/memory_category.py](file://src/memu/database/repositories/memory_category.py#L9-L34)
- [repositories/memory_item.py](file://src/memu/database/repositories/memory_item.py#L9-L55)
- [repositories/category_item.py](file://src/memu/database/repositories/category_item.py#L9-L24)

### ORM Models and Scoped Models
Models define the persisted entities and include helper functions for deduplication and scoping. The scoped model mechanism merges user-defined scope models with core records.

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
+caption : str?
+embedding : list[float]?
}
class MemoryItem {
+resource_id : str?
+memory_type : str
+summary : str
+embedding : list[float]?
+happened_at : datetime?
+extra : dict
}
class MemoryCategory {
+name : str
+description : str
+embedding : list[float]?
+summary : str?
}
class CategoryItem {
+item_id : str
+category_id : str
}
class ToolCallResult {
+tool_name : str
+input : dict|str
+output : str
+success : bool
+time_cost : float
+token_cost : int
+score : float
+call_hash : str
+generate_hash() str
+ensure_hash() void
}
BaseRecord <|-- Resource
BaseRecord <|-- MemoryItem
BaseRecord <|-- MemoryCategory
BaseRecord <|-- CategoryItem
```

**Diagram sources**
- [models.py](file://src/memu/database/models.py#L35-L148)

**Section sources**
- [models.py](file://src/memu/database/models.py#L15-L148)

### In-Memory Backend
The in-memory backend provides a fast, non-persistent store suitable for development and testing. It uses a shared state container and repository implementations that operate on in-memory collections.

```mermaid
sequenceDiagram
participant Cfg as "Settings.DatabaseConfig"
participant Fac as "factory.build_database"
participant IM as "inmemory.build_inmemory_database"
participant Store as "InMemoryStore"
participant Repo as "Repositories"
Cfg->>Fac : provider="inmemory"
Fac->>IM : build_inmemory_database(config, user_model)
IM->>Store : construct with scoped models and state
Store->>Repo : initialize repositories bound to state
Fac-->>Client : Database (InMemoryStore)
```

**Diagram sources**
- [factory.py](file://src/memu/database/factory.py#L28-L31)
- [inmemory/__init__.py](file://src/memu/database/inmemory/__init__.py#L10-L22)
- [inmemory/repo.py](file://src/memu/database/inmemory/repo.py#L20-L61)

**Section sources**
- [inmemory/__init__.py](file://src/memu/database/inmemory/__init__.py#L10-L22)
- [inmemory/repo.py](file://src/memu/database/inmemory/repo.py#L20-L61)
- [inmemory/models.py](file://src/memu/database/inmemory/models.py#L30-L45)
- [inmemory/vector.py](file://src/memu/database/inmemory/vector.py#L56-L138)

### SQLite Backend
The SQLite backend provides a lightweight, file-based persistence layer. It defines models, schema, session management, and repository implementations.

```mermaid
graph TB
subgraph "SQLite Backend"
SM["sqlite/models.py"]
SS["sqlite/schema.py"]
SESS["sqlite/session.py"]
REPO["sqlite/repositories/*"]
INIT["sqlite/__init__.py"]
end
INIT --> SM
INIT --> SS
INIT --> SESS
INIT --> REPO
```

**Diagram sources**
- [sqlite/__init__.py](file://src/memu/database/sqlite/__init__.py#L1-L26)
- [sqlite/models.py](file://src/memu/database/sqlite/models.py#L1-L200)
- [sqlite/schema.py](file://src/memu/database/sqlite/schema.py#L1-L200)
- [sqlite/session.py](file://src/memu/database/sqlite/session.py#L1-L200)
- [sqlite/repositories/category_item_repo.py](file://src/memu/database/sqlite/repositories/category_item_repo.py#L1-L200)
- [sqlite/repositories/memory_category_repo.py](file://src/memu/database/sqlite/repositories/memory_category_repo.py#L1-L200)
- [sqlite/repositories/memory_item_repo.py](file://src/memu/database/sqlite/repositories/memory_item_repo.py#L1-L200)
- [sqlite/repositories/resource_repo.py](file://src/memu/database/sqlite/repositories/resource_repo.py#L1-L200)

**Section sources**
- [sqlite/__init__.py](file://src/memu/database/sqlite/__init__.py#L1-L26)
- [sqlite/models.py](file://src/memu/database/sqlite/models.py#L1-L200)
- [sqlite/schema.py](file://src/memu/database/sqlite/schema.py#L1-L200)
- [sqlite/session.py](file://src/memu/database/sqlite/session.py#L1-L200)

### PostgreSQL Backend
The PostgreSQL backend supports production deployments and optional pgvector for vector operations. It includes models, schema, session management, migrations, and repository implementations.

```mermaid
graph TB
subgraph "PostgreSQL Backend"
PM["postgres/models.py"]
PS["postgres/schema.py"]
PSESS["postgres/session.py"]
PMIG["postgres/migration.py"]
PMENV["postgres/migrations/env.py"]
PREPO["postgres/repositories/*"]
PINIT["postgres/__init__.py"]
end
PINIT --> PM
PINIT --> PS
PINIT --> PSESS
PINIT --> PMIG
PINIT --> PMENV
PINIT --> PREPO
```

**Diagram sources**
- [postgres/__init__.py](file://src/memu/database/postgres/__init__.py#L1-L26)
- [postgres/models.py](file://src/memu/database/postgres/models.py#L1-L200)
- [postgres/schema.py](file://src/memu/database/postgres/schema.py#L1-L200)
- [postgres/session.py](file://src/memu/database/postgres/session.py#L1-L200)
- [postgres/migration.py](file://src/memu/database/postgres/migration.py#L1-L200)
- [postgres/migrations/env.py](file://src/memu/database/postgres/migrations/env.py#L1-L200)
- [postgres/repositories/base.py](file://src/memu/database/postgres/repositories/base.py#L1-L200)
- [postgres/repositories/category_item_repo.py](file://src/memu/database/postgres/repositories/category_item_repo.py#L1-L200)
- [postgres/repositories/memory_category_repo.py](file://src/memu/database/postgres/repositories/memory_category_repo.py#L1-L200)
- [postgres/repositories/memory_item_repo.py](file://src/memu/database/postgres/repositories/memory_item_repo.py#L1-L200)
- [postgres/repositories/resource_repo.py](file://src/memu/database/postgres/repositories/resource_repo.py#L1-L200)

**Section sources**
- [postgres/__init__.py](file://src/memu/database/postgres/__init__.py#L1-L26)
- [postgres/models.py](file://src/memu/database/postgres/models.py#L1-L200)
- [postgres/schema.py](file://src/memu/database/postgres/schema.py#L1-L200)
- [postgres/session.py](file://src/memu/database/postgres/session.py#L1-L200)
- [postgres/migration.py](file://src/memu/database/postgres/migration.py#L1-L200)
- [postgres/migrations/env.py](file://src/memu/database/postgres/migrations/env.py#L1-L200)

### Retrieval and Ranking Pipeline
Retrieval leverages vector search utilities and ranking prompts. The workflow engine coordinates steps that invoke retrieval and ranking.

```mermaid
sequenceDiagram
participant User as "User"
participant WF as "workflow/runner.py"
participant Step as "workflow/step.py"
participant App as "app/retrieve.py"
participant DB as "Database (repo)"
participant Vec as "inmemory/vector.py"
participant Rank as "prompts/retrieve/*"
User->>WF : trigger pipeline
WF->>Step : execute step
Step->>App : retrieve(memory_query)
App->>DB : vector_search_items(query_vec, top_k)
DB->>Vec : cosine_topk / cosine_topk_salience
Vec-->>DB : top-k results
App->>Rank : rank results (category/item/resource)
Rank-->>App : ranked results
App-->>Step : retrieval payload
Step-->>WF : continue pipeline
```

**Diagram sources**
- [workflow/runner.py](file://src/memu/workflow/runner.py#L1-L200)
- [workflow/step.py](file://src/memu/workflow/step.py#L1-L200)
- [app/retrieve.py](file://src/memu/app/retrieve.py#L1-L200)
- [inmemory/vector.py](file://src/memu/database/inmemory/vector.py#L56-L138)
- [prompts/retrieve/llm_category_ranker.py](file://src/memu/prompts/retrieve/llm_category_ranker.py#L1-L200)
- [prompts/retrieve/llm_item_ranker.py](file://src/memu/prompts/retrieve/llm_item_ranker.py#L1-L200)
- [prompts/retrieve/llm_resource_ranker.py](file://src/memu/prompts/retrieve/llm_resource_ranker.py#L1-L200)
- [prompts/retrieve/judger.py](file://src/memu/prompts/retrieve/judger.py#L1-L200)

**Section sources**
- [app/retrieve.py](file://src/memu/app/retrieve.py#L1-L200)
- [inmemory/vector.py](file://src/memu/database/inmemory/vector.py#L56-L138)
- [prompts/retrieve/llm_category_ranker.py](file://src/memu/prompts/retrieve/llm_category_ranker.py#L1-L200)
- [prompts/retrieve/llm_item_ranker.py](file://src/memu/prompts/retrieve/llm_item_ranker.py#L1-L200)
- [prompts/retrieve/llm_resource_ranker.py](file://src/memu/prompts/retrieve/llm_resource_ranker.py#L1-L200)
- [prompts/retrieve/judger.py](file://src/memu/prompts/retrieve/judger.py#L1-L200)

## Dependency Analysis
The database layer maintains low coupling through the Database protocol and repository contracts. Backends are loaded lazily to avoid unnecessary dependencies.

```mermaid
graph LR
F["factory.py"] --> IM["inmemory/*"]
F --> SQ["sqlite/*"]
F --> PG["postgres/*"]
IF["interfaces.py"] --> RE["repositories/*"]
RE --> MD["models.py"]
IM --> ST["state.py"]
SQ --> ST
PG --> ST
```

**Diagram sources**
- [factory.py](file://src/memu/database/factory.py#L15-L44)
- [interfaces.py](file://src/memu/database/interfaces.py#L12-L27)
- [repositories/__init__.py](file://src/memu/database/repositories/__init__.py#L1-L7)
- [models.py](file://src/memu/database/models.py#L35-L148)
- [state.py](file://src/memu/database/state.py#L8-L14)

**Section sources**
- [factory.py](file://src/memu/database/factory.py#L15-L44)
- [interfaces.py](file://src/memu/database/interfaces.py#L12-L27)

## Performance Considerations
- In-memory backend: Fastest for development; no disk I/O; state held in RAM.
- SQLite backend: Single-file database; suitable for small to medium deployments; consider WAL mode and appropriate pragmas for concurrency.
- PostgreSQL backend: Production-grade; enable connection pooling, indexing, and vector extensions for efficient similarity search.

Vector search performance:
- Use vectorized operations and top-k selection to minimize sorting overhead.
- Salience-aware scoring combines similarity, reinforcement count, and recency to improve relevance.

[No sources needed since this section provides general guidance]

## Troubleshooting Guide
Common issues and remedies:
- Unsupported provider: Verify configuration provider value matches supported backends.
- Missing pgvector: Ensure PostgreSQL backend is built with pgvector extension when enabled.
- Session/connection errors: Confirm session factories and environment variables are correctly configured for each backend.
- Migration failures: Review migration scripts and environment configuration for PostgreSQL.

**Section sources**
- [factory.py](file://src/memu/database/factory.py#L42-L43)
- [postgres/migration.py](file://src/memu/database/postgres/migration.py#L1-L200)
- [postgres/migrations/env.py](file://src/memu/database/postgres/migrations/env.py#L1-L200)

## Conclusion
The database layer employs a clean separation of concerns: a backend-agnostic protocol, typed repository contracts, and pluggable implementations. This design enables seamless switching between in-memory, SQLite, and PostgreSQL backends, supporting diverse deployment scenarios from development to production. Vector utilities and retrieval ranking integrate tightly with the workflow engine and LLM providers to deliver effective memory recall.

[No sources needed since this section summarizes without analyzing specific files]

## Appendices

### Infrastructure Requirements and Deployment Topology
- In-memory backend: Minimal footprint; ideal for ephemeral environments and CI.
- SQLite backend: Single-node file-based storage; suitable for embedded or small-scale deployments; ensure file permissions and path availability.
- PostgreSQL backend: Requires a running PostgreSQL instance; optional pgvector extension for vector operations; configure replication and backups for HA.

[No sources needed since this section provides general guidance]

### Technology Stack and Compatibility Notes
- Python libraries: Pydantic for data validation, pendulum for time handling, NumPy for vector math.
- Optional PostgreSQL/pgvector: Ensure compatible versions with the chosen Postgres driver and migration tooling.
- Embedding providers: OpenAI and Doubao backends are available for generating embeddings used by the retrieval pipeline.

[No sources needed since this section provides general guidance]

### Testing Coverage
- Unit tests exercise each backend’s capabilities and repository operations.

**Section sources**
- [test_inmemory.py](file://tests/test_inmemory.py#L1-L200)
- [test_sqlite.py](file://tests/test_sqlite.py#L1-L200)
- [test_postgres.py](file://tests/test_postgres.py#L1-L200)