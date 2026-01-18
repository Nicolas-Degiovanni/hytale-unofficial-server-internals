---
description: Architectural reference for EntityRegistry
---

# EntityRegistry

**Package:** com.hypixel.hytale.server.core.modules.entity
**Type:** Transient

## Definition
```java
// Signature
public class EntityRegistry extends Registry<EntityRegistration> {
```

## Architecture & Concepts
The EntityRegistry is the central authority for defining all entity types available on the server. It acts as a map between a unique string identifier, such as *hytale:zombie*, and the corresponding server-side implementation details: the entity's Java class, its constructor logic, and its network serialization codec.

This class is a specialized implementation of the generic Registry pattern used throughout the engine. Its primary role is to provide the foundational metadata required by other systems, such as the World and the network layer, to spawn, manage, and synchronize entities.

Registration is a protected, time-sensitive operation. The registry is "open" for new entries only during a specific, early phase of the server startup sequence, enforced by a functional precondition. Once this phase is complete, the registry becomes a read-only catalogue, ensuring the set of defined entities remains stable and consistent for the server's entire lifetime. This design prevents unpredictable behavior that could arise from dynamically registering new entity types during active gameplay.

## Lifecycle & Ownership
-   **Creation:** Instantiated by the core `EntityModule` during the server's bootstrap process. It is not intended for direct instantiation by game logic or external modules.
-   **Scope:** The EntityRegistry is a session-scoped object. It is created once when the server starts and persists until the server shuts down. Its data is fundamental to the server's operation.
-   **Destruction:** The object is eligible for garbage collection upon server shutdown when the parent `EntityModule` is destroyed and all references to it are released.

## Internal State & Concurrency
-   **State:** The internal state, inherited from the parent Registry class, is a collection of EntityRegistration objects. This state is **mutable** during the server's initialization phase. After initialization, as enforced by the `checkPrecondition` method, the state is effectively **immutable**.
-   **Thread Safety:** This class is **not thread-safe** for write operations. All calls to `registerEntity` must occur from the main server thread during the designated registration window. Read operations (lookups) performed after the registration window has closed are safe, as the underlying collection is no longer being modified.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| registerEntity(key, clazz, constructor, codec) | EntityRegistration | O(1) | Registers a new entity type. Associates a string key with its class, a factory function for instantiation, and a network codec. Throws an IllegalStateException if called after the registration phase is complete. |

## Integration Patterns

### Standard Usage
The EntityRegistry must only be accessed during the server's initialization event sequence. A module developer defines and registers all custom entities during this phase.

```java
// Example from a module's initialization hook
public void onServerInit(ServerInitializationEvent event) {
    EntityRegistry registry = EntityModule.get().getRegistry();

    registry.registerEntity(
        "mymod:golem",
        GolemEntity.class,
        (world) -> new GolemEntity(world),
        new GolemEntityCodec()
    );
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new EntityRegistry()`. The server's state depends on a single, authoritative instance managed by the EntityModule. Always retrieve the shared instance from the module.
-   **Late Registration:** Do not attempt to call `registerEntity` after the server has finished loading. This will fail the precondition check and crash the server. All entities must be registered before any worlds are loaded or players connect. This is a critical stability guarantee.

## Data Pipeline
The EntityRegistry is not a component in a continuous data stream. Instead, it serves as a static data source that other systems query at runtime. It is populated once during initialization and then consumed by various systems throughout the server's lifecycle.

> **Phase 1: Population (Server Startup)**
> Module Initialization Code -> `EntityRegistry.registerEntity(...)` -> Internal Registry Populated

> **Phase 2: Consumption (Server Runtime)**
> Network Packet or Game Logic -> Entity Spawner -> **EntityRegistry (Lookup by Key)** -> Retrieves `EntityRegistration` -> Uses `constructor` and `codec` -> New Entity in World

