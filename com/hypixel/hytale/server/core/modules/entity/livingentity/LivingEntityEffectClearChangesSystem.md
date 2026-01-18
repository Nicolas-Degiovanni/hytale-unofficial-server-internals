---
description: Architectural reference for LivingEntityEffectClearChangesSystem
---

# LivingEntityEffectClearChangesSystem

**Package:** com.hypixel.hytale.server.core.modules.entity.livingentity
**Type:** System Component

## Definition
```java
// Signature
public class LivingEntityEffectClearChangesSystem extends EntityTickingSystem<EntityStore> {
```

## Architecture & Concepts
The LivingEntityEffectClearChangesSystem is a specialized housekeeping system within the server-side Entity Component System (ECS) framework. Its sole responsibility is to finalize the processing of entity status effects for a given server tick.

Architecturally, it acts as a state-reset mechanism. Other systems, particularly the EffectControllerSystem, apply, update, or remove status effects on an entity's EffectControllerComponent throughout a tick. These modifications are tracked internally within the component. This system is the final step in that pipeline, executing after all effect modifications have occurred. It calls the clearChanges method on the component, wiping its internal change log.

This design pattern is critical for correctness. It ensures that changes from tick N are not accidentally re-processed or carried over into tick N+1, preventing bugs like effects applying twice or removal events being missed. The system's explicit dependency on running *after* the EffectControllerSystem enforces this rigid, sequential data flow.

## Lifecycle & Ownership
- **Creation:** Instantiated automatically by the server's ECS System Scheduler during world initialization. This is not a class that developers instantiate manually.
- **Scope:** The instance persists for the entire lifetime of the server world it is registered with.
- **Destruction:** The instance is destroyed and garbage collected when the associated server world is unloaded or the server shuts down.

## Internal State & Concurrency
- **State:** This system is **stateless**. It contains no mutable instance fields and operates exclusively on the component data passed into its tick method. The static DEPENDENCIES set is immutable.
- **Thread Safety:** This system is **not thread-safe** and is designed to be executed by a single-threaded ECS scheduler. The framework guarantees that the tick method is invoked in a safe context, with exclusive access to the components for the duration of its execution. Concurrently invoking the tick method from multiple threads will lead to race conditions and undefined behavior.

## API Surface
The public API is designed for consumption by the ECS framework, not for direct developer invocation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, index, chunk, store, cmd) | void | O(1) | Executes the core logic for a single entity, clearing the change state of its EffectControllerComponent. |
| getQuery() | Query | O(1) | Defines the selection criteria for this system. It targets all entities possessing an EffectControllerComponent. |
| getDependencies() | Set | O(1) | Declares execution order constraints. Guarantees this system runs after EffectControllerSystem. |

## Integration Patterns

### Standard Usage
This system is not used directly. It is automatically discovered and registered with the server's system scheduler at startup. Its operation is entirely managed by the ECS framework based on its query and dependency definitions. A developer's interaction is limited to creating entities with the required EffectControllerComponent, which makes them eligible for processing by this system.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new LivingEntityEffectClearChangesSystem()`. The ECS framework is solely responsible for its lifecycle.
- **Manual Invocation:** Calling the `tick` method directly will bypass the dependency scheduler. This could cause it to run before effect changes have been processed, leading to a permanent loss of effect updates for that tick.
- **Altering Dependencies:** Modifying the system's dependencies to change its execution order is highly dangerous and will break the fundamental logic of the status effect pipeline.

## Data Pipeline
This system is the terminal stage in the per-tick effect change pipeline. It does not produce data but rather resets state for the next cycle.

> Flow:
> Game Logic (e.g., potion use) -> Modifies **EffectControllerComponent** -> EffectControllerSystem processes changes -> **LivingEntityEffectClearChangesSystem** reads component -> `clearChanges()` is called -> **EffectControllerComponent** state is reset for the next tick.

