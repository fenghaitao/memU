# Testing and Deployment

<cite>
**Referenced Files in This Document**
- [README.md](file://README.md)
- [pyproject.toml](file://pyproject.toml)
- [Makefile](file://Makefile)
- [.pre-commit-config.yaml](file://.pre-commit-config.yaml)
- [setup.cfg](file://setup.cfg)
- [Cargo.toml](file://Cargo.toml)
- [src/memu/__init__.py](file://src/memu/__init__.py)
- [tests/test_inmemory.py](file://tests/test_inmemory.py)
- [tests/test_postgres.py](file://tests/test_postgres.py)
- [tests/test_openrouter.py](file://tests/test_openrouter.py)
- [tests/rust_entry_test.py](file://tests/rust_entry_test.py)
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
This document provides a comprehensive guide to testing and deployment for reliable memU installations and operations. It explains the test suite organization, unit and integration testing strategies, and performance testing methodologies. It also documents deployment procedures for different environments, CI/CD and quality assurance processes, release management, database migrations, configuration validation, rollback strategies, monitoring and logging, alerting, troubleshooting, scaling, secrets management, security, benchmarking, capacity planning, and operational excellence.

## Project Structure
The repository is a Python/Rust hybrid project with a clear separation of concerns:
- Python core and application logic under src/memu
- Rust extension under src/lib.rs, compiled via Cargo and exposed to Python
- Tests under tests/, covering in-memory and PostgreSQL persistence, provider integrations, and Rust entry points
- Tooling and configuration for linting, type checking, coverage, and packaging

```mermaid
graph TB
A["pyproject.toml<br/>Dependencies, dev groups, pytest config"] --> B["tests/<br/>Unit/integration tests"]
A --> C["src/memu/<br/>Application, database, LLM/embedding backends"]
D["Cargo.toml<br/>Rust crate config"] --> E["src/lib.rs<br/>Rust implementation"]
E --> F["src/memu/_core<br/>Compiled extension"]
F --> G["src/memu/__init__.py<br/>Public API and entry"]
H[".pre-commit-config.yaml<br/>Lint/format hooks"] --> I["Makefile<br/>Install/check/test tasks"]
J["setup.cfg<br/>Flake8 config"] --> I
K["README.md<br/>Quick start, examples, environment vars"] --> B
```

**Diagram sources**
- [pyproject.toml](file://pyproject.toml#L1-L181)
- [Cargo.toml](file://Cargo.toml#L1-L15)
- [src/memu/__init__.py](file://src/memu/__init__.py#L1-L10)
- [.pre-commit-config.yaml](file://.pre-commit-config.yaml#L1-L21)
- [Makefile](file://Makefile#L1-L23)
- [setup.cfg](file://setup.cfg#L1-L19)
- [README.md](file://README.md#L276-L317)

**Section sources**
- [pyproject.toml](file://pyproject.toml#L1-L181)
- [Cargo.toml](file://Cargo.toml#L1-L15)
- [src/memu/__init__.py](file://src/memu/__init__.py#L1-L10)
- [.pre-commit-config.yaml](file://.pre-commit-config.yaml#L1-L21)
- [Makefile](file://Makefile#L1-L23)
- [setup.cfg](file://setup.cfg#L1-L19)
- [README.md](file://README.md#L276-L317)

## Core Components
- Application service: MemoryService orchestrates continuous learning (memorize) and dual-mode retrieval (RAG/LLM).
- Database backends: in-memory and PostgreSQL with pgvector support for embeddings.
- Provider backends: OpenAI, OpenRouter, Grok, Doubao, LazyLLM client integration.
- Rust extension: Compiled ABI-compatible module exposing a minimal entry point for validation.

Key testing components:
- In-memory workflow test validates end-to-end memorize and retrieve with RAG and LLM modes.
- PostgreSQL workflow test validates persistent storage and vector index setup.
- OpenRouter workflow test validates provider integration end-to-end.
- Rust entry test validates the compiled extension is importable and functional.

**Section sources**
- [tests/test_inmemory.py](file://tests/test_inmemory.py#L1-L90)
- [tests/test_postgres.py](file://tests/test_postgres.py#L1-L83)
- [tests/test_openrouter.py](file://tests/test_openrouter.py#L1-L162)
- [tests/rust_entry_test.py](file://tests/rust_entry_test.py#L1-L6)
- [src/memu/__init__.py](file://src/memu/__init__.py#L1-L10)

## Architecture Overview
The testing and deployment architecture centers around:
- Python application layer with configurable LLM and embedding providers
- Database abstraction supporting in-memory and PostgreSQL/pgvector
- Rust extension compiled as a shared library for performance-sensitive routines
- Test harnesses validating provider integrations and persistence backends
- Tooling for pre-commit hooks, type checking, linting, and coverage

```mermaid
graph TB
subgraph "Runtime"
Svc["MemoryService<br/>memu.app.service"]
DB["Database Abstraction<br/>inmemory/postgres"]
Prov["Provider Backends<br/>OpenAI/OpenRouter/Grok/Doubao"]
Rust["Rust Extension<br/>_core (compiled)"]
end
subgraph "Testing"
T1["tests/test_inmemory.py"]
T2["tests/test_postgres.py"]
T3["tests/test_openrouter.py"]
TR["tests/rust_entry_test.py"]
end
subgraph "Tooling"
Pyt["pytest<br/>pyproject.toml"]
MyPy["mypy<br/>pyproject.toml"]
Ruff["ruff<br/>.pre-commit-config.yaml"]
Cov["coverage<br/>pyproject.toml"]
end
T1 --> Svc
T2 --> Svc
T3 --> Svc
TR --> Rust
Svc --> DB
Svc --> Prov
Svc --> Rust
Pyt --> T1
Pyt --> T2
Pyt --> T3
Pyt --> TR
MyPy --> Svc
Ruff --> Svc
Cov --> Pyt
```

**Diagram sources**
- [tests/test_inmemory.py](file://tests/test_inmemory.py#L1-L90)
- [tests/test_postgres.py](file://tests/test_postgres.py#L1-L83)
- [tests/test_openrouter.py](file://tests/test_openrouter.py#L1-L162)
- [tests/rust_entry_test.py](file://tests/rust_entry_test.py#L1-L6)
- [pyproject.toml](file://pyproject.toml#L63-L181)
- [.pre-commit-config.yaml](file://.pre-commit-config.yaml#L1-L21)

## Detailed Component Analysis

### Test Suite Organization
- Unit tests: rust_entry_test.py validates the Rust extension import and basic return value.
- Integration tests:
  - test_inmemory.py: end-to-end workflow with in-memory persistence and both retrieval methods.
  - test_postgres.py: end-to-end workflow with PostgreSQL persistence and vector index initialization.
  - test_openrouter.py: end-to-end workflow with OpenRouter provider and multiple retrieval modes.
- Test configuration:
  - pytest configured via pyproject.toml with testpaths, asyncio mode, and logging.
  - Coverage enabled via pytest-cov and configured in pyproject.toml.
  - Linting and formatting enforced by pre-commit hooks and Ruff.

```mermaid
flowchart TD
Start(["Run Tests"]) --> LoadCfg["Load pytest config<br/>pyproject.toml"]
LoadCfg --> Discover["Discover tests<br/>tests/"]
Discover --> Unit["Unit: rust_entry_test.py"]
Discover --> Integrations["Integration:<br/>test_inmemory.py<br/>test_postgres.py<br/>test_openrouter.py"]
Unit --> ExecUnit["Execute unit tests"]
Integrations --> ExecInt["Execute integration tests"]
ExecUnit --> Coverage["Collect coverage<br/>pyproject.toml"]
ExecInt --> Coverage
Coverage --> Report["Generate coverage report"]
Report --> End(["Done"])
```

**Diagram sources**
- [pyproject.toml](file://pyproject.toml#L176-L181)
- [tests/rust_entry_test.py](file://tests/rust_entry_test.py#L1-L6)
- [tests/test_inmemory.py](file://tests/test_inmemory.py#L1-L90)
- [tests/test_postgres.py](file://tests/test_postgres.py#L1-L83)
- [tests/test_openrouter.py](file://tests/test_openrouter.py#L1-L162)

**Section sources**
- [pyproject.toml](file://pyproject.toml#L63-L181)
- [tests/rust_entry_test.py](file://tests/rust_entry_test.py#L1-L6)
- [tests/test_inmemory.py](file://tests/test_inmemory.py#L1-L90)
- [tests/test_postgres.py](file://tests/test_postgres.py#L1-L83)
- [tests/test_openrouter.py](file://tests/test_openrouter.py#L1-L162)

### In-Memory Workflow Test
This test validates:
- Initialization of MemoryService with in-memory metadata store
- Continuous learning (memorize) from a conversation resource
- Dual-mode retrieval (RAG and LLM) with scoped filters

```mermaid
sequenceDiagram
participant Test as "test_inmemory.py"
participant Service as "MemoryService"
participant DB as "InMemory Store"
Test->>Service : Initialize with llm_profiles and metadata_store=inmemory
Test->>Service : memorize(resource_url, modality, user)
Service->>DB : Persist categories/items/resources
Test->>Service : retrieve(queries, where={"user_id" : "..."}, method="rag")
Service->>DB : Query categories/items/resources
Test->>Service : retrieve(queries, where={"user_id" : "..."}, method="llm")
Service->>DB : Query categories/items/resources
Test-->>Test : Assert counts and print summaries
```

**Diagram sources**
- [tests/test_inmemory.py](file://tests/test_inmemory.py#L1-L90)

**Section sources**
- [tests/test_inmemory.py](file://tests/test_inmemory.py#L1-L90)

### PostgreSQL Workflow Test
This test validates:
- Initialization of MemoryService with PostgreSQL metadata store and DDL mode
- Vector index auto-configuration for pgvector
- End-to-end memorize and retrieval with persistent storage

```mermaid
sequenceDiagram
participant Test as "test_postgres.py"
participant Service as "MemoryService"
participant PG as "PostgreSQL + pgvector"
Test->>Service : Initialize with llm_profiles and metadata_store=postgres DSN
Test->>Service : memorize(resource_url, modality, user)
Service->>PG : Create tables and insert categories/items/resources
Test->>Service : retrieve(queries, where={"user_id" : "..."}, method="rag")
Service->>PG : Query embeddings and similarity
Test->>Service : retrieve(queries, where={"user_id" : "..."}, method="llm")
Service->>PG : Query categories/items/resources
Test-->>Test : Assert counts and print summaries
```

**Diagram sources**
- [tests/test_postgres.py](file://tests/test_postgres.py#L1-L83)

**Section sources**
- [tests/test_postgres.py](file://tests/test_postgres.py#L1-L83)

### OpenRouter Integration Test
This test validates:
- Provider configuration for OpenRouter
- Full workflow: memorize, RAG retrieval, LLM retrieval, list items and categories
- Output serialization to a JSON file for inspection

```mermaid
sequenceDiagram
participant Test as "test_openrouter.py"
participant Service as "MemoryService"
participant OR as "OpenRouter API"
Test->>Service : Initialize with llm_profiles.provider=openrouter
Test->>Service : memorize(resource_url, modality, user)
Service->>OR : Embeddings and completions
Test->>Service : retrieve(queries, method="rag")
Service->>OR : Embeddings
Test->>Service : retrieve(queries, method="llm")
Service->>OR : Completions
Test->>Service : list_memory_items/list_memory_categories
Test-->>Test : Save output JSON and assert counts
```

**Diagram sources**
- [tests/test_openrouter.py](file://tests/test_openrouter.py#L1-L162)

**Section sources**
- [tests/test_openrouter.py](file://tests/test_openrouter.py#L1-L162)

### Rust Extension Validation
This test ensures the compiled Rust module is importable and returns the expected value.

```mermaid
flowchart TD
A["tests/rust_entry_test.py"] --> B["Import _rust_entry from memu"]
B --> C["_rust_entry()"]
C --> D["hello_from_bin() from _core"]
D --> E["Assert return equals expected string"]
```

**Diagram sources**
- [tests/rust_entry_test.py](file://tests/rust_entry_test.py#L1-L6)
- [src/memu/__init__.py](file://src/memu/__init__.py#L1-L10)
- [Cargo.toml](file://Cargo.toml#L1-L15)

**Section sources**
- [tests/rust_entry_test.py](file://tests/rust_entry_test.py#L1-L6)
- [src/memu/__init__.py](file://src/memu/__init__.py#L1-L10)
- [Cargo.toml](file://Cargo.toml#L1-L15)

## Dependency Analysis
- Python dependencies are declared in pyproject.toml with strict version pins and optional extras for PostgreSQL, LangGraph, and Claude SDK.
- Dev dependencies include linting (Ruff), type checking (mypy), dependency analysis (deptry), and testing (pytest, pytest-asyncio, pytest-cov).
- Rust dependency pyo3 is configured for ABI stability targeting Python 3.13.
- Pre-commit hooks enforce code quality and formatting prior to commits.

```mermaid
graph LR
P["pyproject.toml"] --> D1["Production deps"]
P --> D2["Dev deps"]
P --> D3["Optional deps"]
R["Cargo.toml"] --> R1["pyo3 (ABI3-Py313)"]
PC[".pre-commit-config.yaml"] --> L["Ruff hooks"]
M["Makefile"] --> T["pytest, mypy, deptry"]
```

**Diagram sources**
- [pyproject.toml](file://pyproject.toml#L20-L73)
- [Cargo.toml](file://Cargo.toml#L11-L15)
- [.pre-commit-config.yaml](file://.pre-commit-config.yaml#L1-L21)
- [Makefile](file://Makefile#L7-L23)

**Section sources**
- [pyproject.toml](file://pyproject.toml#L20-L73)
- [Cargo.toml](file://Cargo.toml#L11-L15)
- [.pre-commit-config.yaml](file://.pre-commit-config.yaml#L1-L21)
- [Makefile](file://Makefile#L7-L23)

## Performance Considerations
- Benchmarking: The project reports average accuracy on a benchmark dataset, indicating readiness for performance validation.
- Retrieval modes:
  - RAG-based retrieval uses embeddings for fast, proactive context assembly.
  - LLM-based retrieval performs deeper reasoning but at higher cost and latency.
- Recommendations:
  - Use RAG for real-time suggestions and continuous monitoring.
  - Use LLM retrieval for complex anticipatory reasoning when acceptable latency/cost is ensured.
  - Monitor embedding throughput and vector index performance in PostgreSQL deployments.

[No sources needed since this section provides general guidance]

## Troubleshooting Guide
Common issues and resolutions:
- Missing environment variables:
  - OPENAI_API_KEY for OpenAI-based tests
  - OPENROUTER_API_KEY for OpenRouter tests
  - POSTGRES_DSN for PostgreSQL tests
- PostgreSQL not running or pgvector missing:
  - Ensure container is started with the correct image and port mapping before running PostgreSQL tests.
- Provider configuration errors:
  - Verify provider profiles in llm_profiles and correct model identifiers.
- Coverage and lint failures:
  - Run pre-commit hooks and mypy locally before committing.
- Rust compilation issues:
  - Confirm Rust toolchain and maturin installation; ensure ABI compatibility settings.

Concrete references:
- Environment variables and quick start examples are documented in the repository’s README.
- Test scripts demonstrate expected environment variables and usage patterns.

**Section sources**
- [README.md](file://README.md#L276-L317)
- [tests/test_inmemory.py](file://tests/test_inmemory.py#L1-L90)
- [tests/test_postgres.py](file://tests/test_postgres.py#L1-L83)
- [tests/test_openrouter.py](file://tests/test_openrouter.py#L1-L162)

## Conclusion
The memU project provides a robust testing and deployment foundation with:
- Clear test organization across unit and integration suites
- Configurable persistence and provider backends validated by dedicated tests
- Tooling for quality assurance and coverage
- Practical deployment examples for self-hosted environments

Adopting the recommended practices in this document will ensure reliable installations, predictable operations, and smooth CI/CD integration.

[No sources needed since this section summarizes without analyzing specific files]

## Appendices

### A. Running Tests Locally
- Install dependencies and pre-commit hooks:
  - Use the provided Makefile targets to synchronize environment and install hooks.
- Execute tests:
  - Run pytest with coverage reporting configured in pyproject.toml.
- Inspect coverage:
  - Coverage XML report is generated per pytest configuration.

**Section sources**
- [Makefile](file://Makefile#L1-L23)
- [pyproject.toml](file://pyproject.toml#L176-L181)

### B. CI/CD and Quality Assurance
- Pre-commit hooks:
  - Enforce case/conflict checks, YAML/JSON formatting, and Ruff lint/format.
- Local quality checks:
  - Lock file verification, pre-commit run, mypy type checks, and deptry dependency analysis.
- Test execution:
  - pytest with asyncio mode and logging enabled.

**Section sources**
- [.pre-commit-config.yaml](file://.pre-commit-config.yaml#L1-L21)
- [Makefile](file://Makefile#L7-L23)
- [pyproject.toml](file://pyproject.toml#L176-L181)

### C. Release Management
- Versioning:
  - Project version is defined in pyproject.toml; increment according to semantic versioning.
- Packaging:
  - Rust extension compiled via maturin with ABI3 targeting Python 3.13.
- Distribution:
  - Publish to PyPI using standard Python packaging workflows.

**Section sources**
- [pyproject.toml](file://pyproject.toml#L1-L181)
- [Cargo.toml](file://Cargo.toml#L1-L15)

### D. Database Migrations and Rollbacks
- PostgreSQL schema:
  - DDL mode is supported in tests; configure DSN and ensure pgvector is available.
- Migration tooling:
  - Alembic is included as a dependency; integrate migration scripts as needed for production deployments.
- Rollback strategy:
  - Maintain safe DDL patterns and versioned migrations; test rollback procedures in staging.

**Section sources**
- [tests/test_postgres.py](file://tests/test_postgres.py#L1-L83)
- [pyproject.toml](file://pyproject.toml#L27-L27)

### E. Monitoring and Logging Best Practices
- Logging:
  - Enable INFO-level logs during tests via pytest configuration.
- Observability:
  - Instrument retrieval and memorize operations with structured logs.
- Alerting:
  - Define thresholds for embedding latency, provider error rates, and database connection health.
- Health checks:
  - Implement lightweight endpoints to verify service readiness and provider connectivity.

[No sources needed since this section provides general guidance]

### F. Scaling and Security
- Scaling:
  - Horizontal scaling of workers for ingestion and retrieval; use async concurrency patterns.
  - Database scaling: read replicas for retrieval-heavy workloads.
- Secrets management:
  - Store API keys and DSNs in environment variables or secret managers; avoid hardcoding.
- Security:
  - Validate provider responses, sanitize inputs, and apply least-privilege network policies.

[No sources needed since this section provides general guidance]

### G. Performance Benchmarking and Capacity Planning
- Benchmarks:
  - Use reported metrics as baselines; extend with custom benchmarks for your workload.
- Capacity planning:
  - Estimate embedding storage, compute, and database I/O needs; provision headroom for growth.
- Profiling:
  - Profile hotspots in retrieval and embedding pipelines; optimize batch sizes and parallelism.

[No sources needed since this section provides general guidance]