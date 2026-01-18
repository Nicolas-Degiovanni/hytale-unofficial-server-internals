---
description: Architectural reference for PickupItemInteraction
---

# PickupItemInteraction

**Package:** com.hypixel.hytale.builtin.buildertools.interactions
**Type:** Transient Handler

## Definition
```java
// Signature
public class PickupItemInteraction extends SimpleInstantInteraction {
```

## Architecture & Concepts
The PickupItemInteraction is a server-side, data-driven handler that encapsulates the business logic for a player entity acquiring an item entity from the game world. It is a concrete implementation within the engine's broader Interaction System, which governs all entity-to-entity and entity-to-world actions.

This class operates strictly within an Entity Component System (ECS) paradigm. It is fundamentally **stateless**; it does not hold any data itself but rather acts as a pure function that processes components from the interacting entities. All world-state modifications, such as removing the item entity or updating the player's inventory, are not performed directly. Instead, they are issued as commands to a CommandBuffer provided within the InteractionContext. This ensures that all changes are queued and executed transactionally at the end of the game tick, preserving world state integrity and preventing race conditions.

The presence of a static BuilderCodec field indicates that this class is designed to be instantiated and configured via the engine's asset loading pipeline, allowing designers to define or modify interaction behaviors without changing Java code.

### Lifecycle & Ownership
- **Creation:** Instances are not created directly using the new keyword. They are deserialized from game data files by the engine's asset loader, which uses the public static CODEC field. A default instance is registered programmatically for baseline functionality.
- **Scope:** An instance of PickupItemInteraction is a long-lived, shared object. Once loaded, a single instance serves as the handler for all item pickup interactions across the entire server until the server shuts down or reloads its assets.
- **Destruction:** The object is garbage collected when the server's asset registry is cleared, typically during a server shutdown.

## Internal State & Concurrency
- **State:** This class is **immutable and stateless**. Its behavior is determined entirely by the arguments passed to its methods, primarily the InteractionContext. It does not cache data or maintain any state between calls.

- **Thread Safety:** The class is **conditionally thread-safe**. Because it is stateless, the `firstRun` method can be executed concurrently without issue. However, the safety of the entire operation is dependent on the thread-safety of the objects passed within the InteractionContext, specifically the CommandBuffer. The engine guarantees that all interactions for a given tick are processed sequentially on a single thread, making this implementation safe within the context of the game loop.

    **Warning:** Any attempt to invoke this class's methods from a multi-threaded context outside the main server tick must provide a thread-safe CommandBuffer implementation, or it will result in world state corruption.

## API Surface
The primary contract is the `firstRun` method, inherited from its parent class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| firstRun(type, context, cooldownHandler) | void | O(N) | Executes the complete item pickup logic. Complexity is O(N) where N is the number of slots in the target player's inventory. This method reads from the world state via the context and writes intended changes to the context's CommandBuffer. It will fail silently by setting the InteractionState to Failed if preconditions are not met. |

## Integration Patterns

### Standard Usage
Developers do not typically invoke this class directly. Instead, it is triggered by the server's core interaction module. An entity is configured in game data to use this interaction, and when a player gets within range or triggers the interaction, the engine calls the `firstRun` method.

A conceptual invocation by the engine might look like this:
```java
// Engine-level code (conceptual)
// This class is not meant to be called directly by game logic developers.
InteractionContext context = createInteractionContextFor(player, itemEntity);
RootInteraction root = interactionRegistry.get("*PickupItem"); // Fetches the configured interaction

// The engine dispatches the context to the appropriate handler
root.getInteraction().firstRun(InteractionType.PROXIMITY, context, playerCooldowns);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new PickupItemInteraction()`. The interaction system relies on instances managed by the asset registry. Direct creation bypasses this and will lead to unmanaged, non-functional objects.
- **Stateful Subclassing:** Avoid extending this class to add stateful fields. The engine assumes interaction handlers are stateless and may reuse a single instance for concurrent or sequential operations, leading to unpredictable behavior if state is introduced.
- **Bypassing the CommandBuffer:** Modifying components retrieved from the context directly, rather than issuing commands to the CommandBuffer, will break the transactional nature of the game tick and cause severe desynchronization and state corruption issues.

## Data Pipeline
The flow of data and control for a successful item pickup event is orchestrated by the server's main loop. This class is a single step in that pipeline.

> Flow:
> Player Proximity Check -> Interaction System identifies a valid target Item Entity -> Engine resolves the interaction type to **PickupItemInteraction** -> `firstRun` is invoked with the current world state encapsulated in an InteractionContext -> **PickupItemInteraction** performs inventory transaction logic -> Commands (e.g., RemoveEntity, AddEntity, UpdateComponent) are written to the CommandBuffer -> End of Tick -> CommandBuffer is flushed, atomically applying all changes to the world state.

