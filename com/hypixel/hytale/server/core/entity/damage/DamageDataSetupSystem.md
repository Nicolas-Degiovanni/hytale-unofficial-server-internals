---
description: Architectural reference for DamageDataSetupSystem
---

# DamageDataSetupSystem

**Package:** com.hypixel.hytale.server.core.entity.damage
**Type:** System Component

## Definition
```java
// Signature
public class DamageDataSetupSystem extends HolderSystem<EntityStore> {
```

## Architecture & Concepts
The DamageDataSetupSystem is a foundational component within the server-side Entity Component System (ECS) framework. Its sole responsibility is to enforce a critical data contract: every entity classified as a "living entity" must possess a DamageDataComponent.

This system operates reactively, triggered by the ECS framework whenever a new entity matching its query is added to the world. By using the `ensureComponent` method, it idempotently attaches a DamageDataComponent, initializing the entity for participation in the game's combat and health mechanics.

Architecturally, this system decouples the logic of entity creation from the specific requirements of the damage simulation. Instead of a spawner or factory being explicitly aware of the DamageDataComponent, this system automatically provisions it based on the entity's fundamental type. This promotes a clean, modular design where systems declare their data dependencies, and initializer systems like this one ensure those dependencies are met at the appropriate time.

### Lifecycle & Ownership
- **Creation:** Instantiated once by the server's ECS registry during the bootstrap sequence. It is registered as part of the core game logic systems that govern entity behavior.
- **Scope:** Session-scoped. The single instance persists for the entire lifetime of the game server. It is stateless and its methods are invoked by the ECS framework in response to world events.
- **Destruction:** The instance is discarded and garbage collected during server shutdown when the parent ECS registry is torn down.

## Internal State & Concurrency
- **State:** This system is **stateless and immutable**. Its only internal field, `damageDataComponentType`, is a final reference injected via its constructor. It holds no per-entity or session-specific data.

- **Thread Safety:** This system is not inherently thread-safe and is designed to be operated by a single-threaded game loop or a well-defined ECS scheduler. The `onEntityAdd` callback is expected to be invoked serially.
    - **Warning:** Manually invoking its methods from multiple threads will lead to race conditions within the underlying component store and is strictly unsupported.

## API Surface
The public API consists of framework callbacks and should not be invoked directly by user code.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdd(holder, reason, store) | void | O(1) | Framework callback. Guarantees the presence of a DamageDataComponent on the target entity. |
| onEntityRemoved(holder, reason, store) | void | O(1) | Framework callback. No-op. This system performs no action on entity removal. |
| getQuery() | Query | O(1) | Framework callback. Returns the query that selects all living entities. |

## Integration Patterns

### Standard Usage
A developer does not directly call methods on this system. Instead, it is registered with the server's system registry during initialization. The framework then manages its lifecycle and invocation automatically.

```java
// Example: Registering the system during server setup
ComponentType<EntityStore, DamageDataComponent> type = ...;
SystemRegistry registry = server.getSystemRegistry();

// The system is instantiated and registered. The framework now owns it.
registry.add(new DamageDataSetupSystem(type));
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation and Invocation:** Never create an instance of this system and call its methods manually. This bypasses the ECS framework's change tracking and event system, leading to an inconsistent world state.

    ```java
    // DO NOT DO THIS
    // This will not work as expected and breaks the ECS pattern.
    DamageDataSetupSystem sys = new DamageDataSetupSystem(type);
    sys.onEntityAdd(someEntityHolder, reason, store);
    ```

- **Assuming Component Exists Beforehand:** While this system guarantees the component's presence, do not rely on this in other systems that may execute in the same tick *before* this one. System execution order is critical.

## Data Pipeline
This system acts as an initializer at the beginning of an entity's data lifecycle. It injects a component into an entity, making it available for downstream processing by other game logic systems.

> Flow:
> Entity Spawner creates a living entity -> ECS Framework detects new entity -> **DamageDataSetupSystem.onEntityAdd** is invoked -> A new `DamageDataComponent` is attached to the entity -> Combat and Health systems can now safely query for and operate on that component.

