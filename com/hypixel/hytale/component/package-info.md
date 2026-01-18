---
description: Architectural reference for ComponentRegistry
---

# ComponentRegistry

**Package:** com.hypixel.hytale.component
**Type:** Singleton

## Definition
```java
// Signature
public final class ComponentRegistry extends AbstractService implements Initializable, Shutdownable {
```

## Architecture & Concepts
The ComponentRegistry is the definitive authority for component schema management within the Hytale engine's Entity-Component-System (ECS) architecture. It functions as a high-performance, centralized mapping service that translates component class definitions, such as PositionComponent or HealthComponent, into unique, densely packed integer identifiers.

This translation is not a trivial convenience; it is fundamental to the engine's performance. By converting type information into primitive integers, the registry enables other core systems, particularly the EntityManager and ArchetypeManager, to organize component data in contiguous memory blocks. This data-oriented design is critical for maximizing CPU cache efficiency during the game loop's tight, performance-sensitive update cycles.

The registry's primary responsibility is to establish the "shape" of all possible entities before the game world is loaded. It does not manage component instances themselves but rather the metadata about component *types*. It is one of the first services to be initialized and is a foundational dependency for nearly all game logic systems.

### Lifecycle & Ownership
-   **Creation:** The ComponentRegistry is instantiated once by the root `EngineContext` during the earliest phase of the engine bootstrap sequence. Its creation precedes the initialization of most other game services.
-   **Scope:** The instance is a global singleton that persists for the entire lifetime of the application process. Its state is considered a permanent fixture after the initial registration phase is complete.
-   **Destruction:** The `shutdown` method is invoked by the `EngineContext` during the engine's graceful shutdown procedure. This clears all internal mappings and releases associated memory. Direct manual destruction is unsupported and will destabilize the engine.

## Internal State & Concurrency
-   **State:** The internal state is highly mutable during the initial bootstrap phase, where it builds its internal maps of classes to integer IDs. After the `seal` method is called by the engine, the registry transitions to an effectively immutable state. All subsequent write attempts will result in a runtime exception. It caches both forward (Class to ID) and reverse (ID to Class) lookups for O(1) access.
-   **Thread Safety:** The ComponentRegistry is **not thread-safe** for write operations. All calls to `register` must be performed synchronously on the main engine thread during the bootstrap phase. Read operations, such as `getId` or `getClass`, are thread-safe and lock-free *after* the registry has been sealed. Accessing the registry for reads before it is sealed is an unsupported operation and may yield inconsistent results.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| register(Class componentClass) | int | O(1) amortized | Registers a new component type, assigning it a unique ID. Throws IllegalStateException if the registry is sealed. |
| getId(Class componentClass) | int | O(1) | Retrieves the unique integer ID for a registered component type. Throws IllegalArgumentException if the type is not registered. |
| getClass(int componentId) | Class | O(1) | Retrieves the component Class associated with a given ID. Returns null or throws if the ID is invalid. |
| seal() | void | O(N) | Locks the registry, preventing any further registrations. This is an irreversible, one-way operation. |

## Integration Patterns

### Standard Usage
The ComponentRegistry must be accessed via the central service context. Systems should retrieve component IDs during their own initialization phase and cache them for use within the game loop.

```java
// During a System's initialization phase
ComponentRegistry registry = engineContext.getService(ComponentRegistry.class);
this.positionComponentId = registry.getId(PositionComponent.class);
this.velocityComponentId = registry.getId(VelocityComponent.class);

// Later, in the update loop, use the cached ID
processEntities(archetype, positionComponentId);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new ComponentRegistry()`. The engine's integrity depends on a single, authoritative instance managed by the `EngineContext`.
-   **Late Registration:** Do not attempt to register components after the engine has finished bootstrapping and sealed the registry. All component types must be known at startup. Late registration will trigger a hard crash to prevent data corruption.
-   **ID Caching Across Sessions:** Do not hard-code or serialize component IDs. The integer values are not guaranteed to be stable between different application runs or versions. Always look up the ID from the class type at runtime.

## Data Pipeline
The ComponentRegistry is a foundational metadata service, not a data processing pipeline. Its role is to provide the schema necessary for other systems to build their own pipelines.

> Flow:
> Engine Bootstrap -> **ComponentRegistry.register()** is called for all known component types -> Engine calls **ComponentRegistry.seal()** -> Systems initialize and query **ComponentRegistry.getId()** -> Game Loop uses cached IDs to access component data arrays.

---

