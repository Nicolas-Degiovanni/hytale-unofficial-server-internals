---
description: Architectural reference for PlayerUtil
---

# PlayerUtil

**Package:** com.hypixel.hytale.server.core.universe.world
**Type:** Utility

## Definition
```java
// Signature
public class PlayerUtil {
```

## Architecture & Concepts
PlayerUtil is a stateless, server-side utility class that provides a high-level API for common operations involving players within a world. It acts as a crucial facade, abstracting the complexity of the underlying Entity Component System (ECS) and networking layers into simple, static methods.

Its primary responsibilities are:
- **Targeted Broadcasting:** Encapsulating the logic for iterating players who can visually perceive a specific entity. This is fundamental for network culling, ensuring that entity-related packets are only sent to relevant clients, thereby conserving server bandwidth and client-side processing power.
- **Global Broadcasting:** Providing convenient methods to send packets and messages to all players currently connected to a world.
- **Player State Manipulation:** Offering helpers for common player-related component modifications, such as model and skin updates.

This class is deeply integrated with the server's core ECS data structures, operating directly on objects like ComponentAccessor, Store, and Ref. It is not a standalone service but a collection of reusable functions intended to be called from within other server systems.

## Lifecycle & Ownership
- **Creation:** As a class containing only static methods, PlayerUtil is never instantiated. Its bytecode is loaded by the Java ClassLoader when it is first referenced by another part of the server application.
- **Scope:** The utility methods are globally accessible and persist for the entire lifetime of the server process.
- **Destruction:** The class is unloaded from memory when the Java Virtual Machine shuts down. There is no instance-level state to manage or clean up.

## Internal State & Concurrency
- **State:** PlayerUtil is entirely **stateless**. It contains no member variables and all operations are performed on data passed in as method arguments. Its behavior is deterministic based on its inputs.
- **Thread Safety:** The methods themselves are inherently thread-safe as they are pure functions with no internal state. However, the ECS data structures they operate on, such as the Store and ComponentAccessor, are **not thread-safe**.

**WARNING:** All methods in PlayerUtil must be called from the main server thread (the world's tick loop). Invoking these methods from an asynchronous task or a different thread without explicit synchronization with the world state will lead to severe data corruption, race conditions, and ConcurrentModificationExceptions.

## API Surface
The public API provides coarse-grained operations for player and entity interactions.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| forEachPlayerThatCanSeeEntity(...) | void | O(P) | Executes a consumer for each player whose EntityViewer component contains the target entity. P is the total number of players in the world. |
| broadcastMessageToPlayers(...) | void | O(P) | Sends a Message to all players, respecting the HiddenPlayersManager for the source player. |
| broadcastPacketToPlayers(...) | void | O(P) | Sends one or more Packets to all players in the world via their PacketHandler. |
| broadcastPacketToPlayersNoCache(...) | void | O(P) | Sends a Packet to all players, bypassing the packet handler's cache. |
| resetPlayerModel(...) | void | O(1) | **Deprecated.** Re-evaluates a player's skin and updates their ModelComponent. |

## Integration Patterns

### Standard Usage
PlayerUtil methods should be called from within server-side systems that need to communicate state changes to clients. The typical pattern involves acquiring a ComponentAccessor and then invoking the required static method.

```java
// Example: A system that broadcasts a sound effect when an entity does something.
// This code would exist inside a system's update method.

// Get the ComponentAccessor for the current world tick
ComponentAccessor<EntityStore> accessor = world.getComponentAccessor();
Ref<EntityStore> someEntityRef = ...; // The entity that made the sound

// Create the packet to send
SoundEffectPacket soundPacket = new SoundEffectPacket(...);

// Use PlayerUtil to send the packet only to players who can see the entity
PlayerUtil.forEachPlayerThatCanSeeEntity(someEntityRef, (playerRef, player, commands) -> {
    player.getPacketHandler().write(soundPacket);
}, accessor);
```

### Anti-Patterns (Do NOT do this)
- **Asynchronous Invocation:** Never call PlayerUtil methods from a separate thread (e.g., a database callback or a network I/O thread) without dispatching the call back to the main server thread. This will corrupt the ECS state.
- **Manual Visibility Checks:** Avoid writing your own loops to iterate all players and check visibility. The forEachPlayerThatCanSeeEntity method is optimized to iterate through ECS chunks efficiently and should always be preferred.
- **Ignoring ComponentAccessor:** Do not hold onto a ComponentAccessor across multiple server ticks. Always retrieve the current one for the tick you are processing before passing it to PlayerUtil.

## Data Pipeline
PlayerUtil sits between high-level game logic and the low-level ECS data and networking layers. It translates a high-level intent (e.g., "tell everyone about this") into a series of low-level operations.

**Global Broadcast Pipeline:**
> Game System Logic -> **PlayerUtil.broadcastPacketToPlayers** -> World.getPlayerRefs() -> [For each PlayerRef] -> PlayerRef.getPacketHandler().write() -> Network Serialization -> Client

**Visibility-Filtered Pipeline:**
> Game System Logic -> **PlayerUtil.forEachPlayerThatCanSeeEntity** -> Store.forEachChunk(PlayerRef) -> Filter by EntityViewer Component -> Execute Callback -> PlayerRef.getPacketHandler().write() -> Network Serialization -> Client

