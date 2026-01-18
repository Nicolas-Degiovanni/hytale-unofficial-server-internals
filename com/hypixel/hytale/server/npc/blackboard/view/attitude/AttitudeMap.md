---
description: Architectural reference for AttitudeMap
---

# AttitudeMap

**Package:** com.hypixel.hytale.server.npc.blackboard.view.attitude
**Type:** Data Structure / View

## Definition
```java
// Signature
public class AttitudeMap {
```

## Architecture & Concepts

The AttitudeMap is a performance-critical, read-optimized data structure that serves as the central truth for NPC-to-entity social relationships. It translates high-level configuration assets into a fast, integer-indexed lookup table that AI systems can query to determine their disposition—such as hostile, neutral, or friendly—towards any other entity.

Architecturally, this class acts as a bridge between the server's configuration/asset system and the real-time AI decision-making loop. During server initialization, the `AttitudeMap.Builder` consumes human-readable `AttitudeGroup` assets, which define relationships using string identifiers. The builder resolves these strings into integer-based group IDs and pre-computes the entire relationship matrix.

The final AttitudeMap object is an immutable-by-default container holding this matrix. The core data structure is an array of maps (`Int2ObjectMap<Attitude>[]`), allowing for a two-step O(1) lookup:
1.  Index the outer array by the querier's `AttitudeGroup` ID.
2.  Index the inner map by the target's `NPCGroup` ID.

This design ensures that the expensive work of string parsing, asset resolution, and relationship calculation is performed once at startup, allowing the frequent, in-game `getAttitude` calls to be extremely fast.

### Lifecycle & Ownership
-   **Creation:** An instance is created exclusively via the `AttitudeMap.Builder`. This process is typically orchestrated by a higher-level manager, such as the `NPCPlugin` or a world loading service, during the server's bootstrap sequence. The builder is populated with all `AttitudeGroup` configurations loaded from game assets.
-   **Scope:** The AttitudeMap is a long-lived, session-scoped object. It is created once when the server or world initializes and persists until shutdown. It is not tied to any single NPC entity but rather represents the global rules for all NPC interactions.
-   **Destruction:** The object has no explicit destruction or cleanup method. It is garbage collected when its owning manager (e.g., the world instance) is destroyed upon server shutdown.

## Internal State & Concurrency
-   **State:** The primary state is the `map` field, an array of `Int2ObjectMap`. While the `map` field itself is `final`, the array it references is mutable. The `updateAttitudeGroup` method allows for hot-swapping one of the inner maps, making the overall object state mutable. This is intended for live configuration reloads.
-   **Thread Safety:** This class is **not** thread-safe for concurrent reads and writes.
    -   The primary read method, `getAttitude`, performs no locking and is safe for concurrent reads.
    -   The write method, `updateAttitudeGroup`, replaces an element in the internal array. This array slot assignment is an atomic operation in Java.
    -   **WARNING:** A thread calling `getAttitude` concurrently with a call to `updateAttitudeGroup` may receive either the old or the new attitude data for that group, but will not see a corrupted state. However, there is no memory barrier, so visibility of the update is not guaranteed without external synchronization. It is designed for a single writer thread (e.g., the main server thread handling a reload command) and multiple reader threads (AI ticks).

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAttitude(role, target, accessor) | Attitude | O(1) | Retrieves the attitude of an NPC (via its Role) towards a target entity. Returns null if no specific attitude is defined. |
| getAttitudeGroupCount() | int | O(1) | Returns the total number of configured attitude groups. |
| updateAttitudeGroup(id, group) | void | O(N) | Replaces the attitude mapping for a single group ID. This is a high-cost operation intended for configuration reloads. N is the total number of entities in all referenced NPC groups. |

## Integration Patterns

### Standard Usage
The AttitudeMap should be treated as a read-only service by game logic. AI systems, such as Behavior Trees or State Machines, query it to make decisions. It is retrieved from a central context or service registry, never constructed directly.

```java
// Within an AI Behavior Node or update tick
// Assume 'world' or 'context' provides access to the AttitudeMap
AttitudeMap attitudeMap = world.getNpcManager().getAttitudeMap();

// Determine attitude towards a potential target
Attitude disposition = attitudeMap.getAttitude(myRole, targetEntityRef, componentAccessor);

if (disposition == Attitude.HOSTILE) {
    // Initiate combat logic
}
```

### Anti-Patterns (Do NOT do this)
-   **Builder in Game Logic:** Do not use the `AttitudeMap.Builder` outside of the server's initial loading sequence. Constructing an AttitudeMap is a heavyweight process designed to be done once.
-   **Frequent Updates:** The `updateAttitudeGroup` method is not designed for dynamic, gameplay-driven faction changes. Calling it frequently, such as every tick or in response to a player action, will cause significant performance degradation. Use a different system for dynamic reputation.
-   **Unsynchronized Concurrent Writes:** Calling `updateAttitudeGroup` from a worker thread without proper synchronization with the main game loop or AI threads can lead to unpredictable behavior due to memory visibility issues. All updates should be managed by a single, authoritative thread.

## Data Pipeline
The AttitudeMap is the end product of a data transformation pipeline that converts static configuration files into a queryable, in-memory object.

> Flow:
> Game Assets (`.json` files defining AttitudeGroups) -> Server Asset Loader -> **AttitudeMap.Builder** (consumes asset objects, resolves string IDs to integers) -> **AttitudeMap** (in-memory instance) -> AI Behavior Node (calls `getAttitude`) -> Attitude Enum (drives AI decision)

