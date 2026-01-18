---
description: Architectural reference for SpawnMinecartInteraction
---

# SpawnMinecartInteraction

**Package:** com.hypixel.hytale.builtin.mounts.interactions
**Type:** Data-Driven Handler

## Definition
```java
// Signature
public class SpawnMinecartInteraction extends SimpleBlockInteraction {
```

## Architecture & Concepts
The SpawnMinecartInteraction is a server-side, data-driven handler responsible for the logic of spawning a minecart entity when a player interacts with a block. It operates within the server's core Interaction System and extends the base SimpleBlockInteraction, inheriting its contract for block-based actions.

Architecturally, this class is not a persistent service but rather a concrete implementation of an *interaction type*. Its behavior is defined almost entirely by data assets, which are deserialized at runtime using its static CODEC. This design allows game designers to create numerous variations of minecart-spawning interactions in configuration files without modifying engine source code.

The core of its operation is the `interactWithBlock` method. This method leverages the engine's Entity Component System (ECS) by constructing a new entity via a Holder and submitting it to a CommandBuffer. The use of a CommandBuffer is a critical architectural pattern here; it decouples the interaction logic from the direct mutation of world state, ensuring that all entity modifications are queued and processed safely within the main server tick.

A key feature is its ability to intelligently position the spawned minecart. If the target block has rail configuration, the `alignToRail` logic performs vector calculations to snap the minecart precisely to the track. Otherwise, it defaults to placing the minecart on top of the target block's bounding box.

## Lifecycle & Ownership
-   **Creation:** Instances of SpawnMinecartInteraction are not instantiated directly using the `new` keyword. They are created by the server's asset loading system during startup. The static `CODEC` field is used to deserialize configuration from game asset files (e.g., JSON or HOCON) into a fully configured Java object.

-   **Scope:** An instance represents a single, configured type of interaction. It is loaded once and held in a central interaction registry for the entire lifetime of the server session. It is effectively a stateless singleton for its specific configuration.

-   **Destruction:** The object is destroyed and garbage collected only when the server shuts down or when a full asset reload is triggered.

## Internal State & Concurrency
-   **State:** The internal state consists of `modelId` and `cartInteractions`. These fields are populated exclusively during deserialization at creation time. After initialization, the object's state is **effectively immutable**.

-   **Thread Safety:** The instance itself is thread-safe due to its immutable state. However, the methods it executes, such as `interactWithBlock`, operate on shared and mutable world state. Concurrency control is not managed within this class but is delegated to the calling system. The engine guarantees safety by invoking these handlers from a single, main server thread and by using a CommandBuffer to batch and defer entity mutations.

    **Warning:** Calling methods on this class from an asynchronous thread without proper synchronization through the CommandBuffer will lead to world state corruption and server instability.

## API Surface
The public contract is primarily defined by its role as a SimpleBlockInteraction handler.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CODEC | BuilderCodec | O(1) | Public static factory used by the asset system to deserialize and validate this interaction type from configuration files. |
| interactWithBlock(...) | void | O(N) | Executes the core spawning logic. Creates a new minecart entity and submits it to the CommandBuffer. Complexity is O(N) where N is the number of points in a rail configuration. |
| simulateInteractWithBlock(...) | void | O(1) | No-op. This interaction is purely server-authoritative and does not support client-side predictive simulation. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in procedural code. Its primary integration point is through asset configuration. A game designer defines the interaction's properties in an asset file, which is then loaded by the server. The interaction system automatically invokes the handler when a player's action matches the configured triggers.

The following conceptual example illustrates how the interaction system would dispatch a call to this handler.

```java
// Conceptual example of the Interaction System invoking the handler
// This code is part of the engine, not typical user code.

// 1. An interaction is triggered and resolved to a SpawnMinecartInteraction instance
SpawnMinecartInteraction handler = InteractionRegistry.getHandlerFor("my_custom_minecart_spawner");

// 2. The system invokes the handler with the current world context
handler.interactWithBlock(
    world,
    commandBuffer,
    interactionType,
    interactionContext,
    itemInHand,
    targetBlock,
    cooldownHandler
);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new SpawnMinecartInteraction()`. Doing so bypasses the critical data-driven configuration from the asset pipeline. The resulting object would be unconfigured and non-functional. Always define interactions in asset files.

-   **State Mutation:** Do not attempt to modify the `modelId` or `cartInteractions` fields after the object has been loaded. This violates its immutability contract and can lead to unpredictable behavior across the server.

-   **External Invocation:** Do not call `interactWithBlock` from outside the server's core interaction module. The method relies on a very specific context, including a valid CommandBuffer and CooldownHandler, which is only guaranteed by the engine's main loop.

## Data Pipeline
The flow of data for a minecart spawning event begins with player input and ends with a new entity being created in the world. This class is a central processing step in that pipeline.

> Flow:
> Player Input -> Network Packet -> Server Interaction System -> **SpawnMinecartInteraction.interactWithBlock** -> CommandBuffer -> New Entity (Holder) with Components -> World State Update -> Network Sync to Clients

