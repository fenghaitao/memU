# Retrieve Configuration

<cite>
**Referenced Files in This Document**
- [retrieve.py](file://src/memu/app/retrieve.py)
- [settings.py](file://src/memu/app/settings.py)
- [pre_retrieval_decision.py](file://src/memu/prompts/retrieve/pre_retrieval_decision.py)
- [query_rewriter.py](file://src/memu/prompts/retrieve/query_rewriter.py)
- [query_rewriter_judger.py](file://src/memu/prompts/retrieve/query_rewriter_judger.py)
- [judger.py](file://src/memu/prompts/retrieve/judger.py)
- [llm_category_ranker.py](file://src/memu/prompts/retrieve/llm_category_ranker.py)
- [llm_item_ranker.py](file://src/memu/prompts/retrieve/llm_item_ranker.py)
- [llm_resource_ranker.py](file://src/memu/prompts/retrieve/llm_resource_ranker.py)
- [pipeline.py](file://src/memu/workflow/pipeline.py)
- [step.py](file://src/memu/workflow/step.py)
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
This document explains how to configure and optimize retrieval in the system. It focuses on the RetrieveConfig structure, the dual-mode retrieval system (RAG vs LLM-based ranking), query processing settings, and retrieval parameters. It also covers query rewriting, intent detection, and context filtering, and provides practical guidance for tuning retrieval across conversational retrieval, semantic search, and proactive context loading scenarios.

## Project Structure
The retrieval system is implemented as a mixin that integrates with a workflow engine. Configuration is centralized in a Pydantic model, while prompts define the behavior of query rewriting, intent detection, and LLM-based ranking.

```mermaid
graph TB
subgraph "App Layer"
A["RetrieveMixin<br/>retrieve()"]
B["RetrieveConfig<br/>settings.py"]
end
subgraph "Prompts"
C["pre_retrieval_decision.py"]
D["query_rewriter.py"]
E["query_rewriter_judger.py"]
F["judger.py"]
G["llm_category_ranker.py"]
H["llm_item_ranker.py"]
I["llm_resource_ranker.py"]
end
subgraph "Workflow"
J["step.py"]
K["pipeline.py"]
end
A --> B
A --> C
A --> D
A --> E
A --> F
A --> G
A --> H
A --> I
A --> J
A --> K
```

**Diagram sources**
- [retrieve.py](file://src/memu/app/retrieve.py#L42-L85)
- [settings.py](file://src/memu/app/settings.py#L175-L202)
- [pre_retrieval_decision.py](file://src/memu/prompts/retrieve/pre_retrieval_decision.py#L1-L54)
- [query_rewriter.py](file://src/memu/prompts/retrieve/query_rewriter.py#L1-L45)
- [query_rewriter_judger.py](file://src/memu/prompts/retrieve/query_rewriter_judger.py#L1-L49)
- [judger.py](file://src/memu/prompts/retrieve/judger.py#L1-L40)
- [llm_category_ranker.py](file://src/memu/prompts/retrieve/llm_category_ranker.py#L1-L36)
- [llm_item_ranker.py](file://src/memu/prompts/retrieve/llm_item_ranker.py#L1-L41)
- [llm_resource_ranker.py](file://src/memu/prompts/retrieve/llm_resource_ranker.py#L1-L41)
- [step.py](file://src/memu/workflow/step.py#L16-L48)
- [pipeline.py](file://src/memu/workflow/pipeline.py#L21-L171)

**Section sources**
- [retrieve.py](file://src/memu/app/retrieve.py#L1-L120)
- [settings.py](file://src/memu/app/settings.py#L175-L202)

## Core Components
- RetrieveConfig: Central configuration for retrieval behavior, including method selection, routing, sufficiency checks, and per-stage retrieval parameters.
- RetrieveMixin: Implements the retrieve() entry point and orchestrates the dual-mode retrieval workflow.
- Prompts: Define query rewriting, intent detection, and LLM-based ranking behaviors.
- Workflow engine: Provides typed steps and pipeline management for modular execution.

Key configuration attributes:
- method: Choose between "rag" (vector similarity + optional LLM sufficiency checks) and "llm" (LLM-driven search and ranking).
- route_intention: Enable/disable intent detection and query rewriting.
- category/item/resource: Per-stage toggles and top_k controls.
- sufficiency_check and sufficiency_check_prompt/sufficiency_check_llm_profile: Configure iterative sufficiency checks and LLM profile for judgment.
- llm_ranking_llm_profile: LLM profile for LLM-based ranking steps.

**Section sources**
- [settings.py](file://src/memu/app/settings.py#L146-L202)
- [retrieve.py](file://src/memu/app/retrieve.py#L42-L85)

## Architecture Overview
The retrieval system supports two execution modes controlled by RetrieveConfig.method:

- RAG mode: Uses vector similarity for category and item recall, cosine similarity for resource recall, and optional LLM-based sufficiency checks between stages.
- LLM mode: Delegates search and ranking to LLM prompts per stage, with optional sufficiency checks.

```mermaid
sequenceDiagram
participant Client as "Caller"
participant Mixin as "RetrieveMixin"
participant WF as "Workflow Engine"
participant Step as "WorkflowStep"
participant LLM as "LLM Client"
participant Embed as "Embedding Client"
Client->>Mixin : retrieve(queries, where)
Mixin->>Mixin : build state (method, queries, filters)
Mixin->>WF : run_workflow(method == "rag" ? "retrieve_rag" : "retrieve_llm", state)
loop For each step
WF->>Step : run(state, step_config)
alt route_intention
Step->>LLM : decide_if_retrieval_needed()
LLM-->>Step : needs_retrieval, rewritten_query
end
alt vector recall
Step->>Embed : embed(active_query)
Embed-->>Step : query_vector
Step->>Step : vector_search_items/top_k
end
alt sufficiency_check
Step->>LLM : judge sufficiency
LLM-->>Step : ENOUGH/MORE
end
end
WF-->>Mixin : final response
Mixin-->>Client : categories/items/resources + next_step_query
```

**Diagram sources**
- [retrieve.py](file://src/memu/app/retrieve.py#L42-L85)
- [retrieve.py](file://src/memu/app/retrieve.py#L106-L210)
- [retrieve.py](file://src/memu/app/retrieve.py#L454-L536)
- [step.py](file://src/memu/workflow/step.py#L40-L101)
- [pipeline.py](file://src/memu/workflow/pipeline.py#L47-L123)

## Detailed Component Analysis

### RetrieveConfig Structure
RetrieveConfig governs the entire retrieval pipeline. It includes:
- method: "rag" or "llm".
- route_intention: Whether to detect intent and rewrite queries.
- category/item/resource: Feature flags and top_k per stage.
- sufficiency_check: Enable iterative sufficiency checks after each stage.
- sufficiency_check_prompt and sufficiency_check_llm_profile: Prompt and LLM profile for sufficiency decisions.
- llm_ranking_llm_profile: LLM profile for LLM-based ranking steps.

Per-stage configuration:
- category.enabled and category.top_k
- item.enabled, item.top_k, item.use_category_references, item.ranking, item.recency_decay_days
- resource.enabled and resource.top_k

**Section sources**
- [settings.py](file://src/memu/app/settings.py#L146-L202)

### Dual-Mode Retrieval System
- RAG mode:
  - Intent detection and query rewriting via LLM when enabled.
  - Category recall uses summary embeddings and cosine_topk.
  - Item recall uses vector_search_items with configurable ranking and recency decay.
  - Resource recall uses cosine similarity over stored embeddings.
  - Sufficiency checks can trigger query rewriting and re-embedding between stages.

- LLM mode:
  - Intent detection and query rewriting via LLM when enabled.
  - Category ranking uses a dedicated LLM prompt.
  - Item ranking filters items by category relevance and ranks them.
  - Resource ranking uses context from categories and items to rank resources.
  - Optional sufficiency checks after each stage.

```mermaid
flowchart TD
Start(["Start"]) --> Mode{"method == 'rag'?"}
Mode --> |Yes| RAG["RAG Pipeline"]
Mode --> |No| LLM["LLM Pipeline"]
subgraph "RAG Steps"
R1["route_intention"] --> R2["route_category"]
R2 --> R3["sufficiency_after_category"]
R3 --> R4{"proceed_to_items?"}
R4 --> |Yes| R5["recall_items"]
R4 --> |No| RBuild["build_context"]
R5 --> R6["sufficiency_after_items"]
R6 --> R7{"proceed_to_resources?"}
R7 --> |Yes| R8["recall_resources"]
R7 --> |No| RBuild
R8 --> RBuild
end
subgraph "LLM Steps"
L1["route_intention"] --> L2["route_category"]
L2 --> L3["sufficiency_after_category"]
L3 --> L4{"proceed_to_items?"}
L4 --> |Yes| L5["recall_items"]
L4 --> |No| LBuild["build_context"]
L5 --> L6["sufficiency_after_items"]
L6 --> L7{"proceed_to_resources?"}
L7 --> |Yes| L8["recall_resources"]
L7 --> |No| LBuild
L8 --> LBuild
end
RBuild --> End(["End"])
LBuild --> End
```

**Diagram sources**
- [retrieve.py](file://src/memu/app/retrieve.py#L106-L210)
- [retrieve.py](file://src/memu/app/retrieve.py#L454-L536)

**Section sources**
- [retrieve.py](file://src/memu/app/retrieve.py#L42-L85)
- [retrieve.py](file://src/memu/app/retrieve.py#L106-L210)
- [retrieve.py](file://src/memu/app/retrieve.py#L454-L536)

### Query Processing Settings
- route_intention: Enables intent detection and query rewriting before retrieval.
- skip_rewrite: Skips rewriting when processing single-turn queries.
- sufficiency_check: Iterative sufficiency checks after each stage.
- sufficiency_check_prompt and sufficiency_check_llm_profile: Customize judgment prompts and LLM profile.
- llm_ranking_llm_profile: Profile for LLM-based ranking steps.

**Section sources**
- [retrieve.py](file://src/memu/app/retrieve.py#L42-L85)
- [retrieve.py](file://src/memu/app/retrieve.py#L228-L258)
- [settings.py](file://src/memu/app/settings.py#L190-L202)

### Retrieval Parameters
- category.top_k: Number of categories to retrieve.
- item.top_k: Number of items to retrieve.
- item.use_category_references: When insufficient categories are retrieved, follow [ref:ITEM_ID] citations to fetch referenced items.
- item.ranking: "similarity" (cosine) or "salience" (reinforcement + recency).
- item.recency_decay_days: Half-life for recency decay in salience scoring.
- resource.top_k: Number of resources to retrieve.

**Section sources**
- [settings.py](file://src/memu/app/settings.py#L146-L173)
- [retrieve.py](file://src/memu/app/retrieve.py#L359-L367)

### Query Rewriting and Intent Detection
- Pre-retrieval decision prompt defines when retrieval is needed and how to rewrite queries.
- Query rewriter prompt transforms ambiguous or pronoun-heavy queries into explicit, self-contained forms.
- Combined judger prompt performs both rewriting and sufficiency judgment in one step.

```mermaid
flowchart TD
A["Original Query"] --> B["Format Conversation History"]
B --> C["Pre-retrieval Decision Prompt"]
C --> D{"Needs Retrieval?"}
D --> |No| E["Keep Original Query"]
D --> |Yes| F["Query Rewriter Prompt"]
F --> G["Rewritten Query"]
G --> H["Next Stage"]
E --> H
```

**Diagram sources**
- [retrieve.py](file://src/memu/app/retrieve.py#L746-L784)
- [pre_retrieval_decision.py](file://src/memu/prompts/retrieve/pre_retrieval_decision.py#L1-L54)
- [query_rewriter.py](file://src/memu/prompts/retrieve/query_rewriter.py#L1-L45)
- [query_rewriter_judger.py](file://src/memu/prompts/retrieve/query_rewriter_judger.py#L1-L49)

**Section sources**
- [retrieve.py](file://src/memu/app/retrieve.py#L746-L784)
- [pre_retrieval_decision.py](file://src/memu/prompts/retrieve/pre_retrieval_decision.py#L1-L54)
- [query_rewriter.py](file://src/memu/prompts/retrieve/query_rewriter.py#L1-L45)
- [query_rewriter_judger.py](file://src/memu/prompts/retrieve/query_rewriter_judger.py#L1-L49)
- [judger.py](file://src/memu/prompts/retrieve/judger.py#L1-L40)

### Context Filtering
- where filters are normalized against the user model fields and applied to each stage’s repository calls.
- Unknown filter fields raise errors to prevent invalid scopes.

**Section sources**
- [retrieve.py](file://src/memu/app/retrieve.py#L87-L104)

### LLM-Based Ranking Prompts
- Category ranking prompt: Given a query and available categories, select and rank up to top_k relevant categories.
- Item ranking prompt: Given relevant categories, select and rank up to top_k items.
- Resource ranking prompt: Given context (categories and items), select and rank up to top_k resources.

```mermaid
classDiagram
class LLM_Category_Ranker {
+format_categories_for_llm()
+rank(query, top_k, categories)
}
class LLM_Item_Ranker {
+format_items_for_llm()
+rank(query, top_k, category_ids, category_hits)
}
class LLM_Resource_Ranker {
+format_resources_for_llm()
+rank(query, top_k, category_hits, item_hits)
}
LLM_Item_Ranker --> LLM_Category_Ranker : "uses relevant categories"
LLM_Resource_Ranker --> LLM_Category_Ranker : "uses categories"
LLM_Resource_Ranker --> LLM_Item_Ranker : "uses items"
```

**Diagram sources**
- [llm_category_ranker.py](file://src/memu/prompts/retrieve/llm_category_ranker.py#L1-L36)
- [llm_item_ranker.py](file://src/memu/prompts/retrieve/llm_item_ranker.py#L1-L41)
- [llm_resource_ranker.py](file://src/memu/prompts/retrieve/llm_resource_ranker.py#L1-L41)
- [retrieve.py](file://src/memu/app/retrieve.py#L1216-L1323)

**Section sources**
- [llm_category_ranker.py](file://src/memu/prompts/retrieve/llm_category_ranker.py#L1-L36)
- [llm_item_ranker.py](file://src/memu/prompts/retrieve/llm_item_ranker.py#L1-L41)
- [llm_resource_ranker.py](file://src/memu/prompts/retrieve/llm_resource_ranker.py#L1-L41)
- [retrieve.py](file://src/memu/app/retrieve.py#L1216-L1323)

## Dependency Analysis
- RetrieveMixin depends on:
  - RetrieveConfig for behavior flags and parameters.
  - Workflow engine for step execution and pipeline registration.
  - LLM and embedding clients for intent detection, rewriting, ranking, and embeddings.
  - Database repositories for categories, items, and resources.
- Prompts are injected into the system via configuration and used by the decision and ranking steps.
- Workflow pipeline enforces capability and profile availability, ensuring steps can run with the configured LLM profiles.

```mermaid
graph LR
Config["RetrieveConfig"] --> Mixin["RetrieveMixin"]
Mixin --> WF["Workflow Engine"]
WF --> Steps["WorkflowStep"]
Steps --> LLM["LLM Client"]
Steps --> Embed["Embedding Client"]
Mixin --> DB["Database Repositories"]
Mixin --> Prompts["Prompts"]
```

**Diagram sources**
- [retrieve.py](file://src/memu/app/retrieve.py#L42-L85)
- [settings.py](file://src/memu/app/settings.py#L175-L202)
- [step.py](file://src/memu/workflow/step.py#L16-L48)
- [pipeline.py](file://src/memu/workflow/pipeline.py#L21-L171)

**Section sources**
- [pipeline.py](file://src/memu/workflow/pipeline.py#L131-L164)
- [step.py](file://src/memu/workflow/step.py#L16-L48)

## Performance Considerations
- Method selection:
  - "rag" reduces LLM calls by relying on vector similarity and optional sufficiency checks.
  - "llm" increases LLM usage but can yield more accurate rankings.
- Ranking strategy:
  - "similarity" is faster; "salience" adds computation for reinforcement and recency weighting.
  - Adjust item.recency_decay_days to balance freshness vs. stability.
- top_k tuning:
  - Increase top_k to reduce iteration count but increase latency and cost.
  - Decrease top_k to speed up and reduce cost but risk insufficient context.
- Embedding and vector index:
  - Ensure embedding client batching and model choices align with throughput targets.
  - For pgvector, ensure proper DSN and provider selection in DatabaseConfig.
- LLM profiles:
  - Use separate profiles for sufficiency checks and ranking to isolate latency and cost.
  - Prefer lower-latency models for sufficiency checks and higher-quality models for ranking.

[No sources needed since this section provides general guidance]

## Troubleshooting Guide
Common issues and resolutions:
- Missing required state keys in workflow steps:
  - Ensure previous steps produce required keys or initialize with initial_state_keys.
- Unknown LLM profile:
  - Verify llm_profile exists in LLMProfilesConfig.
- Unknown filter field:
  - Confirm where filter keys match user model fields; otherwise, raise ValueError.
- Empty or invalid query:
  - Validate query structure and content; reject empty or malformed inputs.
- Insufficient context:
  - Enable sufficiency_check and route_intention; adjust top_k and ranking strategy.

**Section sources**
- [pipeline.py](file://src/memu/workflow/pipeline.py#L131-L164)
- [retrieve.py](file://src/memu/app/retrieve.py#L87-L104)
- [retrieve.py](file://src/memu/app/retrieve.py#L811-L840)

## Conclusion
RetrieveConfig provides a flexible, dual-mode retrieval system that balances performance and accuracy. By tuning method, ranking strategy, top_k, and sufficiency checks—and by leveraging query rewriting and intent detection—you can optimize retrieval for conversational retrieval, semantic search, and proactive context loading. Use LLM profiles strategically and monitor vector index configuration to achieve the desired balance of latency, cost, and quality.

[No sources needed since this section summarizes without analyzing specific files]

## Appendices

### Configuration Examples by Scenario
- Conversational retrieval:
  - Enable route_intention and sufficiency_check.
  - Use "rag" method with moderate top_k and "similarity" ranking for speed.
  - Set sufficiency_check_llm_profile to a fast model; keep llm_ranking_llm_profile for higher quality when needed.
- Semantic search:
  - Use "rag" with "salience" ranking and tuned recency_decay_days.
  - Increase item.top_k and resource.top_k to capture nuanced context.
- Proactive context loading:
  - Disable route_intention and sufficiency_check to minimize iterations.
  - Use "rag" with larger top_k and enable item.use_category_references to expand coverage.

[No sources needed since this section provides general guidance]