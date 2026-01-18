---
description: Architectural reference for UseWateringCanInteraction
---

# UseWateringCanInteraction

**Package:** com.hypixel.hytale.builtin.adventure.farming.interactions
**Type:** Configuration / Behavior Object

## Definition
```java
// Signature
public class UseWateringCanInteraction extends SimpleBlockInteraction {
```

## Architecture & Concepts
The UseWateringCanInteraction class is a server-authoritative behavior definition that encapsulates the logic for watering farmable blocks. It is not a persistent service but rather a data-driven object that represents a specific, configurable type of player-world interaction.

Architecturally, it functions as a concrete strategy within the server's broader Interaction Module. It extends SimpleBlockInteraction, inheriting a framework for handling interactions targeted at a specific block. Its primary role is to bridge a player action with a state change in the world's component system.

The static CODEC field is the most critical architectural feature. It allows instances of this class to be defined and configured entirely within external data files (e.g., JSON). This decouples the game logic (watering a block) from the specific parameters (how long it stays watered), enabling designers to tune game mechanics without requiring code changes.

This interaction is explicitly server-authoritative, as indicated by its getWaitForDataFrom method returning Server. The client performs no simulation and trusts the server to process the action and return the resulting world state.

## Lifecycle & Ownership
- **Creation:** Instances are not created dynamically per-interaction. A single instance representing this behavior is deserialized and instantiated by the engine's Codec system during the server's asset loading phase. Its properties, like duration, are populated from game data files at this time.
- **Scope:** Application-scoped. Once loaded, the instance is stored in a central registry and persists for the entire server session. It is stateless with respect to any single interaction event.
- **Destruction:** The object is garbage collected when the server shuts down and its interaction registries are cleared.

## Internal State & Concurrency
- **State:** The internal state, consisting of the duration and refreshModifiers fields, is considered immutable after the initial asset loading phase. The object itself holds no state related to a specific ongoing interaction; all necessary context is passed as arguments to the interactWithBlock method.
- **Thread Safety:** This class is not thread-safe by design and expects to be executed exclusively on the main server thread. The interactWithBlock method directly mutates world state (ChunkStore, WorldChunk components), which is a non-atomic operation. Calling this method from any other thread will lead to world corruption, race conditions, and server instability. The engine's game loop is responsible for ensuring thread containment.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getWaitForDataFrom() | WaitForDataFrom | O(1) | Returns a constant indicating that the server is the source of truth for this interaction's outcome. |
| interactWithBlock(...) | void | O(C) | Executes the core watering logic. Locates the target block and its underlying soil, modifies their TilledSoilBlock component, and schedules a future block tick for when the watered state expires. Throws exceptions on invalid world state. |

## Integration Patterns

### Standard Usage
A developer or designer does not invoke this class directly in code. Instead, it is configured in an item's definition file. The server's InteractionModule automatically resolves the interaction and dispatches the call.

The following conceptual example illustrates how the system would invoke this handler:

```java
// Conceptual code within the server's InteractionModule
// This is not code a developer would write, but shows the pattern.

void processPlayerInteraction(Player player, Vector3i targetBlock) {
    // 1. System identifies the item in hand and its associated interaction handlers.
    Interaction handler = getInteractionForItem(player.getHeldItem());

    // 2. System verifies the handler is a UseWateringCanInteraction.
    if (handler instanceof UseWateringCanInteraction) {
        // 3. The handler's method is invoked with the current world context.
        // The 'handler' instance was loaded from a config file at startup.
        handler.interactWithBlock(world, commandBuffer, ...);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use new UseWateringCanInteraction(). This bypasses the critical data-driven configuration loaded by the CODEC, resulting in an unconfigured object with a zero duration.
- **Asynchronous Execution:** Do not call interactWithBlock from a separate thread or asynchronous task. All world modifications must occur on the main server thread to prevent data corruption.
- **State Modification:** Do not attempt to modify the public duration field after the server has started. This is configuration data that should be treated as read-only at runtime.

## Data Pipeline
The flow of data and control for this interaction is linear and server-driven. The client's role is limited to initiating the request.

> Flow:
> Player Input on Client -> Network Packet to Server -> Server InteractionModule -> **UseWateringCanInteraction.interactWithBlock** -> Component Lookup (ChunkStore) -> Component State Mutation (TilledSoilBlock) -> World Tick Scheduler -> State Replication to Client

