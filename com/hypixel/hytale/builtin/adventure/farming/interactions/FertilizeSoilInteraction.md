---
description: Architectural reference for FertilizeSoilInteraction
---

# FertilizeSoilInteraction

**Package:** com.hypixel.hytale.builtin.adventure.farming.interactions
**Type:** Configuration-Driven Component

## Definition
```java
// Signature
public class FertilizeSoilInteraction extends SimpleBlockInteraction {
```

## Architecture & Concepts
The FertilizeSoilInteraction class encapsulates a single, server-authoritative game mechanic: applying fertilizer to a block. It is a concrete implementation within the server's broader Interaction System, which processes player and entity actions within the world.

This class is not a long-lived service but rather a stateless logic handler. Its existence and behavior are defined by game data files, which are deserialized at runtime using its static CODEC field. This data-driven approach allows designers to create and modify game interactions without recompiling the engine source code.

Architecturally, it serves as a terminal node in the interaction processing chain. When a player action matches the criteria for this interaction (e.g., using a specific item on a specific block), the system invokes the `interactWithBlock` method. The method's responsibility is to directly query and mutate the state of the world's components, specifically the `TilledSoilBlock` and `FarmingBlock` components attached to block entities.

The `getWaitForDataFrom` method's return value, `WaitForDataFrom.Server`, is critical. It designates this interaction as server-authoritative, preventing client-side cheating. The client may send an interaction request, but the server performs the actual state change and broadcasts the result.

## Lifecycle & Ownership
-   **Creation:** Instances are not created directly via the `new` keyword. The Hytale engine's codec system instantiates this class during server startup when it parses game configuration assets (e.g., JSON files defining items and their behaviors). The static `CODEC` field acts as the factory.
-   **Scope:** An instance loaded from configuration persists for the entire server session. It is effectively a singleton within the context of the interaction registry it belongs to.
-   **Destruction:** The object is eligible for garbage collection when the server shuts down and its configuration registries are cleared.

## Internal State & Concurrency
-   **State:** This class is effectively stateless. Its only field, `refreshModifiers`, is populated once during deserialization and is treated as immutable thereafter. All state modifications performed by its methods are on external objects passed as parameters, primarily the `World` and its underlying `ChunkStore`.
-   **Thread Safety:** This class is **not thread-safe**. Its methods, particularly `interactWithBlock`, perform direct, unsynchronized mutations on world chunk data. It is imperative that this method is only ever invoked from the main server game loop thread for the corresponding world. Calling it from any other thread will lead to world state corruption, race conditions, and server instability. The use of a `CommandBuffer` parameter is a strong indicator that state changes are intended to be queued and executed serially by the engine.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getWaitForDataFrom() | WaitForDataFrom | O(1) | Returns `Server`, indicating this is a server-authoritative interaction. |
| interactWithBlock(...) | void | O(1) | Executes the core fertilization logic. Mutates the state of `TilledSoilBlock` components. Fails silently by setting the `InteractionState` if conditions are not met. |
| simulateInteractWithBlock(...) | void | O(1) | An empty implementation for client-side prediction. Its absence of logic reinforces that the client must wait for the server's authoritative response. |

## Integration Patterns

### Standard Usage
This class is not intended to be invoked directly by most game logic developers. It is automatically triggered by the server's core interaction module based on game data configuration. A designer, not a programmer, would typically associate this interaction with an item in a configuration file.

The system's internal invocation would resemble the following conceptual flow:

```java
// Conceptual example of how the InteractionModule might use this class.
// This code would exist deep within the server engine.

// 1. An interaction event is received for a target block.
InteractionContext context = ...;
Vector3i targetBlock = ...;

// 2. The system looks up the interaction handler from its registry.
// In this case, it finds an instance of FertilizeSoilInteraction.
SimpleBlockInteraction handler = interactionRegistry.getHandlerFor(context.getItemInHand());

// 3. The handler's logic is executed on the main world thread.
if (handler instanceof FertilizeSoilInteraction) {
    handler.interactWithBlock(world, commandBuffer, type, context, item, targetBlock, cooldowns);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new FertilizeSoilInteraction()`. This bypasses the data-driven configuration system and the `CODEC`, resulting in a non-functional object that is not registered with the game engine.
-   **Asynchronous Execution:** Do not call `interactWithBlock` from a separate thread, a network handler, or any context outside the main server tick. This will corrupt world state. All world mutations must be synchronized with the game loop.
-   **State Assumption:** Do not assume the interaction will succeed. The method internally checks for the presence of `TilledSoilBlock` or `FarmingBlock` components. If these are missing on the target block or its neighbor, the interaction fails by setting the `InteractionState` to `Failed`.

## Data Pipeline
The flow of data for this interaction is strictly server-authoritative, originating from a client request but resolved entirely on the server.

> Flow:
> Client Input (Use Item) -> Network Interaction Packet -> Server InteractionModule -> **FertilizeSoilInteraction.interactWithBlock** -> World State Mutation (ChunkStore) -> World Update Packet -> Client Visual Update

