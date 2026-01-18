---
description: Architectural reference for PortalWorld
---

# PortalWorld

**Package:** com.hypixel.hytale.builtin.portals.resources
**Type:** Transient

## Definition
```java
// Signature
public class PortalWorld implements Resource<EntityStore> {
```

## Architecture & Concepts
The PortalWorld class is not a game world itself, but rather a **stateful metadata resource** attached to a server-side EntityStore. It serves as the central authority and configuration hub for a single, instanced "portal world"—a temporary, often challenge-oriented, game instance.

Architecturally, this class acts as the bridge between static asset definitions (PortalType) and the dynamic, runtime state of an active portal. It encapsulates all transient data associated with a portal session, including its time limit, gameplay rules, player state, and links to related game events like a Void Event.

It is a critical component of the PortalsPlugin system, enabling the creation and management of isolated, temporary game loops that exist parallel to the main persistent world.

### Lifecycle & Ownership
The lifecycle of a PortalWorld instance is strictly coupled to the lifecycle of the World (and its associated EntityStore) to which it is attached.

-   **Creation:** A PortalWorld is never instantiated directly by gameplay code. It is created and attached to a new EntityStore by the core portal management system when a player or event triggers the creation of a new portal instance. The `init` method is then invoked immediately to populate the resource with configuration from a static PortalType asset.
-   **Scope:** The instance persists for the exact duration of the portal world's existence. It is a session-scoped object.
-   **Destruction:** The object is marked for garbage collection when its parent EntityStore is destroyed. This typically occurs when the `worldRemovalCondition` is met (e.g., the timer expires, all players leave, or the objective is completed).

## Internal State & Concurrency
-   **State:** PortalWorld is highly mutable. It maintains the core runtime state of the portal, including the remaining time (delegated to the PortalRemovalCondition strategy), collections of player UUIDs, and a reference to an optional, active Void Event. It effectively caches a resolved set of configurations derived from multiple asset sources upon initialization.
-   **Thread Safety:** This class is **not fully thread-safe** and requires careful management.
    -   The `diedInWorld` and `seesUi` sets are backed by a ConcurrentHashMap, making them safe for modification from multiple threads (e.g., a player death event on a worker thread).
    -   However, fields such as `spawnPoint` and `voidEventRef` are not synchronized. Access to these fields must be externally synchronized or, by convention, confined to the main server thread that ticks the parent World.

    **WARNING:** Unsynchronized, multi-threaded access to non-concurrent fields will lead to race conditions and unpredictable server state.

## API Surface
The public API provides methods for initializing, querying, and serializing the portal's state.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| init(PortalType, int, ...) | void | O(1) | Configures the resource post-creation. This is the effective constructor and must be called once. |
| getPortalType() | PortalType | O(1) | Retrieves the static asset definition for this portal world. May perform an asset map lookup. |
| getRemainingSeconds(World) | double | O(1) | Returns the time left before the world is destroyed. Delegates to the PortalRemovalCondition strategy. |
| setRemainingSeconds(World, double) | void | O(1) | Directly sets the remaining time. Use with caution as it overrides the natural countdown. |
| getDiedInWorld() | Set<UUID> | O(1) | Returns a thread-safe set of players who have died in this instance. |
| createFullPacket(World) | UpdatePortal | O(1) | Serializes the complete portal state into a network packet for initial client synchronization. |
| createUpdatePacket(World) | UpdatePortal | O(1) | Serializes the minimal state (time, event status) into a network packet for frequent updates. |

## Integration Patterns

### Standard Usage
A PortalWorld resource should always be retrieved from the EntityStore of an active world. It is never managed directly. Systems interact with it to check state or trigger updates.

```java
// A server system running on the world's tick
World portalInstance = ... // Obtain reference to the portal world
EntityStore store = portalInstance.getEntityStore();

// Retrieve the resource; may be null if not a portal world
PortalWorld portalState = store.getResource(PortalWorld.class);

if (portalState != null && portalState.exists()) {
    double remaining = portalState.getRemainingSeconds(portalInstance);
    if (remaining <= 0) {
        // Logic to end the portal session
    }

    // Send a state update to all relevant clients
    UpdatePortal packet = portalState.createUpdatePacket(portalInstance);
    // networkSystem.sendToPlayersInWorld(packet, portalInstance);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new PortalWorld()`. The resource system manages its lifecycle. Manual creation will result in an unmanaged, non-functional object that is not attached to any world.
-   **State Caching:** Do not cache the value from `getRemainingSeconds`. This value is dynamic and is expected to change on every server tick. Caching it will lead to stale logic.
-   **Cross-Thread Mutation:** Do not call setters like `setSpawnPoint` or `setVoidEventRef` from asynchronous tasks or network threads without explicit synchronization with the main world thread. This will corrupt the portal's state.

## Data Pipeline
PortalWorld primarily transforms static configuration into dynamic state and serializes that state for network transmission to the client.

> **Flow (State to Client):**
> World Tick Event -> System queries **PortalWorld** state -> **PortalWorld**.createUpdatePacket() -> Packet is serialized -> Server Network Layer -> Client Network Layer -> Client UI System updates portal timer display

