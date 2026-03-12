# Workflow Pipeline Architecture

<cite>
**Referenced Files in This Document**
- [pipeline.py](file://src/memu/workflow/pipeline.py)
- [step.py](file://src/memu/workflow/step.py)
- [runner.py](file://src/memu/workflow/runner.py)
- [interceptor.py](file://src/memu/workflow/interceptor.py)
- [service.py](file://src/memu/app/service.py)
- [memorize.py](file://src/memu/app/memorize.py)
- [retrieve.py](file://src/memu/app/retrieve.py)
- [crud.py](file://src/memu/app/crud.py)
- [architecture.md](file://docs/architecture.md)
- [0001-workflow-pipeline-architecture.md](file://docs/adr/0001-workflow-pipeline-architecture.md)
- [models.py](file://src/memu/database/models.py)
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
This document explains memU’s modular execution engine built around workflow pipelines. It covers how the PipelineManager registers and mutates pipelines, how WorkflowStep defines discrete memory operations, and how WorkflowRunner executes them asynchronously. It also documents the interceptor system for workflow hooks, error handling, state management across steps, and how the pipeline architecture enables extensibility and customization. Practical examples describe how to construct workflows, insert/remove steps, and configure pipelines.

## Project Structure
The workflow pipeline lives under src/memu/workflow and integrates with application mixins under src/memu/app. The MemoryService composes capabilities, registers pipelines, and exposes mutation APIs. Workflows for memorize, retrieve, and CRUD operations are defined as lists of WorkflowStep instances.

```mermaid
graph TB
subgraph "Workflow Core"
PM["PipelineManager<br/>registers/mutates pipelines"]
Step["WorkflowStep<br/>step definition"]
RunSteps["run_steps()<br/>executes steps"]
Runner["WorkflowRunner<br/>execution backend"]
Interceptor["WorkflowInterceptorRegistry<br/>hooks"]
end
subgraph "Application Integrations"
Service["MemoryService<br/>orchestrates"]
Mem["MemorizeMixin<br/>memorize workflow"]
Ret["RetrieveMixin<br/>retrieve workflow"]
CRUD["CRUDMixin<br/>CRUD workflows"]
end
Service --> PM
Service --> Runner
Service --> Interceptor
Mem --> PM
Ret --> PM
CRUD --> PM
PM --> Step
Runner --> RunSteps
Interceptor --> RunSteps
```

**Diagram sources**
- [pipeline.py](file://src/memu/workflow/pipeline.py#L21-L171)
- [step.py](file://src/memu/workflow/step.py#L16-L102)
- [runner.py](file://src/memu/workflow/runner.py#L12-L82)
- [interceptor.py](file://src/memu/workflow/interceptor.py#L56-L219)
- [service.py](file://src/memu/app/service.py#L49-L427)
- [memorize.py](file://src/memu/app/memorize.py#L47-L326)
- [retrieve.py](file://src/memu/app/retrieve.py#L27-L226)
- [crud.py](file://src/memu/app/crud.py#L100-L245)

**Section sources**
- [architecture.md](file://docs/architecture.md#L1-L50)
- [0001-workflow-pipeline-architecture.md](file://docs/adr/0001-workflow-pipeline-architecture.md#L1-L36)
- [service.py](file://src/memu/app/service.py#L49-L427)

## Core Components
- PipelineManager: Central registry for named pipelines with revisioning and mutation. Validates step uniqueness, capability availability, LLM profile validity, and state key dependencies.
- WorkflowStep: Encapsulates a single step with handler, requires/produces state keys, capabilities, and optional config. Provides async run() that validates handler return type.
- WorkflowRunner: Pluggable execution backend protocol with a default local runner. Supports registering external runners.
- WorkflowInterceptorRegistry: Manages before/after/on_error hooks around each step, with snapshot-based invocation and strict-mode exception handling.
- MemoryService: Composes the system, registers pipelines, resolves runners, and exposes mutation APIs for runtime customization.

**Section sources**
- [pipeline.py](file://src/memu/workflow/pipeline.py#L21-L171)
- [step.py](file://src/memu/workflow/step.py#L16-L102)
- [runner.py](file://src/memu/workflow/runner.py#L12-L82)
- [interceptor.py](file://src/memu/workflow/interceptor.py#L56-L219)
- [service.py](file://src/memu/app/service.py#L49-L427)

## Architecture Overview
The system models each high-level operation (memorize, retrieve, CRUD) as a named pipeline composed of ordered WorkflowStep units. Execution is asynchronous and orchestrated by a WorkflowRunner. Interceptors wrap each step to provide instrumentation and control.

```mermaid
sequenceDiagram
participant Client as "Caller"
participant Service as "MemoryService"
participant PM as "PipelineManager"
participant Runner as "WorkflowRunner"
participant Steps as "run_steps()"
participant Step as "WorkflowStep.handler"
Client->>Service : "invoke operation"
Service->>PM : "build(workflow_name)"
PM-->>Service : "list[WorkflowStep]"
Service->>Runner : "run(workflow_name, steps, initial_state)"
Runner->>Steps : "execute steps with interceptors"
loop "for each step"
Steps->>Step : "run(state, step_context)"
Step-->>Steps : "updated state"
end
Steps-->>Runner : "final state"
Runner-->>Service : "final state"
Service-->>Client : "result"
```

**Diagram sources**
- [service.py](file://src/memu/app/service.py#L350-L360)
- [pipeline.py](file://src/memu/workflow/pipeline.py#L47-L49)
- [runner.py](file://src/memu/workflow/runner.py#L28-L39)
- [step.py](file://src/memu/workflow/step.py#L50-L101)

## Detailed Component Analysis

### PipelineManager
- Responsibilities:
  - Register pipelines with initial state keys.
  - Build copies of the latest pipeline revision for execution.
  - Mutate pipelines (configure step, insert before/after, replace, remove) with validation.
  - Enforce uniqueness of step IDs, capability availability, LLM profile validity, and state key dependencies.
  - Revisioning: each mutation creates a new revision with incremented revision number and timestamp.
- Key behaviors:
  - Validation ensures requires keys are satisfied by prior steps’ produces.
  - Capability checks against available capabilities set.
  - LLM profile validation against configured profiles.

```mermaid
classDiagram
class PipelineManager {
+available_capabilities : set
+llm_profiles : set
+register(name, steps, initial_state_keys)
+build(name) list[WorkflowStep]
+config_step(name, step_id, configs) int
+insert_after(name, target, new_step) int
+insert_before(name, target, new_step) int
+replace_step(name, target, new_step) int
+remove_step(name, target) int
+revision_token() str
-_mutate(name, mutator) int
-_current_revision(name) PipelineRevision
-_validate_steps(steps, initial_state_keys)
}
class PipelineRevision {
+name : str
+revision : int
+steps : list[WorkflowStep]
+created_at : float
+metadata : dict
}
PipelineManager --> PipelineRevision : "manages"
```

**Diagram sources**
- [pipeline.py](file://src/memu/workflow/pipeline.py#L21-L171)

**Section sources**
- [pipeline.py](file://src/memu/workflow/pipeline.py#L27-L122)

### WorkflowStep
- Defines a single unit of work with:
  - step_id: unique identifier within a pipeline.
  - role: semantic role for categorization.
  - handler: async callable(state, context) -> state.
  - requires/produces: state key contracts.
  - capabilities: required backend capabilities.
  - config: step-level configuration (e.g., LLM profiles).
- Execution:
  - run() invokes handler and validates return type is a mapping.

```mermaid
flowchart TD
Start(["Step.run(state, context)"]) --> Call["Invoke handler(state, context)"]
Call --> Await{"Awaitable?"}
Await --> |Yes| Wait["await handler(...)"]
Await --> |No| Skip["use result"]
Wait --> Coerce["Coerce to dict"]
Skip --> Coerce
Coerce --> Validate{"isinstance(..., Mapping)?"}
Validate --> |No| Raise["raise TypeError"]
Validate --> |Yes| Return["return dict(result)"]
```

**Diagram sources**
- [step.py](file://src/memu/workflow/step.py#L40-L47)

**Section sources**
- [step.py](file://src/memu/workflow/step.py#L16-L48)

### run_steps and Interceptors
- run_steps orchestrates step execution with:
  - Pre-step before interceptors.
  - Step execution with state validation.
  - Post-step after interceptors.
  - On-error interceptors if a step raises.
- Interceptors:
  - Registered via WorkflowInterceptorRegistry.
  - Snapshot taken per execution to capture current registrations.
  - Strict mode controls whether interceptor exceptions propagate or are logged.

```mermaid
sequenceDiagram
participant Exec as "run_steps"
participant Reg as "InterceptorRegistry"
participant Before as "Before Interceptors"
participant Step as "Step.run"
participant After as "After Interceptors"
participant Err as "On-error Interceptors"
Exec->>Reg : "snapshot()"
Reg-->>Exec : "before/after/on_error tuples"
Exec->>Before : "invoke(step_ctx, state)"
Exec->>Step : "run(state, step_context)"
alt "exception"
Exec->>Err : "invoke(step_ctx, state, error)"
Err-->>Exec : "handled or re-raise"
else "success"
Exec->>After : "invoke(step_ctx, state)"
end
```

**Diagram sources**
- [step.py](file://src/memu/workflow/step.py#L50-L101)
- [interceptor.py](file://src/memu/workflow/interceptor.py#L163-L219)

**Section sources**
- [step.py](file://src/memu/workflow/step.py#L50-L101)
- [interceptor.py](file://src/memu/workflow/interceptor.py#L56-L219)

### WorkflowRunner
- Protocol defines run(workflow_name, steps, initial_state, context, interceptor_registry) -> state.
- LocalWorkflowRunner delegates to run_steps.
- External runners can be registered via register_workflow_runner and resolved by name.

```mermaid
classDiagram
class WorkflowRunner {
<<protocol>>
+name : str
+run(workflow_name, steps, initial_state, context, registry) WorkflowState
}
class LocalWorkflowRunner {
+name = "local"
+run(...)
}
WorkflowRunner <|.. LocalWorkflowRunner
```

**Diagram sources**
- [runner.py](file://src/memu/workflow/runner.py#L12-L49)

**Section sources**
- [runner.py](file://src/memu/workflow/runner.py#L12-L82)

### MemoryService Integration and Pipeline Registration
- MemoryService composes:
  - LLM clients and interceptors.
  - Database and blob storage.
  - PipelineManager with available capabilities and LLM profiles.
  - Registers pipelines for memorize, retrieve (RAG and LLM variants), and CRUD operations.
- Exposes mutation APIs:
  - configure_pipeline(step_id, configs, pipeline)
  - insert_step_after/before(target_step_id, new_step, pipeline)
  - replace_step(target_step_id, new_step, pipeline)
  - remove_step(target_step_id, pipeline)

```mermaid
graph TB
Service["MemoryService"] --> PM["PipelineManager"]
Service --> Runner["WorkflowRunner"]
Service --> Interceptors["WorkflowInterceptorRegistry"]
Service --> Mixins["Memorize/Retrieve/CRUD Mixins"]
PM --> Steps["WorkflowStep[]"]
Runner --> Exec["run_steps()"]
```

**Diagram sources**
- [service.py](file://src/memu/app/service.py#L49-L427)
- [memorize.py](file://src/memu/app/memorize.py#L97-L166)
- [retrieve.py](file://src/memu/app/retrieve.py#L106-L210)
- [crud.py](file://src/memu/app/crud.py#L100-L148)

**Section sources**
- [service.py](file://src/memu/app/service.py#L91-L95)
- [service.py](file://src/memu/app/service.py#L315-L348)
- [service.py](file://src/memu/app/service.py#L390-L426)

### Step-Based Architecture for Memory Operations
- Ingestion (memorize): ingest resource → preprocess multimodal → extract items → categorize and persist → build response.
- Retrieval (retrieve): route intention → route category → sufficiency check → recall items → sufficiency check → recall resources → build context.
- CRUD: list, create, update, delete memory items and categories.

```mermaid
flowchart TD
A["memorize"] --> A1["ingest_resource"]
A1 --> A2["preprocess_multimodal"]
A2 --> A3["extract_items"]
A3 --> A4["dedupe_merge"]
A4 --> A5["categorize_items"]
A5 --> A6["persist_index"]
A6 --> A7["build_response"]
B["retrieve_rag"] --> B1["route_intention"]
B1 --> B2["route_category"]
B2 --> B3["sufficiency_after_category"]
B3 --> B4["recall_items"]
B4 --> B5["sufficiency_after_items"]
B5 --> B6["recall_resources"]
B6 --> B7["build_context"]
C["retrieve_llm"] --> C1["route_intention"]
C1 --> C2["route_category"]
C2 --> C3["sufficiency_after_category"]
C3 --> C4["recall_items"]
C4 --> C5["sufficiency_after_items"]
C5 --> C6["recall_resources"]
C6 --> C7["build_context"]
```

**Diagram sources**
- [memorize.py](file://src/memu/app/memorize.py#L97-L166)
- [retrieve.py](file://src/memu/app/retrieve.py#L106-L210)
- [retrieve.py](file://src/memu/app/retrieve.py#L454-L536)

**Section sources**
- [memorize.py](file://src/memu/app/memorize.py#L97-L325)
- [retrieve.py](file://src/memu/app/retrieve.py#L106-L723)
- [crud.py](file://src/memu/app/crud.py#L100-L245)

### Examples of Workflow Construction and Mutation
- Constructing a workflow:
  - Define a list of WorkflowStep with handler functions and state contracts.
  - Register with PipelineManager and specify initial state keys.
- Inserting/removing steps:
  - Use insert_after/insert_before/replace_step/remove_step to mutate pipelines.
  - PipelineManager enforces validation on each mutation.
- Configuring steps:
  - Use config_step to merge step-level config (e.g., LLM profiles).

Practical usage patterns are visible in:
- Pipeline registration in MemoryService.
- Workflow builders in MemorizeMixin and RetrieveMixin.
- Mutation APIs exposed by MemoryService.

**Section sources**
- [service.py](file://src/memu/app/service.py#L315-L348)
- [memorize.py](file://src/memu/app/memorize.py#L97-L166)
- [retrieve.py](file://src/memu/app/retrieve.py#L106-L210)
- [service.py](file://src/memu/app/service.py#L390-L426)

## Dependency Analysis
- PipelineManager depends on WorkflowStep and enforces validation across steps.
- run_steps depends on WorkflowInterceptorRegistry and WorkflowStep.
- WorkflowRunner depends on run_steps.
- MemoryService composes all pieces and registers pipelines.

```mermaid
graph LR
Step["WorkflowStep"] --> RunSteps["run_steps"]
Interceptor["WorkflowInterceptorRegistry"] --> RunSteps
RunSteps --> Runner["WorkflowRunner"]
Runner --> Service["MemoryService"]
Service --> PM["PipelineManager"]
PM --> Step
```

**Diagram sources**
- [step.py](file://src/memu/workflow/step.py#L50-L101)
- [runner.py](file://src/memu/workflow/runner.py#L28-L39)
- [interceptor.py](file://src/memu/workflow/interceptor.py#L163-L219)
- [service.py](file://src/memu/app/service.py#L350-L360)
- [pipeline.py](file://src/memu/workflow/pipeline.py#L47-L49)

**Section sources**
- [pipeline.py](file://src/memu/workflow/pipeline.py#L131-L165)
- [step.py](file://src/memu/workflow/step.py#L50-L101)
- [runner.py](file://src/memu/workflow/runner.py#L28-L39)
- [interceptor.py](file://src/memu/workflow/interceptor.py#L163-L219)
- [service.py](file://src/memu/app/service.py#L350-L360)

## Performance Considerations
- Asynchronous execution: Handlers are awaited; use async I/O for LLM and vector operations to minimize blocking.
- Step-level concurrency: The current run_steps executes steps sequentially. To enable parallelism, introduce fan-out/fan-in patterns at the workflow level (e.g., parallelize independent branches) while preserving state guarantees.
- Interceptor overhead: Interceptors are invoked per step; keep them lightweight and avoid heavy synchronous operations.
- LLM profile selection: Step config supports selecting LLM profiles; choose appropriate models and batch sizes to balance latency and cost.
- Vector search: Retrieval steps rely on vector similarity; tune top_k and ranking strategies to reduce downstream processing.

[No sources needed since this section provides general guidance]

## Troubleshooting Guide
Common issues and resolutions:
- Missing required state keys: run_steps validates requires against current state; ensure upstream steps produce the required keys.
- Unknown step ID: PipelineManager raises KeyError when mutating steps that do not exist.
- Duplicate step IDs: PipelineManager enforces uniqueness within a pipeline.
- Unavailable capabilities: PipelineManager validates step capabilities against available capabilities set.
- Unknown LLM profile: PipelineManager validates step config against configured profiles.
- Interceptor errors: In non-strict mode, interceptor exceptions are logged; switch to strict mode to surface errors immediately.

Operational tips:
- Use interceptors to log step_context and state snapshots for debugging.
- Leverage revision_token to detect pipeline changes across deployments.
- Validate pipeline mutations locally before deploying to production.

**Section sources**
- [step.py](file://src/memu/workflow/step.py#L68-L72)
- [pipeline.py](file://src/memu/workflow/pipeline.py#L131-L165)
- [interceptor.py](file://src/memu/workflow/interceptor.py#L205-L219)

## Conclusion
memU’s workflow pipeline architecture provides a modular, extensible, and observable execution model for memory operations. PipelineManager centralizes registration and mutation with strong validation, WorkflowStep encapsulates discrete units of work with explicit state contracts, and WorkflowRunner offers a pluggable execution backend. The interceptor system adds instrumentation and control hooks, while MemoryService composes these pieces and exposes runtime customization APIs. This design enables teams to tailor memory processing workflows to their needs, safely evolve pipelines, and instrument execution for observability and debugging.