---
description: Architectural reference for PrefabEditingMetadata
---

# PrefabEditingMetadata

**Package:** com.hypixel.hytale.builtin.buildertools.prefabeditor
**Type:** Transient Data Model

## Definition
```java
// Signature
public class PrefabEditingMetadata {
```

## Architecture & Concepts
The PrefabEditingMetadata class is a stateful data model that represents a single, active prefab editing session on the server. It acts as the authoritative source of truth for the properties of a prefab being manipulated by a user, such as its bounding box, file path, and anchor point.

Its primary architectural role is to bridge the gap between the abstract, file-based definition of a prefab and its tangible, in-world representation. It achieves this by managing a dedicated, visible "anchor" entity within the game world. This entity provides immediate visual feedback to the user about the prefab's origin and placement.

This class is designed for persistence. The static CODEC field enables the entire state of an editing session—including the positions and unique identifiers—to be serialized and deserialized. This allows for features like saving and resuming complex building tasks. All world-mutating operations, such as creating or moving the anchor entity, are correctly marshaled onto the main server thread via the World.execute method, ensuring safe interaction with the core game loop.

### Lifecycle & Ownership
- **Creation:** An instance is created by a higher-level management system, such as a PrefabEditorService, when a user initiates an editing session for a specific prefab. The public constructor requires a valid World object, as it immediately triggers a side-effect: the creation of the visual anchor entity in the game world.
- **Scope:** The object's lifetime is bound to the duration of the user's editing session. A unique instance exists for each prefab being actively edited.
- **Destruction:** The owning service is responsible for cleanup. When the editing session ends (e.g., the user saves or cancels), the owner must manually remove the associated anchor entity from the World. The PrefabEditingMetadata object is then dereferenced and becomes eligible for garbage collection. There is no internal destructor or cleanup method.

## Internal State & Concurrency
- **State:** This object is highly mutable. It maintains the core state of the editing session, including coordinates, file paths, and the UUID of the live anchor entity. It also contains a boolean *dirty* flag to track unsaved changes.
- **Thread Safety:** **This class is not thread-safe.** Its internal state is not protected by locks or other synchronization primitives. All method calls that mutate state, such as setAnchorPoint or setDirty, must be executed on the main server thread to prevent race conditions and data corruption. The class ensures that its own interactions with the World are thread-safe by using World.execute, but it does not protect itself from improper concurrent access by external callers.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| PrefabEditingMetadata(Path, Vector3i, ...) | constructor | O(N) | Instantiates the session state and spawns the initial anchor entity in the world. Throws IllegalStateException if bounds are invalid. |
| setAnchorPoint(newPosition, world) | void | O(N) | Updates the logical anchor and moves the in-world visual entity. This is an expensive operation involving entity destruction and creation. |
| recreateAnchorEntity(world) | void | O(N) | Re-spawns the anchor entity using the current state. Typically used after deserializing a saved session. |
| sendAnchorHighlightingPacket(displayTo) | void | O(1) | Sends a network packet to a specific client to trigger a visual effect on the anchor. |
| isLocationWithinPrefabBoundingBox(location) | boolean | O(1) | Performs a fast, mathematical check to see if a point is inside the prefab's defined volume. |
| isReadOnly() | boolean | O(1) | Checks if the underlying file system for the prefab path is read-only, preventing modifications to core game assets. |

## Integration Patterns

### Standard Usage
The class should be instantiated and managed by a service that has access to the server's World context. The service is responsible for the object's entire lifecycle.

```java
// In a server-side service that handles builder tools
World world = server.getWorld();
Path prefabToEdit = Paths.get("prefabs/my_structure.pfb");

// 1. Create the metadata, which spawns the anchor entity
PrefabEditingMetadata session = new PrefabEditingMetadata(
    prefabToEdit,
    minBounds,
    maxBounds,
    anchor,
    pastePosition,
    world
);

// 2. Later, in response to player input, update the anchor
Vector3i newPlayerPosition = player.getPosition();
session.setAnchorPoint(newPlayerPosition, world);
session.setDirty(true);

// 3. On session end, the service must clean up the entity
// (This logic is external to PrefabEditingMetadata)
```

### Anti-Patterns (Do NOT do this)
- **State Mutation from Worker Threads:** Never call methods like setAnchorPoint or setMinPoint from an asynchronous task or a different thread. All interactions must be on the main server thread.
- **Leaking Anchor Entities:** Do not dereference a PrefabEditingMetadata object without first ensuring its associated anchor entity (retrieved via getAnchorEntityUuid) has been removed from the World. Failure to do so will leave orphaned entities in the game.
- **Reusing Instances:** Do not attempt to reuse a single PrefabEditingMetadata instance for a different editing session. Always create a new instance when a user starts editing a new or different prefab.

## Data Pipeline
The flow of data for a core user interaction, such as moving the prefab anchor, follows this sequence. The PrefabEditingMetadata object acts as the central state coordinator for the operation.

> Flow:
> Client Input Command -> Server Network Handler -> PrefabEditorService -> **PrefabEditingMetadata.setAnchorPoint()** -> World.execute(lambda) -> EntityStore removes old entity & adds new entity -> Network sync to clients -> Player sees updated anchor position

