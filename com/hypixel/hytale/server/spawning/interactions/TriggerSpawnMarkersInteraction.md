---
description: Architectural reference for TriggerSpawnMarkersInteraction
---

# TriggerSpawnMarkersInteraction

**Package:** com.hypixel.hytale.server.spawning.interactions
**Type:** Data-Driven Component

## Definition
```java
// Signature
public class TriggerSpawnMarkersInteraction extends SimpleInstantInteraction {
```

## Architecture & Concepts
The TriggerSpawnMarkersInteraction is a concrete implementation of the server's interaction system. It functions as a data-driven, single-shot action that connects a game event to the entity spawning system. It inherits from SimpleInstantInteraction, signifying that its execution is immediate and atomic within a single server tick.

Its primary architectural role is to act as a command object, configured entirely through the Hytale Codec system. This allows game designers to define complex spawning behaviors in data files (e.g., JSON or HOCON) without writing or modifying Java code. The class is responsible for finding and activating specific SpawnMarkerEntity instances within a defined area of effect around the interacting entity.

To achieve this, it integrates with several core server systems:
*   **Entity Component System:** It reads entity data (e.g., TransformComponent) and schedules state changes using the CommandBuffer, ensuring that all mutations are applied safely at the end of the current tick.
*   **Spatial Partitioning:** It leverages a SpatialResource to perform an efficient, proximity-based query for nearby spawn markers, avoiding a costly iteration over all entities in the world.
*   **Spawning System:** It directly invokes the trigger method on the target SpawnMarkerEntity component, initiating the spawn logic defined within that marker.

The public static CODEC field is the cornerstone of its design, defining the serializable properties—MarkerType, Range, and Count—that can be configured by designers.

### Lifecycle & Ownership
-   **Creation:** Instances are not created manually using the new keyword. They are deserialized and instantiated by the Hytale Codec system when loading game assets, such as entity definitions or world zone files. An instance of this class is typically a component of a larger interaction configuration.
-   **Scope:** The object's lifetime is tied to the parent configuration that owns it. It is effectively a stateless configuration object whose methods are invoked when a corresponding game event occurs. It persists as long as its parent object is loaded in memory.
-   **Destruction:** Managed by the Java garbage collector. When the parent configuration is unloaded (e.g., an entity is despawned or a zone is unloaded), the instance becomes eligible for garbage collection. There is no explicit destruction method.

## Internal State & Concurrency
-   **State:** The internal state (markerType, range, count) is set once during deserialization via the CODEC. It is considered immutable after creation. For performance, the derived value rangeSquared is calculated and cached in the afterDecode lifecycle hook of the codec.
-   **Thread Safety:** This class is **not thread-safe**. It is designed to be executed exclusively on the main server thread for a given world. All interactions with the game state are mediated through a CommandBuffer, which is a thread-safe mechanism for queuing operations to be executed at a deterministic point in the server tick. This design delegates concurrency control to the engine's core loop, obviating the need for internal locking.

## API Surface
The primary contract is the inherited firstRun method, which is invoked by the server's interaction module.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| firstRun(type, context, cooldownHandler) | void | O(K) | Executes the core logic. Finds, filters, and triggers spawn markers. K is the number of entities within the specified range. Throws NullPointerException if context is invalid. |
| filterMarker(targetRef, position, commandBuffer) | Ref | O(1) | Protected helper method. Evaluates if a given entity reference is a valid, in-range, manually-triggered spawn marker matching the configured type. |

## Integration Patterns

### Standard Usage
This class is not intended to be used imperatively in Java code. It is designed to be configured declaratively in data files. The engine then invokes it based on game events. A hypothetical engine-level invocation might look like the following.

```java
// This code is conceptual and represents how the engine might
// execute an interaction defined in a data file.

InteractionContext context = createInteractionContextForEntity(entityRef);
TriggerSpawnMarkersInteraction interaction = entityRef.getInteraction(TriggerSpawnMarkersInteraction.class);

if (interaction != null) {
    // The engine's interaction system invokes the method
    interaction.firstRun(InteractionType.PRIMARY, context, CooldownHandler.NONE);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new TriggerSpawnMarkersInteraction()`. This bypasses the CODEC's initialization logic, such as the calculation of the rangeSquared field, leading to incorrect behavior.
-   **State Mutation:** Do not modify public or private fields after the object has been created. The object is designed to be immutable post-deserialization.
-   **External Invocation:** Do not call the firstRun method from outside the server's interaction processing system. It relies on a valid InteractionContext and CommandBuffer provided by the engine during a standard game tick.

## Data Pipeline
The flow of data and control for this interaction follows a clear, event-driven path through the server.

> Flow:
> Game Event (e.g., Player Action) -> Interaction System -> **TriggerSpawnMarkersInteraction.firstRun()** -> SpatialResource Query -> Filter Nearby Entities -> CommandBuffer.run(lambda) -> SpawnMarkerEntity.trigger() -> New Entity Spawns

