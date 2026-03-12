# Database Configuration

<cite>
**Referenced Files in This Document**
- [settings.py](file://src/memu/app/settings.py)
- [factory.py](file://src/memu/database/factory.py)
- [interfaces.py](file://src/memu/database/interfaces.py)
- [models.py](file://src/memu/database/models.py)
- [state.py](file://src/memu/database/state.py)
- [__init__.py (database)](file://src/memu/database/__init__.py)
- [__init__.py (inmemory)](file://src/memu/database/inmemory/__init__.py)
- [__init__.py (postgres)](file://src/memu/database/postgres/__init__.py)
- [postgres.py](file://src/memu/database/postgres/postgres.py)
- [schema.py](file://src/memu/database/postgres/schema.py)
- [models.py (postgres)](file://src/memu/database/postgres/models.py)
- [session.py](file://src/memu/database/postgres/session.py)
- [__init__.py (sqlite)](file://src/memu/database/sqlite/__init__.py)
- [sqlite.md](file://docs/sqlite.md)
- [0002-pluggable-storage-and-vector-strategy.md](file://docs/adr/0002-pluggable-storage-and-vector-strategy.md)
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
This document explains database configuration options in MemoryService with a focus on the DatabaseConfig structure, metadata_store and vector_index configurations, and the pluggable database architecture supporting in-memory, SQLite, and PostgreSQL backends. It covers connection string formats, authentication settings, vector index configuration for PostgreSQL with pgvector, practical examples for development, testing, and production, initialization and migration handling, performance tuning, scaling considerations, and backup strategies.

## Project Structure
The database configuration and pluggable storage are implemented under the database package. Key areas:
- Settings define DatabaseConfig, MetadataStoreConfig, and VectorIndexConfig.
- Factory builds the appropriate backend based on provider selection.
- Backends implement a unified Database protocol via repositories.
- PostgreSQL integrates SQLModel, SQLAlchemy, and pgvector for vector storage and indexing.
- SQLite provides a lightweight, file-based option suitable for local and testing scenarios.

```mermaid
graph TB
subgraph "App Layer"
S["Settings<br/>DatabaseConfig"]
F["Factory<br/>build_database()"]
end
subgraph "Database Abstraction"
I["Interfaces<br/>Database Protocol"]
M["Models<br/>BaseRecord, Records"]
ST["State<br/>DatabaseState"]
end
subgraph "Backends"
IM["InMemory<br/>build_inmemory_database()"]
PG["Postgres<br/>build_postgres_database()"]
SQ["SQLite<br/>build_sqlite_database()"]
end
S --> F
F --> IM
F --> PG
F --> SQ
IM --> I
PG --> I
SQ --> I
I --> M
I --> ST
```

**Diagram sources**
- [settings.py](file://src/memu/app/settings.py#L299-L322)
- [factory.py](file://src/memu/database/factory.py#L15-L43)
- [interfaces.py](file://src/memu/database/interfaces.py#L12-L27)
- [models.py](file://src/memu/database/models.py#L35-L148)
- [state.py](file://src/memu/database/state.py#L8-L16)
- [__init__.py (inmemory)](file://src/memu/database/inmemory/__init__.py#L10-L22)
- [__init__.py (postgres)](file://src/memu/database/postgres/__init__.py#L10-L33)
- [__init__.py (sqlite)](file://src/memu/database/sqlite/__init__.py#L11-L33)

**Section sources**
- [settings.py](file://src/memu/app/settings.py#L299-L322)
- [factory.py](file://src/memu/database/factory.py#L15-L43)
- [interfaces.py](file://src/memu/database/interfaces.py#L12-L27)
- [models.py](file://src/memu/database/models.py#L35-L148)
- [state.py](file://src/memu/database/state.py#L8-L16)
- [__init__.py (database)](file://src/memu/database/__init__.py#L1-L29)

## Core Components
- DatabaseConfig: Top-level configuration combining metadata_store and vector_index.
- MetadataStoreConfig: Defines provider, DDL mode, and DSN for metadata persistence.
- VectorIndexConfig: Controls vector search provider and optional pgvector DSN.
- Database protocol: A backend-agnostic interface exposing repositories and state.

Key behaviors:
- Automatic vector index provisioning defaults based on metadata_store provider.
- Lazy imports for PostgreSQL to avoid requiring pgvector unless selected.
- SQLite defaults to a local file DSN if none provided.

**Section sources**
- [settings.py](file://src/memu/app/settings.py#L299-L322)
- [factory.py](file://src/memu/database/factory.py#L15-L43)
- [interfaces.py](file://src/memu/database/interfaces.py#L12-L27)

## Architecture Overview
The pluggable architecture routes configuration to a backend builder, which constructs a store implementing the Database protocol. PostgreSQL leverages SQLModel and SQLAlchemy with optional pgvector for vector similarity, while SQLite uses brute-force similarity and stores embeddings as JSON. In-memory backend provides ephemeral state.

```mermaid
sequenceDiagram
participant Cfg as "DatabaseConfig"
participant Fac as "build_database()"
participant IM as "build_inmemory_database()"
participant PG as "build_postgres_database()"
participant SQ as "build_sqlite_database()"
participant Store as "Database Store"
Cfg->>Fac : provider, metadata_store.dsn, vector_index
alt provider == "inmemory"
Fac->>IM : config, user_model
IM-->>Store : InMemoryStore
else provider == "postgres"
Fac->>PG : config, user_model
PG-->>Store : PostgresStore
else provider == "sqlite"
Fac->>SQ : config, user_model
SQ-->>Store : SQLiteStore
end
Store-->>Cfg : ready to use repositories
```

**Diagram sources**
- [factory.py](file://src/memu/database/factory.py#L15-L43)
- [__init__.py (inmemory)](file://src/memu/database/inmemory/__init__.py#L10-L22)
- [__init__.py (postgres)](file://src/memu/database/postgres/__init__.py#L10-L33)
- [__init__.py (sqlite)](file://src/memu/database/sqlite/__init__.py#L11-L33)

## Detailed Component Analysis

### DatabaseConfig and Sub-configurations
- MetadataStoreConfig
  - provider: "inmemory", "postgres", or "sqlite".
  - ddl_mode: "create" or "validate".
  - dsn: Required for "postgres" and "sqlite"; optional for "inmemory".
- VectorIndexConfig
  - provider: "bruteforce", "pgvector", or "none".
  - dsn: Required when provider is "pgvector".
- Automatic defaults
  - If vector_index is omitted:
    - For "postgres", defaults to "pgvector" with the same DSN as metadata_store.
    - Otherwise, defaults to "bruteforce".

Practical implications:
- Omitting vector_index in production with PostgreSQL enables pgvector automatically.
- For non-Postgres, vector_index defaults to brute-force, ensuring compatibility.

**Section sources**
- [settings.py](file://src/memu/app/settings.py#L299-L322)

### In-Memory Backend
- Purpose: Local development and ephemeral state.
- Behavior: No persistence; stores state in-memory via DatabaseState.
- Vector search: Brute-force similarity by default.
- Typical use: Unit tests, demos, or temporary environments.

Implementation highlights:
- Builds scoped models and initializes an InMemoryStore with user scope models.

**Section sources**
- [__init__.py (inmemory)](file://src/memu/database/inmemory/__init__.py#L10-L22)
- [interfaces.py](file://src/memu/database/interfaces.py#L12-L27)
- [state.py](file://src/memu/database/state.py#L8-L16)

### SQLite Backend
- Purpose: Lightweight, file-based persistence; good for local and small-scale deployments.
- DSN defaults: If not provided, defaults to a local file DSN.
- Vector search: Brute-force cosine similarity; embeddings stored as JSON text.
- Migration: Uses SQLAlchemy-based schema creation/validation.

Connection string formats:
- File-based: sqlite:///path/to/database.db
- In-memory: sqlite:///:memory:
- Relative and absolute paths supported.

Migration and initialization:
- DDL mode controls whether to create tables or validate existing schema.

Backup and migration:
- Copy the SQLite file for backups.
- Data export/import examples are provided in documentation.

**Section sources**
- [__init__.py (sqlite)](file://src/memu/database/sqlite/__init__.py#L11-L33)
- [sqlite.md](file://docs/sqlite.md#L62-L89)
- [sqlite.md](file://docs/sqlite.md#L102-L154)

### PostgreSQL Backend
- Purpose: Production-grade persistence with optional pgvector vector indexing.
- Dependencies: SQLModel, SQLAlchemy, and pgvector (optional).
- Vector index: When provider is "pgvector", vectors are stored using the VECTOR type and indexed for similarity search.
- DSN requirement: Required for metadata_store; vector DSN mirrors metadata_store when omitted.

Initialization and migrations:
- SessionManager manages the SQLAlchemy engine and sessions.
- run_migrations applies schema changes based on ddl_mode.
- SQLModel table models are built dynamically with user scope fields and indexes.

```mermaid
classDiagram
class Database {
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
class PostgresStore {
+dsn
+ddl_mode
+vector_provider
+close()
}
class SessionManager {
+session()
+close()
}
Database <|.. PostgresStore
PostgresStore --> SessionManager : "uses"
```

**Diagram sources**
- [interfaces.py](file://src/memu/database/interfaces.py#L12-L27)
- [postgres.py](file://src/memu/database/postgres/postgres.py#L23-L103)
- [session.py](file://src/memu/database/postgres/session.py#L15-L32)

**Section sources**
- [__init__.py (postgres)](file://src/memu/database/postgres/__init__.py#L10-L33)
- [postgres.py](file://src/memu/database/postgres/postgres.py#L33-L103)
- [schema.py](file://src/memu/database/postgres/schema.py#L51-L101)
- [models.py (postgres)](file://src/memu/database/postgres/models.py#L46-L74)
- [session.py](file://src/memu/database/postgres/session.py#L15-L32)

### Vector Index Configuration for PostgreSQL with pgvector
- Provider selection:
  - "pgvector": Enables native vector similarity using PostgreSQL with pgvector extension.
  - "bruteforce": Brute-force cosine similarity fallback for portability.
  - "none": Disables vector index.
- DSN for pgvector:
  - When provider is "pgvector", a DSN is required. If omitted, it inherits from metadata_store.dsn.
- Schema and indexing:
  - Embeddings are stored using the VECTOR column type.
  - Additional indexes and constraints are applied during schema generation.

**Section sources**
- [settings.py](file://src/memu/app/settings.py#L305-L322)
- [schema.py](file://src/memu/database/postgres/schema.py#L20-L24)
- [models.py (postgres)](file://src/memu/database/postgres/models.py#L46-L74)

### Connection Strings and Authentication
- SQLite
  - sqlite:///path/to/db
  - sqlite:///:memory:
- PostgreSQL
  - postgresql+psycopg://user:password@host:port/dbname
  - Environment variable example usage is demonstrated in tests.

Authentication:
- Username/password embedded in DSN for PostgreSQL.
- For production, prefer environment variables or secret managers to avoid hardcoding credentials.

**Section sources**
- [sqlite.md](file://docs/sqlite.md#L62-L68)
- [test_postgres.py](file://tests/test_postgres.py#L10-L10)

### Database Initialization and Migration Handling
- InMemory: No schema or migration steps; state initialized in-memory.
- SQLite: Uses SQLAlchemy-based schema creation/validation controlled by ddl_mode.
- PostgreSQL: run_migrations applies schema changes; SessionManager handles engine lifecycle.

```mermaid
flowchart TD
Start(["Initialize Database"]) --> CheckProvider["Check metadata_store.provider"]
CheckProvider --> |inmemory| IMInit["Initialize InMemoryStore"]
CheckProvider --> |sqlite| SQInit["Initialize SQLiteStore<br/>Apply DDL mode"]
CheckProvider --> |postgres| PGInit["Initialize PostgresStore<br/>Run migrations"]
IMInit --> Ready(["Ready"])
SQInit --> Ready
PGInit --> Ready
```

**Diagram sources**
- [factory.py](file://src/memu/database/factory.py#L15-L43)
- [postgres.py](file://src/memu/database/postgres/postgres.py#L57-L57)

**Section sources**
- [postgres.py](file://src/memu/database/postgres/postgres.py#L57-L57)
- [__init__.py (sqlite)](file://src/memu/database/sqlite/__init__.py#L11-L33)

## Dependency Analysis
- Factory depends on settings and backend builders.
- PostgreSQL backend depends on SQLModel, SQLAlchemy, and pgvector when vector provider is enabled.
- Vector index provider selection affects schema generation and repository behavior.

```mermaid
graph LR
Settings["DatabaseConfig"] --> Factory["build_database()"]
Factory --> IM["InMemory Builder"]
Factory --> PG["Postgres Builder"]
Factory --> SQ["SQLite Builder"]
PG --> SQLM["SQLModel/SQLAlchemy"]
PG --> PGV["pgvector"]
SQ --> SQLM
```

**Diagram sources**
- [factory.py](file://src/memu/database/factory.py#L15-L43)
- [schema.py](file://src/memu/database/postgres/schema.py#L51-L101)
- [models.py (postgres)](file://src/memu/database/postgres/models.py#L46-L74)

**Section sources**
- [factory.py](file://src/memu/database/factory.py#L15-L43)
- [schema.py](file://src/memu/database/postgres/schema.py#L51-L101)

## Performance Considerations
- InMemory: Fastest for local development; no disk I/O; no persistence.
- SQLite: Good for small datasets; brute-force vector search; JSON embeddings.
- PostgreSQL with pgvector: Indexed vector similarity; scales to large datasets; requires proper indexing and hardware.

Recommendations:
- Use "bruteforce" for small datasets or non-Postgres backends.
- Enable "pgvector" for production vector search needs.
- Tune PostgreSQL parameters (shared_buffers, work_mem, effective_cache_size) and ensure appropriate indexes.

[No sources needed since this section provides general guidance]

## Troubleshooting Guide
Common issues and resolutions:
- Missing DSN for PostgreSQL or SQLite:
  - Ensure metadata_store.dsn is provided; otherwise, PostgreSQL requires a DSN and SQLite defaults to a file path.
- Missing pgvector dependency:
  - When vector provider is "pgvector", pgvector must be installed; otherwise, an ImportError is raised.
- Vector index provider mismatch:
  - If vector_index is "pgvector" but no DSN is provided, it will inherit from metadata_store.dsn.
- Engine lifecycle:
  - Always call close() on stores that manage sessions to release connections.

**Section sources**
- [__init__.py (postgres)](file://src/memu/database/postgres/__init__.py#L15-L18)
- [schema.py](file://src/memu/database/postgres/schema.py#L20-L24)
- [postgres.py](file://src/memu/database/postgres/postgres.py#L101-L103)

## Conclusion
DatabaseConfig provides a flexible, backend-agnostic configuration for MemoryService. The pluggable architecture supports in-memory, SQLite, and PostgreSQL backends, with automatic vector index defaults and explicit control over DSNs and providers. Choose the backend and vector strategy aligned with your deployment footprint, and leverage migrations and session management for robust operation.

[No sources needed since this section summarizes without analyzing specific files]

## Appendices

### Practical Examples

- Development (SQLite default)
  - Configure metadata_store.provider to "sqlite" without a DSN to use the default file path.
  - Keep vector_index as "bruteforce" for simplicity.

- Testing (InMemory)
  - Set metadata_store.provider to "inmemory" for ephemeral state and fast iteration.

- Production (PostgreSQL with pgvector)
  - Set metadata_store.provider to "postgres" with a DSN.
  - Leave vector_index unset to auto-enable "pgvector" with the same DSN.
  - Ensure pgvector is installed and enabled in the target PostgreSQL instance.

- Migration from SQLite to PostgreSQL
  - Use the documented pattern to load from SQLite and write to PostgreSQL.

**Section sources**
- [sqlite.md](file://docs/sqlite.md#L14-L37)
- [sqlite.md](file://docs/sqlite.md#L115-L154)
- [test_postgres.py](file://tests/test_postgres.py#L18-L29)

### Scaling and Backup Strategies
- Scaling
  - PostgreSQL with pgvector scales better for large datasets and vector similarity.
  - Consider read replicas, connection pooling, and appropriate PostgreSQL tuning for production.

- Backup
  - PostgreSQL: Use logical or physical backups (e.g., pg_dump, base backups).
  - SQLite: Copy the database file for backups; consider WAL mode for improved durability.

**Section sources**
- [sqlite.md](file://docs/sqlite.md#L102-L114)
- [0002-pluggable-storage-and-vector-strategy.md](file://docs/adr/0002-pluggable-storage-and-vector-strategy.md#L1-L43)