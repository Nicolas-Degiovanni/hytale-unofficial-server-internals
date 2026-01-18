---
description: Architectural reference for DamageMemorySystems.CollectDamage
---

# DamageMemorySystems.CollectDamage

**Package:** com.hypixel.hytale.builtin.npccombatactionevaluator.memory
**Type:** Component System

## Definition
```java
// Signature
public static class CollectDamage extends DamageEventSystem {
```

## Architecture & Concepts
The CollectDamage system is a specialized event listener within the server-side Entity Component System (ECS) framework. It operates as a passive data collector, bridging the core **Damage Pipeline** with the NPC **AI Memory** layer.

Its sole responsibility is to observe damage events targeted at entities that possess a DamageMemory component. When such an event occurs, this system intercepts it and records the magnitude of the damage into the entity's DamageMemory component. This mechanism allows AI-controlled entities to "remember" recent damage, a critical input for behavior trees and combat action evaluators that might trigger defensive or retaliatory actions.

Crucially, this system is registered within the InspectDamageGroup, as specified by the getGroup method. This placement in the execution order is significant: it ensures that CollectDamage runs after initial damage values are calculated but before final application or mitigation. It is an observer, not a mutator, of the damage event itself.

## Lifecycle & Ownership
- **Creation:** This system is not instantiated directly by user code. It is constructed and registered by a higher-level module, typically the AI or Combat module, during the server's world initialization sequence. The ECS framework manages its instance.
- **Scope:** The instance persists for the entire lifetime of a game world. It is a long-lived, stateless service object.
- **Destruction:** The system is destroyed and garbage collected when the game world is unloaded or the server shuts down, as part of the ECS engine's teardown process.

## Internal State & Concurrency
- **State:** The CollectDamage system is **stateless**. Its internal fields, such as the query and component type references, are configured at construction and are treated as immutable. It does not store or cache any per-entity data itself; all state modifications are performed on external Component data within the ECS data store.

- **Thread Safety:** This class is not designed for concurrent execution. The ECS scheduler guarantees that the handle method is invoked from a single, predictable thread for each world tick. The CommandBuffer parameter is a key part of this design, ensuring that any structural entity changes are deferred and executed in a safe, synchronized phase of the game loop.

    **WARNING:** Manually invoking the handle method from multiple threads will corrupt component state and lead to critical server instability.

## API Surface
The primary contract is its role as a DamageEventSystem, which is fulfilled for the ECS engine. Direct invocation of its methods is not a supported use case.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| handle(index, chunk, store, buffer, damage) | void | O(1) | Engine callback. Updates the DamageMemory component for a single entity. This is the system's core logic. |
| getQuery() | Query | O(1) | Provides the ECS engine with the query to identify relevant entities (those with a DamageMemory component). |
| getGroup() | SystemGroup | O(1) | Registers the system into the InspectDamageGroup, defining its precise execution order within the damage pipeline. |

## Integration Patterns

### Standard Usage
A developer does not interact with this class directly. To use its functionality, one simply adds the DamageMemory component to an entity's archetype. The engine, with this system registered, automatically handles the data collection.

A separate AI system would then read the populated DamageMemory component to influence behavior.

```java
// An AI system reads the data that CollectDamage writes.
// This code would exist in a different system, running later in the tick.

// Get the component for an entity
DamageMemory memory = entity.getComponent(DamageMemory.class);

// Make a decision based on the collected data
if (memory.getDamageTakenInLastSecond() > 20) {
    // Trigger a defensive maneuver
    aiController.execute(DefensiveStance.class);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new CollectDamage()`. Systems must be registered with the ECS engine's SystemGroup builder during module setup to function correctly.
- **Manual Invocation:** Do not call the handle method directly. This bypasses the entire event pipeline and query filtering, which will lead to incorrect behavior and potential state corruption.
- **External Modification:** Do not attempt to modify the system's internal Query object after it has been constructed. This is an unsupported operation and will break the ECS scheduler's internal caches.

## Data Pipeline
This system acts as a specific sink in the server's damage event flow. It transforms a transient event into persistent component state for later analysis by other systems.

> Flow:
> Damage Event Fired -> ECS Event Bus -> **CollectDamage System** -> Reads Damage Amount -> Writes to DamageMemory Component

