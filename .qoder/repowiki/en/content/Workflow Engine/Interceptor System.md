# Interceptor System

<cite>
**Referenced Files in This Document**
- [interceptor.py](file://src/memu/workflow/interceptor.py)
- [step.py](file://src/memu/workflow/step.py)
- [runner.py](file://src/memu/workflow/runner.py)
- [pipeline.py](file://src/memu/workflow/pipeline.py)
- [service.py](file://src/memu/app/service.py)
- [architecture.md](file://docs/architecture.md)
- [__init__.py](file://src/memu/workflow/__init__.py)
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
This document explains the workflow interceptor system that enables observation and modification of workflow execution. It covers how interceptors are registered, where they execute in the workflow lifecycle, and how execution context is managed. It also documents interceptor types, execution order, error handling, and practical guidance for implementing, debugging, and extending interceptors while maintaining compatibility.

The workflow interceptor system is distinct from LLM interceptors and operates around each workflow step, providing before, after, and on-error hooks. Interceptors receive a step context and current workflow state, allowing them to read and modify state, log, enforce policies, or short-circuit execution when appropriate.

## Project Structure
The interceptor system lives under the workflow package and integrates with the pipeline manager, runner, and service layer.

```mermaid
graph TB
subgraph "Workflow Package"
I["interceptor.py<br/>Registry, handles, invocation"]
S["step.py<br/>WorkflowStep, run_steps"]
R["runner.py<br/>WorkflowRunner protocol, LocalWorkflowRunner"]
P["pipeline.py<br/>PipelineManager, PipelineRevision"]
end
subgraph "Application Layer"
SVC["service.py<br/>MemoryService wiring"]
end
SVC --> R
R --> S
S --> I
SVC --> P
```

**Diagram sources**
- [interceptor.py](file://src/memu/workflow/interceptor.py#L56-L219)
- [step.py](file://src/memu/workflow/step.py#L50-L102)
- [runner.py](file://src/memu/workflow/runner.py#L28-L82)
- [pipeline.py](file://src/memu/workflow/pipeline.py#L21-L171)
- [service.py](file://src/memu/app/service.py#L350-L360)

**Section sources**
- [interceptor.py](file://src/memu/workflow/interceptor.py#L56-L219)
- [step.py](file://src/memu/workflow/step.py#L50-L102)
- [runner.py](file://src/memu/workflow/runner.py#L28-L82)
- [pipeline.py](file://src/memu/workflow/pipeline.py#L21-L171)
- [service.py](file://src/memu/app/service.py#L350-L360)
- [architecture.md](file://docs/architecture.md#L64-L71)

## Core Components
- WorkflowInterceptorRegistry: Central registry for before, after, and on-error interceptors. Supports thread-safe registration and removal, and snapshots for deterministic execution.
- WorkflowInterceptorHandle: Disposable handle to remove a registered interceptor by its internal ID.
- WorkflowStepContext: Immutable context passed to interceptors containing workflow name, step identity, role, and step-scoped context.
- Execution helpers: run_before_interceptors, run_after_interceptors, run_on_error_interceptors, and a safe invocation utility.

Key behaviors:
- Registration order determines execution order for before interceptors and reverse order for after/on_error interceptors.
- Strict mode controls whether interceptor exceptions propagate or are logged.
- Interceptors receive (step_context, state) for before/after, and (step_context, state, error) for on_error.

**Section sources**
- [interceptor.py](file://src/memu/workflow/interceptor.py#L56-L219)
- [step.py](file://src/memu/workflow/step.py#L50-L102)
- [runner.py](file://src/memu/workflow/runner.py#L28-L82)
- [service.py](file://src/memu/app/service.py#L258-L295)

## Architecture Overview
The interceptor system participates in the workflow execution loop around each step. The runner resolves a workflow backend, builds steps from the pipeline, and delegates execution to run_steps. During each step, the system constructs a step context and invokes interceptors before, after, and on error.

```mermaid
sequenceDiagram
participant Client as "Caller"
participant Runner as "LocalWorkflowRunner"
participant Steps as "run_steps"
participant Reg as "WorkflowInterceptorRegistry"
participant Intc as "Interceptors"
participant Step as "WorkflowStep"
Client->>Runner : run(workflow_name, steps, initial_state, context, registry)
Runner->>Steps : run_steps(...)
Steps->>Reg : snapshot()
loop For each step
Steps->>Steps : build step_context
Steps->>Intc : run_before_interceptors(snapshot.before)
Steps->>Step : run(state, step_context)
alt Exception
Steps->>Intc : run_on_error_interceptors(snapshot.on_error)
Steps-->>Runner : re-raise
else Success
Steps->>Intc : run_after_interceptors(snapshot.after)
end
end
Steps-->>Runner : final state
Runner-->>Client : state
```

**Diagram sources**
- [runner.py](file://src/memu/workflow/runner.py#L28-L40)
- [step.py](file://src/memu/workflow/step.py#L50-L102)
- [interceptor.py](file://src/memu/workflow/interceptor.py#L168-L219)

## Detailed Component Analysis

### WorkflowInterceptorRegistry
Responsibilities:
- Thread-safe registration of before/after/on_error interceptors.
- Deterministic snapshot capture for a consistent execution view during a single run.
- Removal of interceptors by ID.
- Strict mode toggle affecting exception propagation.

Execution semantics:
- Before: invoked in registration order.
- After: invoked in reverse registration order.
- On-error: invoked in reverse registration order.

```mermaid
classDiagram
class WorkflowInterceptorRegistry {
-before : tuple
-after : tuple
-on_error : tuple
-lock
-seq : int
-strict : bool
+register_before(fn, name)
+register_after(fn, name)
+register_on_error(fn, name)
+remove(interceptor_id) bool
+snapshot() _WorkflowInterceptorSnapshot
+strict bool
}
class WorkflowInterceptorHandle {
-registry : WorkflowInterceptorRegistry
-interceptor_id : int
-disposed : bool
+dispose() bool
}
class _WorkflowInterceptor {
+interceptor_id : int
+fn : Callable
+name : str?
}
WorkflowInterceptorRegistry --> WorkflowInterceptorHandle : "creates"
WorkflowInterceptorRegistry --> _WorkflowInterceptor : "stores"
```

**Diagram sources**
- [interceptor.py](file://src/memu/workflow/interceptor.py#L56-L166)

**Section sources**
- [interceptor.py](file://src/memu/workflow/interceptor.py#L56-L166)

### Execution Helpers and Safe Invocation
- run_before_interceptors: iterates forward over snapshot.before.
- run_after_interceptors: iterates backward over snapshot.after.
- run_on_error_interceptors: iterates backward over snapshot.on_error.
- _safe_invoke_interceptor: wraps interceptor invocation, awaiting coroutines and handling exceptions according to strict mode.

```mermaid
flowchart TD
Start(["Invoke Interceptor"]) --> Try["Call fn(*args)"]
Try --> Await{"Result is awaitable?"}
Await --> |Yes| DoAwait["await result"]
Await --> |No| Done["Done"]
DoAwait --> Done
Done --> Catch{"Exception?"}
Catch --> |No| End(["Return"])
Catch --> |Yes| Strict{"strict == True?"}
Strict --> |Yes| ReRaise["raise"]
Strict --> |No| Log["log exception"] --> End
```

**Diagram sources**
- [interceptor.py](file://src/memu/workflow/interceptor.py#L205-L219)

**Section sources**
- [interceptor.py](file://src/memu/workflow/interceptor.py#L168-L219)

### WorkflowStepContext and run_steps Integration
- run_steps constructs a step_context from the workflow context and step metadata, then builds a WorkflowStepContext for each step.
- Interceptors are executed around step.run, with error handling and reverse-order after/on_error execution.

```mermaid
sequenceDiagram
participant Steps as "run_steps"
participant Ctx as "WorkflowStepContext"
participant Before as "Before Interceptors"
participant Step as "WorkflowStep.run"
participant After as "After Interceptors"
participant Err as "On-error Interceptors"
Steps->>Ctx : build step_context
Steps->>Before : run_before_interceptors
Steps->>Step : run(state, step_context)
alt success
Steps->>After : run_after_interceptors
else exception
Steps->>Err : run_on_error_interceptors
Steps-->>Steps : re-raise
end
```

**Diagram sources**
- [step.py](file://src/memu/workflow/step.py#L50-L102)
- [interceptor.py](file://src/memu/workflow/interceptor.py#L168-L219)

**Section sources**
- [step.py](file://src/memu/workflow/step.py#L50-L102)

### Service Integration and Public API
- MemoryService exposes convenience methods to register workflow interceptors and passes the registry to the runner.
- The runner receives the registry and forwards it to run_steps, ensuring interceptors participate in every step.

```mermaid
graph LR
SVC["MemoryService"] -- "register_*" --> REG["WorkflowInterceptorRegistry"]
SVC -- "run(..., interceptor_registry=REG)" --> RUN["LocalWorkflowRunner"]
RUN -- "run_steps(..., registry)" --> STEP["run_steps"]
STEP -- "snapshot(), invoke" --> INT["Interceptors"]
```

**Diagram sources**
- [service.py](file://src/memu/app/service.py#L258-L295)
- [runner.py](file://src/memu/workflow/runner.py#L28-L40)
- [step.py](file://src/memu/workflow/step.py#L50-L102)
- [interceptor.py](file://src/memu/workflow/interceptor.py#L163-L166)

**Section sources**
- [service.py](file://src/memu/app/service.py#L258-L295)
- [runner.py](file://src/memu/workflow/runner.py#L28-L40)
- [step.py](file://src/memu/workflow/step.py#L50-L102)

## Dependency Analysis
- Registry depends on immutable tuples to maintain deterministic iteration order and uses a lock for thread safety.
- run_steps depends on the registry’s snapshot to avoid dynamic changes mid-run.
- Runner delegates execution to run_steps and optionally passes the registry.
- Service composes the registry and passes it to the runner.

```mermaid
graph TB
REG["WorkflowInterceptorRegistry"] --> SNAP["_WorkflowInterceptorSnapshot"]
REG --> INTL["_WorkflowInterceptor"]
RUN["LocalWorkflowRunner"] --> RS["run_steps"]
RS --> REG
RS --> STEP["WorkflowStep"]
SVC["MemoryService"] --> RUN
SVC --> REG
```

**Diagram sources**
- [interceptor.py](file://src/memu/workflow/interceptor.py#L56-L166)
- [step.py](file://src/memu/workflow/step.py#L50-L102)
- [runner.py](file://src/memu/workflow/runner.py#L28-L40)
- [service.py](file://src/memu/app/service.py#L350-L360)

**Section sources**
- [interceptor.py](file://src/memu/workflow/interceptor.py#L56-L166)
- [step.py](file://src/memu/workflow/step.py#L50-L102)
- [runner.py](file://src/memu/workflow/runner.py#L28-L40)
- [service.py](file://src/memu/app/service.py#L350-L360)

## Performance Considerations
- Interceptors are synchronous or awaited per invocation; keep logic lightweight to avoid slowing step execution.
- Snapshot captures a fixed view of interceptors; frequent registration/removal mid-run is not supported.
- Reverse-order after/on_error execution implies last-in-first-out semantics; consider interceptor ordering carefully.
- Strict mode disables exception suppression, which can reduce resilience but improves visibility during development.

[No sources needed since this section provides general guidance]

## Troubleshooting Guide
Common issues and remedies:
- Interceptor not invoked:
  - Verify registration via the public service methods and confirm the registry is passed to the runner.
  - Ensure the step context and state are valid; run_steps validates required keys before invoking interceptors.
- Exceptions in interceptors:
  - In non-strict mode, exceptions are logged; switch to strict mode during debugging to surface errors immediately.
  - Use the disposable handle to remove problematic interceptors.
- Conflicting interceptors:
  - Leverage reverse-order execution for cleanup-like after handlers.
  - Use step-scoped context to coordinate behavior across interceptors.
- Monitoring:
  - Add structured logging inside interceptors using the provided logger.
  - Attach correlation IDs to step_context to trace end-to-end execution.

**Section sources**
- [interceptor.py](file://src/memu/workflow/interceptor.py#L205-L219)
- [step.py](file://src/memu/workflow/step.py#L69-L72)
- [service.py](file://src/memu/app/service.py#L258-L295)

## Conclusion
The workflow interceptor system offers a simple, deterministic way to observe and modify step-level behavior. By registering before/after/on-error hooks and leveraging the provided execution helpers, developers can implement cross-cutting concerns such as auditing, validation, metrics, and error handling. The system’s design prioritizes clarity and ease of use, with strict mode and snapshots supporting robust operation in production.

[No sources needed since this section summarizes without analyzing specific files]

## Appendices

### Interceptor Types and Execution Order
- Before: executed in registration order.
- After: executed in reverse registration order.
- On-error: executed in reverse registration order when a step raises an exception.

**Section sources**
- [interceptor.py](file://src/memu/workflow/interceptor.py#L168-L219)

### Implementation Examples (by reference)
- Registering a before interceptor:
  - See [service.py](file://src/memu/app/service.py#L258-L269)
- Registering an after interceptor:
  - See [service.py](file://src/memu/app/service.py#L271-L282)
- Registering an on-error interceptor:
  - See [service.py](file://src/memu/app/service.py#L284-L295)
- Passing registry to runner:
  - See [service.py](file://src/memu/app/service.py#L350-L360)
- Exported API:
  - See [__init__.py](file://src/memu/workflow/__init__.py#L1-L30)

### Relationship to LLM Interceptors
- LLM interceptors support filtering, priority, and ordering; workflow interceptors do not.
- LLM interceptors operate around LLM calls; workflow interceptors operate around workflow steps.

**Section sources**
- [architecture.md](file://docs/architecture.md#L64-L71)
- [interceptor.py](file://src/memu/workflow/interceptor.py#L57-L63)