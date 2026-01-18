---
description: Architectural reference for WorldNotificationHandler
---

# WorldNotificationHandler

**Package:** com.hypixel.hytale.server.core.universe.world
**Type:** Transient

## Definition
```java
// Signature
public class WorldNotificationHandler {
```

## Architecture & Concepts
The WorldNotificationHandler is a critical server-side component that acts as the bridge between the abstract world simulation and the concrete network packets sent to clients. Its primary responsibility is to translate high-level world events, such as a block changing or a particle effect spawning, into the appropriate network protocol messages.

This class embodies the server's **relevancy management** system. It does not broadcast world updates indiscriminately. Instead, for every potential notification, it iterates through all connected players and uses each player's ChunkTracker to determine if the event occurred within a chunk currently loaded by that player's client. This filtering is the primary mechanism for conserving server bandwidth and reducing client-side processing load.

A key design pattern employed is the delegation of serialization and visibility logic to the BlockState objects themselves, specifically those implementing the SendableBlockState interface. The WorldNotificationHandler does not need to know the specific packet required for a custom block; it simply invokes methods like sendTo and canPlayerSee on the state object, allowing for extensible and encapsulated block behavior.

## Lifecycle & Ownership
- **Creation:** A WorldNotificationHandler is instantiated directly by its parent World object, typically during the World's own construction. It is provided a reference to the World it will manage notifications for.

- **Scope:** The lifecycle of a WorldNotificationHandler is strictly bound to its parent World. It persists for the entire duration that the World is loaded in memory.

- **Destruction:** The object is eligible for garbage collection when its parent World is unloaded and all references to it are released. It does not manage any native resources and has no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** This class is stateful, but its state consists of a single, immutable reference to the parent World instance. It does not maintain any other mutable state, such as queues or caches. Each method call operates statelessly on the provided arguments and the current state of the World.

- **Thread Safety:** **This class is not thread-safe.** Its methods iterate over the world's player collection. Any call to this handler must be synchronized with the main server world thread (the "tick" thread). Accessing this class from asynchronous tasks, network threads, or other worker threads without external locking will result in a high probability of ConcurrentModificationException and other unpredictable race conditions.

## API Surface
The public API provides methods to broadcast various world-related events to relevant players.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| updateState(x, y, z, state, oldState, skip) | void | O(P) | Broadcasts a block state change. Delegates packet creation to the state objects. O(P) where P is the number of players in the world. |
| updateChunk(indexChunk) | void | O(P) | Notifies all players to reload a specific chunk. This is a heavy operation and should be used sparingly. |
| sendBlockParticle(...) | void | O(P) | Creates and sends a SpawnBlockParticleSystem packet to all players for whom the target chunk is loaded. |
| updateBlockDamage(...) | void | O(P) | Creates and sends an UpdateBlockDamage packet to relevant players, typically used for visual feedback during block breaking. |
| sendPacketIfChunkLoaded(packet, indexChunk, filter) | void | O(P) | The core primitive for sending a pre-constructed packet to players who have a specific chunk loaded, with an optional filter. |

## Integration Patterns

### Standard Usage
The handler should always be retrieved from its parent World instance. Systems that modify the world, such as block physics or player actions, use the handler to notify clients of the changes.

```java
// A system that needs to change a block in the world
World world = ...;
BlockState newState = ...;
BlockState oldState = world.getBlockState(x, y, z);

// Update the world state first
world.setBlockState(x, y, z, newState);

// Then, use the world's handler to notify clients
world.getNotificationHandler().updateState(x, y, z, newState, oldState);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance of this class directly using `new WorldNotificationHandler()`. The World is responsible for creating and managing its own handler. Bypassing this will lead to a disconnected handler that does not function.

- **Asynchronous Modification:** Do not call any methods on this handler from a separate thread without synchronizing with the main world tick. This will corrupt the player list iteration and cause server instability.

- **Bypassing the Handler:** Avoid sending world update packets (like block changes) directly through a player's PacketHandler. Doing so bypasses the crucial relevancy checks, sending unnecessary data and potentially causing client-side desynchronization if the client does not have the required chunk loaded.

## Data Pipeline
The WorldNotificationHandler is a key stage in the data flow from server-side game logic to the client. It acts as a filter and translator.

> Flow:
> Server Logic (e.g., Block Physics Tick) -> **WorldNotificationHandler**.updateState() -> Iterates Players & Checks ChunkTracker -> PlayerRef.getPacketHandler().write(packet) -> Network Subsystem -> Client Render Engine

