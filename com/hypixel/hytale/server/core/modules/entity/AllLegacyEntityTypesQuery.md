---
description: Architectural reference for AllLegacyEntityTypesQuery
---

# AllLegacyEntityTypesQuery

**Package:** com.hypixel.hytale.server.core.modules.entity
**Type:** Singleton

## Definition
```java
// Signature
@Deprecated
public class AllLegacyEntityTypesQuery implements Query<EntityStore> {
```

## Architecture & Concepts
The AllLegacyEntityTypesQuery is a specialized filter predicate used within the server-side Entity Component System (ECS). Its primary function is to identify all entity archetypes that conform to a legacy structure.

Marked as **Deprecated**, this class serves as a critical component of a compatibility layer. It allows newer, query-based systems to operate on entities that may not have been fully migrated to the modern component architecture. It formalizes the check for a legacy entity into a reusable object that conforms to the standard Query interface, enabling it to be passed to high-level ECS query processors.

The core logic is delegated to the EntityUtils helper, making this class a thin, stateless wrapper that acts as a formal contract for legacy entity identification. It does not query for specific components, but rather for a structural property of the entity archetype itself.

**WARNING:** This query is part of a transitional architecture. New game systems should not be designed with a dependency on this class. Its existence is a signal that a legacy entity model is still present in the codebase.

### Lifecycle & Ownership
- **Creation:** The singleton INSTANCE is instantiated statically by the JVM class loader when the AllLegacyEntityTypesQuery class is first referenced. There is no explicit creation by an engine factory or dependency injection container.
- **Scope:** Application-level. The singleton instance persists for the entire lifetime of the server process.
- **Destruction:** The object is garbage collected when the server shuts down and its class loader is unloaded. No manual cleanup is necessary.

## Internal State & Concurrency
- **State:** This is a stateless and immutable object. It contains no instance fields and its behavior is constant.
- **Thread Safety:** Inherently thread-safe. As a stateless singleton, it can be safely used by multiple systems across different threads without any need for external synchronization or locks.

## API Surface
The public contract is defined by the Query interface. The primary method of interest is *test*.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(Archetype) | boolean | O(1) | Returns true if the provided Archetype represents a legacy entity. This is the core predicate logic. |
| requiresComponentType(ComponentType) | boolean | O(1) | Always returns false. This query does not depend on the presence of any specific component. |

## Integration Patterns

### Standard Usage
This query should be used by systems that need to process, migrate, or otherwise handle legacy entities. It is passed directly to the world or entity manager's query execution methods.

```java
// Example: A migration system finding all legacy entities
EntitySystem system = ...;
QueryResult<EntityStore> legacyEntities = system.query(AllLegacyEntityTypesQuery.INSTANCE);

for (EntityRef<EntityStore> entity : legacyEntities) {
    // Perform migration or compatibility logic
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new AllLegacyEntityTypesQuery()`. This is wasteful and defeats the purpose of the provided singleton. Always use `AllLegacyEntityTypesQuery.INSTANCE`.
- **Usage in New Systems:** Do not use this query to build new gameplay features. Its presence indicates a dependency on a deprecated entity format. New features should be built using modern component queries.
- **Extending the Class:** This class is not designed for extension. Its purpose is highly specific and tied to a single, deprecated entity check.

## Data Pipeline
AllLegacyEntityTypesQuery does not process a stream of data itself. Instead, it acts as a filter within a larger data retrieval pipeline orchestrated by the ECS.

> Flow:
> ECS System -> Query Executor -> **AllLegacyEntityTypesQuery.test(archetype)** -> Filtered Set of Legacy Entities -> System Logic

