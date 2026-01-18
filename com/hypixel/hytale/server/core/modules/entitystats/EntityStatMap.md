---
description: Architectural reference for EntityStatMap
---

# EntityStatMap

**Package:** com.hypixel.hytale.server.core.modules.entitystats
**Type:** Component

## Definition
```java
// Signature
public class EntityStatMap implements Component<EntityStore> {
```

## Architecture & Concepts

The EntityStatMap is the authoritative, server-side state manager for all numerical statistics of a single game entity. It functions as a component within Hytale's entity-component system, holding runtime data for attributes like health, mana, speed, and damage.

Its core architectural purpose is twofold:
1.  **State Management:** It provides a high-performance, index-based container for an entity's `EntityStatValue` objects, which are the live instances of stats defined by `EntityStatType` assets.
2.  **Network Replication:** It implements a sophisticated change-tracking and delta-compression system. Every mutation to a stat is recorded as a discrete `EntityStatUpdate` operation. This log is consumed by the network layer to broadcast efficient, minimal-size packets to clients, rather than synchronizing the entire state object.

The system distinguishes between updates for the entity's owner (selfUpdates) and updates for all other observing clients (otherUpdates). This allows for sending more detailed or predictive information to the controlling player. The `Predictable` enum is a key part of this, enabling client-side prediction by flagging operations that the client can safely simulate before receiving server confirmation, thus reducing perceived latency.

Internally, the class maintains synchronization with the global `EntityStatType` asset table. The `update` method reconciles the entity's current stats with the game's asset definitions, handling data migration for newly added or removed stats between game updates.

### Lifecycle & Ownership
-   **Creation:** An EntityStatMap is instantiated and attached to an entity when that entity is created by the server's `EntityStore`. It is also instantiated during deserialization from persistent storage or a network snapshot, a process governed by its static `CODEC` field.
-   **Scope:** The lifecycle of an EntityStatMap is strictly bound to the entity it belongs to. It persists for the entire duration of the entity's existence within the game world.
-   **Destruction:** The component is marked for garbage collection when its parent entity is destroyed and removed from the `EntityStore`. There are no external systems that hold persistent references to it.

## Internal State & Concurrency
-   **State:** This component is highly mutable. Its primary state consists of the `values` array of `EntityStatValue` objects. It also maintains several transient, mutable fields for network synchronization: `selfUpdates`, `otherUpdates`, `selfStatValues`, `isSelfNetworkOutdated`, and `isNetworkOutdated`. These fields act as a temporary change log that is periodically consumed and cleared by the networking system.

-   **Thread Safety:** **This class is not thread-safe.** All interactions with an EntityStatMap instance must be performed on the main server thread that owns the entity. The internal data structures (e.g., `Int2ObjectOpenHashMap`, `ObjectArrayList`) are not synchronized. Unmanaged concurrent access will lead to race conditions, data corruption, and server instability.

    **WARNING:** Do not access or modify an EntityStatMap from asynchronous tasks, worker threads, or parallel streams without explicit, external synchronization managed by the core game loop.

## API Surface
The public API is designed for stat mutation and network data consumption.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| update() | void | O(A) | Synchronizes the map with global `EntityStatType` assets. Handles data migration. |
| get(index) | EntityStatValue | O(1) | Retrieves the stat value object at the specified index. |
| putModifier(index, key, modifier) | Modifier | O(log N) | Adds or replaces a modifier on a stat and logs the change for network replication. |
| removeModifier(index, key) | Modifier | O(log N) | Removes a modifier from a stat and logs the change. |
| setStatValue(index, newValue) | float | O(1) | Directly sets the base value of a stat and logs the change. |
| addStatValue(index, amount) | float | O(1) | Adds to the base value of a stat and logs the change. |
| consumeSelfUpdates() | Int2ObjectMap | O(N) | Destructively reads the change log for the entity owner. Called by the network layer. |
| consumeOtherUpdates() | Int2ObjectMap | O(N) | Destructively reads the change log for other clients. Called by the network layer. |
| consumeSelfNetworkOutdated() | boolean | O(1) | Returns true if the self-update log has new data, then resets the flag. |
| consumeNetworkOutdated() | boolean | O(1) | Returns true if the other-update log has new data, then resets the flag. |

*Complexity: A = Total number of defined stat assets, N = Number of modifiers on a single stat.*

## Integration Patterns

### Standard Usage
Interaction with the EntityStatMap should always be performed through a reference to an existing entity. Game logic, such as a combat system, retrieves the component and calls its mutation methods.

```java
// Example: A combat system applying damage to an entity
void applyDamage(Entity target, float damageAmount) {
    EntityStatMap stats = target.getComponent(EntityStatMap.getComponentType());
    if (stats != null) {
        // The call to subtractStatValue automatically queues the change for networking.
        int healthStatIndex = EntityStatType.getAssetMap().getIndex("hytale:health");
        stats.subtractStatValue(healthStatIndex, damageAmount);
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new EntityStatMap()`. The component's lifecycle is exclusively managed by the entity system to ensure it is correctly initialized and registered.
-   **State Polling:** Do not poll stat values every frame to check for changes. The `consume...Updates` methods provide a reactive, event-driven mechanism for detecting and broadcasting changes.
-   **Update Hoarding:** The network system must call `consumeSelfUpdates` and `consumeOtherUpdates` regularly. Failure to do so will result in a memory leak, as the internal change log lists will grow indefinitely.
-   **Cross-Thread Mutation:** As stated in the concurrency section, never modify an EntityStatMap from a thread other than the one that owns the entity's game tick.

## Data Pipeline
The flow of a single stat modification from game logic to a remote client is a well-defined pipeline orchestrated by this component.

> Flow:
> Game Logic (e.g., Combat System) -> `EntityStatMap.addStatValue()` -> Internal `EntityStatValue` is mutated -> An `EntityStatUpdate` object is created and queued in `otherUpdates` -> `isNetworkOutdated` flag is set to true -> Server Network System calls `consumeOtherUpdates()` -> The queue of `EntityStatUpdate` objects is serialized into a network packet -> Packet is sent to relevant clients.

