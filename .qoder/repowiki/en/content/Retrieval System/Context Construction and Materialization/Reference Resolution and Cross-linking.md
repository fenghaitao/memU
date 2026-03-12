# Reference Resolution and Cross-linking

<cite>
**Referenced Files in This Document**
- [references.py](file://src/memu/utils/references.py)
- [category_with_refs.py](file://src/memu/prompts/category_summary/category_with_refs.py)
- [memorize.py](file://src/memu/app/memorize.py)
- [models.py](file://src/memu/database/models.py)
- [interfaces.py](file://src/memu/database/interfaces.py)
- [memory_item_repo.py (SQLite)](file://src/memu/database/sqlite/repositories/memory_item_repo.py)
- [memory_item_repo.py (InMemory)](file://src/memu/database/inmemory/repositories/memory_item_repo.py)
- [category_item_repo.py (SQLite)](file://src/memu/database/sqlite/repositories/category_item_repo.py)
- [category_item_repo.py (Postgres)](file://src/memu/database/postgres/repositories/category_item_repo.py)
- [category_item_repo.py (InMemory)](file://src/memu/database/inmemory/repositories/category_item_repo.py)
- [test_references.py](file://tests/test_references.py)
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
This document explains the reference resolution and cross-linking system used to extract and resolve cross-references between categories and items. It focuses on how category summaries embed inline references to memory items, how those references are extracted and resolved, and how cross-links are established and maintained. It also covers the relationship mapping between categories and items, performance considerations for large knowledge bases, and strategies for handling broken or missing references.

## Project Structure
The reference resolution system spans several modules:
- Utilities for parsing and formatting references
- Prompts that require references in category summaries
- Application logic that orchestrates reference-aware summarization and persistence
- Database models and repositories that store and query references
- Tests validating the behavior of reference extraction and formatting

```mermaid
graph TB
subgraph "Utilities"
U1["references.py<br/>Extraction, stripping, formatting, fetching"]
end
subgraph "Prompts"
P1["category_with_refs.py<br/>LLM prompt requiring [ref:ID]"]
end
subgraph "Application"
A1["memorize.py<br/>Build prompts, extract refs, persist refs"]
end
subgraph "Database Models"
M1["models.py<br/>MemoryItem, MemoryCategory, CategoryItem"]
end
subgraph "Repositories"
R1["memory_item_repo.py (SQLite/InMemory)<br/>list_items_by_ref_ids"]
R2["category_item_repo.py (SQLite/Postgres/InMemory)<br/>link_item_category, get_item_categories"]
end
U1 --> A1
P1 --> A1
A1 --> R1
A1 --> R2
R1 --> M1
R2 --> M1
```

**Diagram sources**
- [references.py](file://src/memu/utils/references.py#L1-L173)
- [category_with_refs.py](file://src/memu/prompts/category_summary/category_with_refs.py#L1-L141)
- [memorize.py](file://src/memu/app/memorize.py#L981-L1037)
- [models.py](file://src/memu/database/models.py#L76-L106)
- [memory_item_repo.py (SQLite)](file://src/memu/database/sqlite/repositories/memory_item_repo.py#L119-L151)
- [category_item_repo.py (SQLite)](file://src/memu/database/sqlite/repositories/category_item_repo.py#L84-L173)

**Section sources**
- [references.py](file://src/memu/utils/references.py#L1-L173)
- [category_with_refs.py](file://src/memu/prompts/category_summary/category_with_refs.py#L1-L141)
- [memorize.py](file://src/memu/app/memorize.py#L981-L1037)
- [models.py](file://src/memu/database/models.py#L76-L106)
- [memory_item_repo.py (SQLite)](file://src/memu/database/sqlite/repositories/memory_item_repo.py#L119-L151)
- [category_item_repo.py (SQLite)](file://src/memu/database/sqlite/repositories/category_item_repo.py#L84-L173)

## Core Components
- Reference extraction and normalization: Extracts unique item IDs from inline references in category summaries.
- Reference stripping: Removes inline references for clean display.
- Citation formatting: Converts inline references to numbered citations with a reference list.
- Reference-aware item fetching: Resolves referenced IDs to actual memory items.
- Short reference IDs: Generates short identifiers for items included in prompts and maps them back to full IDs during persistence.
- Relationship mapping: Maintains links between items and categories via a junction table.

Key implementation references:
- Extraction and helpers: [references.py](file://src/memu/utils/references.py#L20-L173)
- Prompt requiring references: [category_with_refs.py](file://src/memu/prompts/category_summary/category_with_refs.py#L1-L141)
- Short ID mapping and persistence: [memorize.py](file://src/memu/app/memorize.py#L981-L1037)
- Item and category models: [models.py](file://src/memu/database/models.py#L76-L106)
- Item lookup by ref_id: [memory_item_repo.py (SQLite)](file://src/memu/database/sqlite/repositories/memory_item_repo.py#L119-L151), [memory_item_repo.py (InMemory)](file://src/memu/database/inmemory/repositories/memory_item_repo.py#L27-L51)
- Category-item relations: [category_item_repo.py (SQLite)](file://src/memu/database/sqlite/repositories/category_item_repo.py#L84-L173), [category_item_repo.py (Postgres)](file://src/memu/database/postgres/repositories/category_item_repo.py#L35-L87), [category_item_repo.py (InMemory)](file://src/memu/database/inmemory/repositories/category_item_repo.py#L24-L42)

**Section sources**
- [references.py](file://src/memu/utils/references.py#L20-L173)
- [category_with_refs.py](file://src/memu/prompts/category_summary/category_with_refs.py#L1-L141)
- [memorize.py](file://src/memu/app/memorize.py#L981-L1037)
- [models.py](file://src/memu/database/models.py#L76-L106)
- [memory_item_repo.py (SQLite)](file://src/memu/database/sqlite/repositories/memory_item_repo.py#L119-L151)
- [memory_item_repo.py (InMemory)](file://src/memu/database/inmemory/repositories/memory_item_repo.py#L27-L51)
- [category_item_repo.py (SQLite)](file://src/memu/database/sqlite/repositories/category_item_repo.py#L84-L173)
- [category_item_repo.py (Postgres)](file://src/memu/database/postgres/repositories/category_item_repo.py#L35-L87)
- [category_item_repo.py (InMemory)](file://src/memu/database/inmemory/repositories/category_item_repo.py#L24-L42)

## Architecture Overview
The reference resolution pipeline integrates LLM-driven category summarization with database-backed reference tracking and cross-linking.

```mermaid
sequenceDiagram
participant User as "Caller"
participant App as "Memorize App"
participant LLM as "LLM Client"
participant DB as "Database Store"
User->>App : "Provide new memories per category"
App->>App : "_build_category_summary_prompt()<br/>Enable refs if configured"
App->>LLM : "Chat with category summary prompt"
LLM-->>App : "Updated category summary with [ref : ID]"
App->>App : "_extract_refs_from_summaries()"
App->>App : "_build_item_ref_id() for each item"
App->>DB : "Persist item.extra.ref_id for referenced items"
App-->>User : "Updated summaries and persisted references"
```

**Diagram sources**
- [memorize.py](file://src/memu/app/memorize.py#L1038-L1139)
- [category_with_refs.py](file://src/memu/prompts/category_summary/category_with_refs.py#L1-L141)
- [references.py](file://src/memu/utils/references.py#L20-L48)

## Detailed Component Analysis

### Reference Extraction and Parsing
The system uses a regular expression pattern to detect inline references of the form [ref:ID] and supports comma-separated IDs within a single reference. It ensures uniqueness and strips whitespace.

```mermaid
flowchart TD
Start(["Text Input"]) --> CheckEmpty{"Text empty?"}
CheckEmpty --> |Yes| ReturnEmpty["Return []"]
CheckEmpty --> |No| Iterate["Iterate matches of [ref:IDs]"]
Iterate --> SplitIDs["Split IDs by ',' and strip"]
SplitIDs --> Dedup{"Already seen?"}
Dedup --> |Yes| NextMatch["Next match"]
Dedup --> |No| Append["Append to result and mark seen"]
Append --> NextMatch
NextMatch --> Done{"More matches?"}
Done --> |Yes| Iterate
Done --> |No| ReturnRes["Return unique ordered IDs"]
```

**Diagram sources**
- [references.py](file://src/memu/utils/references.py#L20-L48)

**Section sources**
- [references.py](file://src/memu/utils/references.py#L20-L48)

### Reference Stripping and Citation Formatting
- Stripping removes inline references and normalizes spacing/punctuation for clean display.
- Citation formatting converts inline references to numbered citations and appends a reference list.

```mermaid
flowchart TD
S0["Input text"] --> S1["Extract references (unique order)"]
S1 --> S2{"Any refs?"}
S2 --> |No| S3["Return original text"]
S2 --> |Yes| S4["Build ID->number map"]
S4 --> S5["Replace [ref:ID,...] with [n,m,...]"]
S5 --> S6["Append 'References:\\n[n] ID' per ref"]
S6 --> S7["Return formatted text"]
```

**Diagram sources**
- [references.py](file://src/memu/utils/references.py#L77-L115)

**Section sources**
- [references.py](file://src/memu/utils/references.py#L77-L115)

### Reference-Aware Category Summary Generation
The application builds a category summary prompt that requires inline references for new information. It also constructs a short reference map for the LLM to choose from.

```mermaid
sequenceDiagram
participant App as "Memorize App"
participant Prompt as "Prompt Builder"
participant LLM as "LLM Client"
App->>Prompt : "Build category summary prompt"
Prompt->>Prompt : "Enable refs if configured"
Prompt->>Prompt : "Build short ref map : [ref : short] summary"
Prompt-->>App : "Formatted prompt"
App->>LLM : "Chat with prompt"
LLM-->>App : "Summary with [ref : ID]"
```

**Diagram sources**
- [memorize.py](file://src/memu/app/memorize.py#L1038-L1098)
- [category_with_refs.py](file://src/memu/prompts/category_summary/category_with_refs.py#L1-L141)
- [references.py](file://src/memu/utils/references.py#L149-L172)

**Section sources**
- [memorize.py](file://src/memu/app/memorize.py#L1038-L1098)
- [category_with_refs.py](file://src/memu/prompts/category_summary/category_with_refs.py#L1-L141)
- [references.py](file://src/memu/utils/references.py#L149-L172)

### Relationship Mapping Between Categories and Items
Items are linked to categories via a junction table. The system provides:
- Link creation and removal
- Retrieval of categories for a given item
- Listing relations with filtering

```mermaid
classDiagram
class MemoryItem {
+string id
+string resource_id
+string memory_type
+string summary
+dict extra
}
class MemoryCategory {
+string id
+string name
+string description
+string summary
}
class CategoryItem {
+string id
+string item_id
+string category_id
}
MemoryItem "1" <--* "many" CategoryItem : "links to"
MemoryCategory "1" <--* "many" CategoryItem : "links to"
```

**Diagram sources**
- [models.py](file://src/memu/database/models.py#L76-L106)
- [category_item_repo.py (SQLite)](file://src/memu/database/sqlite/repositories/category_item_repo.py#L84-L173)
- [category_item_repo.py (Postgres)](file://src/memu/database/postgres/repositories/category_item_repo.py#L35-L87)
- [category_item_repo.py (InMemory)](file://src/memu/database/inmemory/repositories/category_item_repo.py#L24-L42)

**Section sources**
- [models.py](file://src/memu/database/models.py#L76-L106)
- [category_item_repo.py (SQLite)](file://src/memu/database/sqlite/repositories/category_item_repo.py#L84-L173)
- [category_item_repo.py (Postgres)](file://src/memu/database/postgres/repositories/category_item_repo.py#L35-L87)
- [category_item_repo.py (InMemory)](file://src/memu/database/inmemory/repositories/category_item_repo.py#L24-L42)

### Persisting References to Items
After generating summaries, the system extracts referenced short IDs and persists them into the items’ extra metadata. This enables later retrieval of referenced items.

```mermaid
flowchart TD
P0["updated_summaries dict"] --> P1["_extract_refs_from_summaries()"]
P1 --> P2{"Any refs?"}
P2 --> |No| PEnd["Return"]
P2 --> |Yes| P3["Build short_id->full_item_id map"]
P3 --> P4["For each referenced short_id:<br/>lookup full_item_id"]
P4 --> P5{"Match found?"}
P5 --> |No| P6["Skip (missing reference)"]
P5 --> |Yes| P7["store.memory_item_repo.update_item()<br/>set extra.ref_id"]
P6 --> P8["Next referenced short_id"]
P7 --> P8
P8 --> P9{"More refs?"}
P9 --> |Yes| P4
P9 --> |No| PEnd
```

**Diagram sources**
- [memorize.py](file://src/memu/app/memorize.py#L984-L1037)
- [interfaces.py](file://src/memu/database/interfaces.py#L12-L26)

**Section sources**
- [memorize.py](file://src/memu/app/memorize.py#L984-L1037)
- [interfaces.py](file://src/memu/database/interfaces.py#L12-L26)

### Resolving References to Actual Memory Items
There are two complementary mechanisms:
- Direct resolution: Extract referenced IDs and fetch items from the store.
- Reverse lookup: Query items whose extra.ref_id matches a set of short IDs.

```mermaid
sequenceDiagram
participant Util as "references.py"
participant DB as "Database Store"
participant Repo as "MemoryItemRepo"
Util->>Util : "extract_references(text)"
Util->>Repo : "get_item(item_id)" for each ID
Repo-->>Util : "MemoryItem or None"
Util-->>DB : "Return list of item dicts"
```

**Diagram sources**
- [references.py](file://src/memu/utils/references.py#L118-L146)
- [interfaces.py](file://src/memu/database/interfaces.py#L12-L26)
- [memory_item_repo.py (SQLite)](file://src/memu/database/sqlite/repositories/memory_item_repo.py#L119-L151)
- [memory_item_repo.py (InMemory)](file://src/memu/database/inmemory/repositories/memory_item_repo.py#L27-L51)

**Section sources**
- [references.py](file://src/memu/utils/references.py#L118-L146)
- [interfaces.py](file://src/memu/database/interfaces.py#L12-L26)
- [memory_item_repo.py (SQLite)](file://src/memu/database/sqlite/repositories/memory_item_repo.py#L119-L151)
- [memory_item_repo.py (InMemory)](file://src/memu/database/inmemory/repositories/memory_item_repo.py#L27-L51)

## Dependency Analysis
The reference system depends on:
- Regex-based parsing for inline references
- Prompt templates that enforce reference inclusion
- Short ID mapping for prompts and reverse mapping for persistence
- Database repositories supporting both forward and reverse reference queries

```mermaid
graph LR
RE["references.py"] --> PR["category_with_refs.py"]
RE --> APP["memorize.py"]
APP --> IF["interfaces.py"]
APP --> MI_SQL["memory_item_repo.py (SQLite)"]
APP --> CI_SQL["category_item_repo.py (SQLite)"]
APP --> MI_IM["memory_item_repo.py (InMemory)"]
APP --> CI_PG["category_item_repo.py (Postgres)"]
APP --> CI_IM["category_item_repo.py (InMemory)"]
```

**Diagram sources**
- [references.py](file://src/memu/utils/references.py#L1-L173)
- [category_with_refs.py](file://src/memu/prompts/category_summary/category_with_refs.py#L1-L141)
- [memorize.py](file://src/memu/app/memorize.py#L981-L1037)
- [interfaces.py](file://src/memu/database/interfaces.py#L12-L26)
- [memory_item_repo.py (SQLite)](file://src/memu/database/sqlite/repositories/memory_item_repo.py#L119-L151)
- [category_item_repo.py (SQLite)](file://src/memu/database/sqlite/repositories/category_item_repo.py#L84-L173)
- [memory_item_repo.py (InMemory)](file://src/memu/database/inmemory/repositories/memory_item_repo.py#L27-L51)
- [category_item_repo.py (Postgres)](file://src/memu/database/postgres/repositories/category_item_repo.py#L35-L87)
- [category_item_repo.py (InMemory)](file://src/memu/database/inmemory/repositories/category_item_repo.py#L24-L42)

**Section sources**
- [references.py](file://src/memu/utils/references.py#L1-L173)
- [category_with_refs.py](file://src/memu/prompts/category_summary/category_with_refs.py#L1-L141)
- [memorize.py](file://src/memu/app/memorize.py#L981-L1037)
- [interfaces.py](file://src/memu/database/interfaces.py#L12-L26)
- [memory_item_repo.py (SQLite)](file://src/memu/database/sqlite/repositories/memory_item_repo.py#L119-L151)
- [category_item_repo.py (SQLite)](file://src/memu/database/sqlite/repositories/category_item_repo.py#L84-L173)
- [memory_item_repo.py (InMemory)](file://src/memu/database/inmemory/repositories/memory_item_repo.py#L27-L51)
- [category_item_repo.py (Postgres)](file://src/memu/database/postgres/repositories/category_item_repo.py#L35-L87)
- [category_item_repo.py (InMemory)](file://src/memu/database/inmemory/repositories/category_item_repo.py#L24-L42)

## Performance Considerations
- Regex scanning: Linear in text length; efficient for typical summary sizes.
- Unique ID extraction: Uses a set for O(1) dedup checks; overall linear in number of matches.
- Short ID mapping: One pass over items in category_updates; constant-time lookups.
- Database queries:
  - Forward lookup: One item fetch per referenced ID; batch by ID list if needed.
  - Reverse lookup: JSON extraction and IN-list scan; consider indexing extra fields if supported by backend.
- Parallelism: LLM calls for multiple categories can be executed concurrently; ensure thread-safe repository usage.
- Caching: Keep recent items and relations cached in-memory to reduce repeated lookups.

[No sources needed since this section provides general guidance]

## Troubleshooting Guide
Common issues and resolutions:
- Missing references in summaries: Ensure the prompt requires [ref:ID] and that new items are passed as (id, summary) tuples when reference mode is enabled.
- Broken references: During persistence, missing matches are skipped; verify item IDs exist and were created before summarization.
- Duplicate references: Extraction guarantees uniqueness; verify the input text does not repeat the same ID unnecessarily.
- Display artifacts: Use stripping to remove inline references for clean rendering; verify punctuation normalization.
- Reverse lookup failures: Confirm that items were persisted with extra.ref_id and that the ref_id values are correct.

Validation references:
- Tests for extraction, stripping, citation formatting, and round-trip behavior: [test_references.py](file://tests/test_references.py#L1-L192)

**Section sources**
- [test_references.py](file://tests/test_references.py#L1-L192)

## Conclusion
The reference resolution and cross-linking system combines structured prompts, robust parsing, and database-backed persistence to maintain traceable links between category summaries and memory items. By enforcing inline references, mapping short IDs during prompting, and persisting ref_id metadata, the system enables reliable cross-linking and retrieval. For large-scale knowledge bases, careful attention to regex efficiency, caching, and database indexing will ensure responsive performance while maintaining data integrity.