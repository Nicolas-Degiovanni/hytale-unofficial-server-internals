---
description: Architectural reference for ParameterStore
---

# ParameterStore

**Package:** com.hypixel.hytale.server.npc.storage
**Type:** Abstract Base Class

## Definition
```java
// Signature
public abstract class ParameterStore<Type extends PersistentParameter<?>> {
```

## Architecture & Concepts
The ParameterStore is an abstract base class that provides a foundational implementation for storing and retrieving named, persistent parameters associated with an entity. It serves as a specialized, type-safe data store for a single category of parameters, such as integers, strings, or booleans.

Architecturally, this class employs the **Template Method Pattern**. The public `get` method defines a fixed algorithm for parameter retrieval, which includes a critical lazy-initialization step. The variable part of this algorithm, the actual creation of a new parameter instance, is delegated to subclasses through the abstract `createParameter` method.

A key design feature is its direct integration with the server's persistence layer. When a parameter is requested for the first time, the store not only instantiates it but also immediately flags the owning entity as "dirty" by calling `owner.markNeedsSave()`. This ensures that newly created NPC state is automatically queued for writing to the database, preventing data loss. This makes a seemingly read-only operation (`get`) a potentially mutating one.

## Lifecycle & Ownership
- **Creation:** As an abstract class, ParameterStore is never instantiated directly. Concrete subclasses (e.g., `IntegerParameterStore`, `StringParameterStore`) are created and managed by a higher-level state container, typically associated with a specific NPC or entity instance.
- **Scope:** The lifetime of a ParameterStore instance is tightly coupled to its owning entity's in-memory representation. It is created when the entity's state is loaded and persists as long as the entity remains active on the server.
- **Destruction:** The object is eligible for garbage collection when the parent entity is unloaded or destroyed. There is no explicit cleanup or `close` method.

## Internal State & Concurrency
- **State:** The core state is a mutable `HashMap` that maps a parameter's string name to its corresponding `PersistentParameter` instance. The state is dynamic, growing on-demand as new parameters are accessed for the first time. This acts as an in-memory cache for the entity's parameters.
- **Thread Safety:** **This class is not thread-safe.** The internal use of a standard `HashMap` without any synchronization mechanisms makes it vulnerable to race conditions. Concurrent calls to the `get` method with the same, previously unknown parameter name could result in multiple parameter instances being created or lead to `ConcurrentModificationException`.

    **WARNING:** All access to a ParameterStore instance must be externally synchronized. It is designed to be called exclusively from the main server thread that owns the target entity.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(Entity owner, String name) | Type | O(1) | Retrieves the parameter by name. If not found, creates it, stores it, and marks the owner entity for a database save. |
| createParameter() | Type | Varies | **Protected abstract method.** Subclasses must implement this to provide the logic for instantiating a new parameter of the specific type. |

## Integration Patterns

### Standard Usage
The intended use is to retrieve a specific store from an entity's state manager and then request a parameter from it. The logic handles whether the parameter needs to be created or simply returned.

```java
// Assume 'npc' is a valid Entity object with a state manager
// and IntegerParameterStore is a concrete implementation.

// 1. Get the specific store for integer parameters
IntegerParameterStore intStore = npc.getState().getIntegerStore();

// 2. Get a parameter. This is safe even if "quest_progress" doesn't exist yet.
// If it's the first time, it will be created and the npc will be marked for saving.
IntegerParameter progress = intStore.get(npc, "quest_progress");

// 3. Interact with the parameter
progress.setValue(progress.getValue() + 1);
```

### Anti-Patterns (Do NOT do this)
- **Multi-threaded Access:** Never call the `get` method from multiple threads without an external locking mechanism. This will lead to data corruption.
- **Ignoring Side Effects:** Do not treat the `get` method as a pure read operation. It can trigger a database write on the owning entity, which may have performance implications if called improperly in a tight loop with new keys.
- **Incorrect Subclassing:** Implementations of `createParameter` must be lightweight and must not have complex dependencies, as this method can be called during critical game logic.

## Data Pipeline
The ParameterStore acts as a lazy-loading cache between the game logic and an entity's persistent state.

> Flow:
> NPC Behavior Script -> `IntegerParameterStore.get(npc, "param_name")` -> **ParameterStore** (Internal HashMap lookup) -> [Cache Miss] -> `createParameter()` call -> `parameters.put()` -> `owner.markNeedsSave()` -> Return new `PersistentParameter` instance to caller.

