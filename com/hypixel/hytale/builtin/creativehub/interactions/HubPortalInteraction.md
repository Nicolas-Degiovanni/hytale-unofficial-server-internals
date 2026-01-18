---
description: Architectural reference for HubPortalInteraction
---

# HubPortalInteraction

**Package:** com.hypixel.hytale.builtin.creativehub.interactions
**Type:** Configured Object

## Definition
```java
// Signature
public class HubPortalInteraction extends SimpleInstantInteraction {
```

## Architecture & Concepts
The HubPortalInteraction is a server-side event handler that orchestrates the teleportation of a player entity to a different world. It is a concrete implementation of the engine's interaction system, designed to be triggered by a player action, such as right-clicking a specific entity configured as a "portal".

This class acts as a high-level controller for the complex, and often asynchronous, process of world transition. Its primary responsibilities include:

1.  **World Resolution:** Determining if the target world is already loaded, needs to be loaded from storage, or must be created from scratch.
2.  **Asynchronous Orchestration:** Managing the non-blocking lifecycle of world creation and player transfer using Java's CompletableFuture API. This prevents the server from stalling during I/O-heavy operations like loading world data from disk.
3.  **Entity State Management:** Interfacing with the Entity Component System (ECS) via a CommandBuffer to safely add and remove components (e.g., Teleport, PendingTeleport) that drive the underlying teleportation mechanics.
4.  **Failure Recovery:** Providing a robust fallback mechanism to handle scenarios where the target world fails to load, ensuring the player is not left in a void state.

Architecturally, it sits between the server's core interaction module and the Universe module, translating a simple player action into a sophisticated world management and entity lifecycle operation.

### Lifecycle & Ownership
-   **Creation:** HubPortalInteraction instances are not created directly using the new keyword. They are instantiated by the engine's serialization system via the static CODEC field. This process typically occurs when the server loads entity configurations or game mode assets from disk. The fields such as worldName and worldGenType are populated from the configuration data during this deserialization.
-   **Scope:** An instance's lifetime is scoped to a single interaction event. When a player triggers the interaction, the engine invokes the firstRun method on the configured instance. The object itself holds no state across multiple interactions and is effectively stateless from the perspective of the caller.
-   **Destruction:** The object is eligible for garbage collection once the firstRun method completes and any associated asynchronous callbacks are resolved. It does not manage any unmanaged resources that require explicit cleanup.

## Internal State & Concurrency
-   **State:** The internal state (worldName, worldGenType, instanceTemplate) is set once during deserialization and is treated as immutable for the duration of the object's lifecycle. It serves as the configuration for the interaction logic. The class itself does not cache data or maintain state between invocations.
-   **Thread Safety:** This class is **not thread-safe** and is designed to be accessed only from the main server or world thread. The primary entry point, firstRun, must be called from the appropriate engine thread.

    **WARNING:** The class initiates asynchronous operations which execute callbacks on different threads. All interactions with the game state (e.g., adding components, moving the player) within these callbacks are carefully marshaled back to the correct world thread via mechanisms like world.execute or by operating on a CommandBuffer. Direct modification of game state from a CompletableFuture callback without proper thread synchronization will lead to severe concurrency issues, including data corruption and server crashes.

## API Surface
The public contract is defined by its parent class, SimpleInstantInteraction.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| firstRun(type, context, cooldown) | void | O(N) | The primary entry point. Orchestrates the entire teleportation flow. Complexity is dominated by the asynchronous world load/creation time (N), which is an I/O-bound operation. |
| getWaitForDataFrom() | WaitForDataFrom | O(1) | Returns WaitForDataFrom.Server, signaling to the engine that this interaction can execute immediately on the server without waiting for client-side data. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in procedural code. It is designed to be defined declaratively within an entity's configuration files (e.g., a JSON asset). The engine's interaction system reads this configuration and triggers the interaction automatically.

A conceptual configuration might look like this:
```json
{
  "id": "my_portal_entity",
  "components": {
    "Interaction": {
      "type": "HubPortalInteraction",
      "WorldName": "PlayerPlot_1234",
      "WorldGenType": "Flat"
    }
  }
}
```

The engine would then be responsible for invoking the interaction:
```java
// Conceptual engine code
Interaction configuredInteraction = entity.getComponent(Interaction.class);
InteractionContext context = createInteractionContext(player, entity);

// The engine calls firstRun, triggering the entire teleportation sequence
configuredInteraction.firstRun(InteractionType.USE, context, cooldownHandler);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance with `new HubPortalInteraction()`. The internal configuration fields will be null, causing a NullPointerException when firstRun is invoked. Always allow the engine to create it via its CODEC.
-   **State Mutation:** Do not attempt to modify the worldName or other fields after the object has been created. This can lead to unpredictable behavior, as the fields are intended to be immutable configuration.
-   **Blocking Operations:** Calling `get()` or `join()` on the `CompletableFuture` returned by world loading methods from within the main server thread will freeze the entire server. The design relies exclusively on non-blocking callbacks (`thenCompose`, `whenComplete`).

## Data Pipeline
The flow of data and control for a successful teleport to a non-existent world is a multi-stage, asynchronous process.

> Flow:
> Player Input -> Network Packet -> Server Interaction System -> **HubPortalInteraction.firstRun()**
> 1.  The Universe is queried for a world named `worldName`. The world is not found.
> 2.  A call is made to `Universe.addWorld()`, which returns a `CompletableFuture<World>`.
> 3.  The player entity is temporarily removed from its current world (`originalWorld.execute(playerRefComponent::removeFromStore)`). This prevents physics and other systems from acting on the player during the transition.
> 4.  A series of non-blocking callbacks are attached to the `CompletableFuture`.
> 5.  **On Completion (Success):** The callback receives the newly created `World` object. It then calls `world.addPlayer()` to spawn the player entity into the new world at its designated spawn point.
> 6.  **On Completion (Failure):** The `whenComplete` block executes. It logs the error and attempts a series of fallbacks:
>     a. Try to re-add the player to their original world.
>     b. If the original world is no longer alive, try a configured parent hub world.
>     c. If no parent exists, try the server's default world.
>     d. If no fallback world is available, disconnect the player to prevent them from being stuck in an invalid state.

