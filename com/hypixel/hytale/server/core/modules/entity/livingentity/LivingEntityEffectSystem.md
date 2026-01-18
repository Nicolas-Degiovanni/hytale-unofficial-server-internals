---
description: Architectural reference for LivingEntityEffectSystem
---

# LivingEntityEffectSystem

**Package:** com.hypixel.hytale.server.core.modules.entity.livingentity
**Type:** System Component

## Definition
```java
// Signature
public class LivingEntityEffectSystem extends EntityTickingSystem<EntityStore> implements DisableProcessingAssert {
```

## Architecture & Concepts
The LivingEntityEffectSystem is a server-side system within the Entity Component System (ECS) framework responsible for managing the lifecycle of all active status effects on entities. It functions as a core processor in the game loop, ensuring that effects like poison, regeneration, or burning are updated, applied, and removed correctly over time.

This system operates on a specific subset of entities: those that possess an EffectControllerComponent. During each server tick, it iterates through every active effect on every matched entity. For each effect, it calculates the time delta, applies the effect's logic (which may involve stat modification, damage application, or model changes), and decrements its remaining duration.

Crucially, this system is part of the DamageModule's **gatherDamageGroup**. This explicit grouping dictates its execution order relative to other game logic, ensuring that effect-related state changes are calculated before the final damage application phase of a game tick. This prevents logical race conditions where an entity might die before a life-saving regeneration effect has a chance to tick.

## Lifecycle & Ownership
-   **Creation:** The LivingEntityEffectSystem is discovered and instantiated by the server's central SystemManager during world initialization. It is not intended for manual creation by developers.
-   **Scope:** An instance of this system persists for the entire lifetime of the server world it is associated with. It is a long-lived, foundational component of the server's core logic.
-   **Destruction:** The system is destroyed and its resources are released when the server world is shut down or unloaded.

## Internal State & Concurrency
-   **State:** This class is designed to be **stateless**. It does not maintain any internal state between ticks. All data it operates on is read from and written to external components, primarily the EffectControllerComponent and the global CommandBuffer. This stateless design is fundamental to the ECS pattern.
-   **Thread Safety:** This system is **not thread-safe**. The `isParallel` method explicitly returns false, signaling to the ECS scheduler that its `tick` method must be executed serially. This is a critical safeguard, as the system performs complex, non-atomic operations on shared component data. The implementation of the `DisableProcessingAssert` interface further reinforces that this system has strict execution constraints that must be respected by the engine.

## API Surface
The primary contract is with the ECS scheduler, not direct callers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getQuery() | Query | O(1) | Defines the component query that selects entities for processing. |
| isParallel(int, int) | boolean | O(1) | Returns false, indicating the system cannot be run in parallel. |
| tick(float, int, ...) | void | O(N * M) | The main update loop called by the engine. N is the number of entities, M is the average number of effects per entity. |
| getGroup() | SystemGroup | O(1) | Specifies the execution group, linking it to the DamageModule's lifecycle. |
| canApplyEffect(...) | static boolean | O(V) | A static utility to check environmental conditions. V is the volume of the entity's bounding box in blocks. |

## Integration Patterns

### Standard Usage
Developers do not interact with this system directly. To apply a status effect to an entity, one must add an ActiveEntityEffect to its EffectControllerComponent. The system will automatically detect and process the entity on the next server tick.

```java
// CORRECT: Add a component to an entity to have the system process it.
// This code would exist in another system or game logic handler.

// Assume 'entityRef' is a valid reference to an entity.
// Assume 'burnEffect' is a valid, loaded EntityEffect asset.

EffectControllerComponent controller = commandBuffer.getOrAddComponent(entityRef, EffectControllerComponent.class);
ActiveEntityEffect activeBurn = new ActiveEntityEffect(burnEffect.getAssetIndex(), 10.0f); // 10 second duration

controller.addEffect(activeBurn);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new LivingEntityEffectSystem()`. The system is managed entirely by the engine's lifecycle. Direct creation will result in a non-functional object that is not registered with the scheduler.
-   **Manual Ticking:** Do not call the `tick` method manually. This bypasses the engine's CommandBuffer, dependency management via SystemGroup, and thread safety guarantees, which will corrupt entity state and cause severe, difficult-to-debug issues.
-   **Concurrent Modification:** Do not modify an entity's EffectControllerComponent from another thread while the game loop is running. The system's non-parallel nature exists specifically to prevent race conditions from such modifications.

## Data Pipeline
The LivingEntityEffectSystem acts as a state transformer within the main game tick. It does not generate new data from external sources but rather evolves the existing state of entity components.

> Flow:
> Game Tick Begins -> ECS Scheduler queries for entities with EffectControllerComponent -> **LivingEntityEffectSystem.tick()** is invoked for each entity -> Reads `ActiveEntityEffect` state -> Reads `EntityEffect` asset data -> Performs environmental checks (e.g., `canApplyEffect`) -> Writes updated component state (e.g., new durations, stat modifiers) to the `CommandBuffer` -> The `CommandBuffer` applies these changes atomically at the end of the system processing phase.

