---
description: Architectural reference for ChangeBlockInteraction
---

# ChangeBlockInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.client
**Type:** Transient Configuration Object

## Definition
```java
// Signature
public class ChangeBlockInteraction extends SimpleBlockInteraction {
```

## Architecture & Concepts

The ChangeBlockInteraction is a server-authoritative, data-driven component responsible for one of the most fundamental actions in the game: transforming one block into another. It serves as a concrete implementation within the server's broader Interaction System, which translates player actions or game events into specific world mutations.

This class is not intended for direct instantiation by developers. Instead, it is deserialized from asset configuration files (e.g., JSON) via its static **CODEC** field. This allows game designers to define complex block-changing behaviors without writing any Java code. For example, a designer could specify that interacting with a "Dirt" block while holding a "Shovel" item changes it to a "Path" block.

Architecturally, it extends SimpleBlockInteraction, inheriting the foundational logic for targeting a single block in the world. Its specialized purpose is to consult an internal mapping—a "change map"—to determine the outcome. If the targeted block exists as a key in this map, it is replaced by the corresponding value block. This process is atomic from the perspective of the interaction system.

The class also handles auxiliary effects, such as playing a sound event upon a successful block change, and includes preconditions, like checking the durability of the item used in the interaction.

## Lifecycle & Ownership

-   **Creation:** Instances are created exclusively by the **BuilderCodec** during the server's asset loading phase. The server reads interaction definition files from disk, and the codec deserializes this data into a fully configured ChangeBlockInteraction object.
-   **Scope:** An instance persists for the entire server session, cached within the server's asset management system. It is effectively a singleton relative to its specific interaction definition.
-   **Destruction:** The object is marked for garbage collection when the server's asset registry is cleared, typically during a server shutdown or a full asset reload.

## Internal State & Concurrency

-   **State:** The internal state is **mutable**, primarily due to a lazy-initialization performance optimization.
    -   **blockTypeKeys**: A Map of String to String, loaded directly from the configuration file. This represents the human-readable definition of block transformations.
    -   **changeMapIds**: An Int2IntMap that serves as a high-performance cache. It is populated on first use by resolving the string-based keys from blockTypeKeys into their integer-based asset IDs. This avoids repeated, costly string-to-ID lookups during the game loop.
    -   **soundEventIndex**: An integer representation of the configured sound event ID, also cached for performance.

-   **Thread Safety:** This class is **not thread-safe** and must be managed carefully.
    -   The lazy initialization of the changeMapIds cache within the getChangeMapIds method presents a potential race condition. If multiple threads were to access an uninitialized instance simultaneously, they could concurrently attempt to create and assign the cache map, leading to unpredictable behavior.
    -   **WARNING:** It is assumed that all interactions on a given world entity or chunk occur on a single, designated game thread. All public methods that mutate state or access world data must be synchronized externally or confined to a single-threaded execution context.

## API Surface

The primary contract is defined by its overrides of the parent SimpleBlockInteraction class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| interactWithBlock(...) | void | O(1) | Executes the core block-changing logic. Accesses world state, mutates a WorldChunk, and enqueues sound events to a CommandBuffer. This is the authoritative server-side action. |
| simulateInteractWithBlock(...) | void | O(1) | Performs a read-only check to see if an interaction would succeed. Used for client-side prediction or server-side validation without causing mutation. |
| configurePacket(Interaction) | void | O(1) | Serializes the interaction's configuration (the change map, sound index) into a network packet for client synchronization. |

## Integration Patterns

### Standard Usage

This class is not invoked directly. It is managed and executed by the server's core InteractionModule. The system identifies the relevant interaction based on player input and world state, then calls the appropriate methods on the configured instance.

```java
// Hypothetical invocation by the server's InteractionModule

// 1. The system retrieves the configured interaction instance from an asset registry
ChangeBlockInteraction interaction = interactionRegistry.get("my.namespace:dirt_to_path");

// 2. The system invokes the interaction logic with the current game context
interaction.interactWithBlock(world, commandBuffer, type, context, itemInHand, targetBlock, cooldownHandler);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never use `new ChangeBlockInteraction()`. This bypasses the codec-based initialization, resulting in a non-functional object with a null state. All instances must be created via asset loading.
-   **Post-Creation State Mutation:** Do not modify public or protected fields like blockTypeKeys after the object has been initialized. This will cause a desynchronization with the lazily-initialized `changeMapIds` cache and lead to incorrect behavior.
-   **Concurrent Execution:** Do not invoke `interactWithBlock` from multiple threads for the same world instance. This will corrupt chunk data and cause severe stability issues.

## Data Pipeline

The flow of data begins with game design configuration and ends with a mutated world state visible to players.

> Flow:
> 1.  **Asset File (JSON/HOCON)**: A designer defines a map of `{"source_block": "destination_block"}`.
> 2.  **BuilderCodec**: Deserializes the file into a `ChangeBlockInteraction` object during server startup.
> 3.  **Player Action**: A player interacts with a block in the world.
> 4.  **InteractionModule**: The server identifies the target block and determines this interaction should run.
> 5.  **interactWithBlock()**: The method is called.
> 6.  **getChangeMapIds()**: The string-based block names are resolved to integer IDs and cached (if not already).
> 7.  **WorldChunk.setBlock()**: The method directly mutates the chunk data, replacing the old block ID with the new one.
> 8.  **CommandBuffer**: A sound playback command is enqueued for processing.
> 9.  **Network Packet**: The world change is eventually serialized and sent to all relevant clients.

