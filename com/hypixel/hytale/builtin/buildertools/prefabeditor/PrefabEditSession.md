---
description: Architectural reference for PrefabEditSession
---

# PrefabEditSession

**Package:** com.hypixel.hytale.builtin.buildertools.prefabeditor
**Type:** Session State Object

## Definition
```java
// Signature
public class PrefabEditSession implements Resource<EntityStore> {
```

## Architecture & Concepts
The PrefabEditSession is the central, server-side state container for a prefab editing environment. It is not a world itself, but rather a managed **Resource** attached to the temporary `EntityStore` that constitutes the editing world. Its primary responsibility is to track the lifecycle, state, and player interactions for all prefabs loaded within that isolated session.

This class acts as the authoritative model for the editing environment. It holds metadata for each loaded prefab, including its source path, in-world boundaries, and modification status. It also manages player-specific state, such as which prefab a given player has currently selected for manipulation.

A critical architectural feature is its implementation of the `Resource` interface and its associated `CODEC`. This design ensures that the entire state of a prefab editing session is serializable. When the editing world is saved, this object is persisted along with it. Upon loading, the `afterDecode` hook in its codec re-establishes its relationship with the global `PrefabEditSessionManager`, ensuring seamless session restoration.

## Lifecycle & Ownership
-   **Creation:** A PrefabEditSession is instantiated exclusively by the `PrefabEditSessionManager` when a user initiates an editing flow. This typically involves creating a new, isolated world and attaching a new session instance to it as a resource. Direct instantiation is a critical anti-pattern.

-   **Scope:** The object's lifetime is strictly bound to the `EntityStore` (the editing world) to which it is attached. It persists as long as the editing session is active, whether the world is loaded in memory or unloaded to disk.

-   **Destruction:** The session is marked for garbage collection when the editing world is permanently deleted. The `PrefabEditSessionManager` orchestrates this cleanup process when a session is formally concluded.

## Internal State & Concurrency
-   **State:** This object is highly mutable. It maintains a collection of `PrefabEditingMetadata` objects, which track the state of each individual prefab. It also maps player UUIDs to their selected prefab UUIDs. Methods like `updatePrefabBounds` and `markPrefabsDirtyAtPosition` modify this internal state and flag contained objects as "dirty", signaling the need for persistence.

-   **Thread Safety:** **This class is not thread-safe.** All interactions with a PrefabEditSession instance must be performed on the main server thread that owns the corresponding `EntityStore`. Its internal collections, such as `Object2ObjectOpenHashMap`, are not concurrent. Unsynchronized access from other threads will result in state corruption, race conditions, and server instability.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| addPrefab(path, min, max, anchor, paste) | void | O(1) | Adds a new prefab's metadata to the session. |
| setSelectedPrefab(ref, metadata, accessor) | void | O(1) | Sets the active prefab for a player. Triggers network packets and server-side selection commands. |
| clearSelectedPrefab(ref, accessor) | boolean | O(1) | Clears a player's active prefab selection. Triggers network packets and deselection commands. |
| updatePrefabBounds(uuid, newMin, newMax) | PrefabEditingMetadata | O(1) | Modifies the bounding box of a loaded prefab and marks it as dirty. |
| markPrefabsDirtyInBounds(min, max) | void | O(N) | Marks all prefabs intersecting the given bounds as dirty. N is the number of loaded prefabs. |
| createPrefabMarkers() | MapMarker[] | O(N) | Generates map marker data for all loaded prefabs, used for UI integration. |

## Integration Patterns

### Standard Usage
A PrefabEditSession should never be accessed directly. It must be retrieved as a resource from the world's `EntityStore`. All operations are performed through this retrieved instance.

```java
// Assuming 'world' is the EntityStore for the editing session
PrefabEditSession session = world.getResource(PrefabEditSession.getResourceType());

if (session != null) {
    // Mark any prefabs in a region as modified
    session.markPrefabsDirtyInBounds(minVector, maxVector);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new PrefabEditSession()`. Doing so creates an unmanaged, "orphaned" object that is not registered with the `PrefabEditSessionManager` or the world's resource system. This object will not be persisted and will fail to integrate with other game systems.

-   **State Mutation without Dirtying:** Avoid retrieving the internal `PrefabEditingMetadata` map and modifying its contents directly. Always use provided methods like `updatePrefabBounds`, which ensure the object's dirty flag is correctly set for persistence. Failure to do so will result in lost data.

-   **Cross-Thread Access:** Do not cache a reference to a PrefabEditSession and access it from asynchronous tasks or different threads. All calls must be synchronized with the main server tick for the world it belongs to.

## Data Pipeline
The PrefabEditSession acts as a central controller in the data flow for prefab editing actions.

**Player Selects a Prefab:**
> Player Input -> Server Command -> `PrefabEditSessionManager` -> **`PrefabEditSession.setSelectedPrefab`** -> `BuilderToolsPlugin.addToQueue` -> `BlockSelection` State Update

**Client-Side Visual Feedback:**
> **`PrefabEditSession.setSelectedPrefab`** -> `PrefabEditingMetadata.sendAnchorHighlightingPacket` -> `PacketHandler` -> Network Layer -> Client-side Anchor Rendering

