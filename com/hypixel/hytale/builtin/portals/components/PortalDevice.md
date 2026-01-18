---
description: Architectural reference for PortalDevice
---

# PortalDevice

**Package:** com.hypixel.hytale.builtin.portals.components
**Type:** Data Component

## Definition
```java
// Signature
public class PortalDevice implements Component<ChunkStore> {
```

## Architecture & Concepts
The **PortalDevice** is a data component that represents the state and configuration of a single portal structure within the game world. In Hytale's component-based architecture, this class does not contain any active logic. Instead, it serves as a plain data container that is attached to a **ChunkStore**, effectively anchoring the portal's data to a specific chunk in the world.

Its primary architectural role is to bridge a physical location in one world to a destination in another. It achieves this by storing a configuration object (**PortalDeviceConfig**), a visual identifier (**baseBlockTypeKey**), and a crucial link to another world (**destinationWorldUuid**).

The static **CODEC** field is fundamental to its design. It enables the engine's serialization system to read and write the portal's state to disk as part of the world save format. This ensures that portals are persistent across server restarts and chunk loading cycles.

## Lifecycle & Ownership
- **Creation:** A **PortalDevice** instance is created under two circumstances:
    1.  **Programmatically:** By a higher-level system, such as one that manages world generation or player actions, when a new portal is constructed in the world. This involves calling the public constructor `new PortalDevice(config, key)`.
    2.  **Deserialization:** By the persistence layer when a chunk containing a portal is loaded from disk. The engine uses the static **CODEC** to reconstruct the **PortalDevice** object from its serialized representation.

- **Scope:** The lifecycle of a **PortalDevice** is strictly tied to the **ChunkStore** component to which it is attached. It remains in memory only as long as the corresponding world chunk is active on the server.

- **Destruction:** The object is eligible for garbage collection when its parent **ChunkStore** is unloaded from memory. There are no manual destruction or cleanup methods.

## Internal State & Concurrency
- **State:** The internal state of a **PortalDevice** is **mutable**. Key properties, most notably the **destinationWorldUuid**, can be changed at runtime via methods like **setDestinationWorld**. The state represents the live, current configuration of a specific portal instance.

- **Thread Safety:** This class is **not thread-safe**. It contains no internal locking mechanisms or concurrency controls. All reads and writes to a **PortalDevice** instance must be synchronized externally.

    **WARNING:** It is critically important to ensure that all interactions with a **PortalDevice** component occur on the main server thread to prevent race conditions, inconsistent state, and unpredictable behavior, especially when another thread might be unloading the destination world.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | ComponentType | O(1) | Static method to retrieve the registered type definition for this component. |
| getBaseBlockType() | BlockType | O(1) | Resolves the base block type key into a concrete **BlockType** asset from the global asset map. |
| getDestinationWorld() | World | O(1) | Resolves the destination UUID into a live **World** object from the global **Universe**. Returns null if the UUID is not set or the target world is not loaded or alive. |
| setDestinationWorld(World) | void | O(1) | Sets the portal's destination by storing the UUID of the provided **World** object. This is the primary method for linking portals. |
| clone() | Component | O(1) | Creates a shallow copy of the component. Used by the engine for component duplication. |

## Integration Patterns

### Standard Usage
The **PortalDevice** is intended to be retrieved from a **ChunkStore** by a governing system that implements portal logic. The system reads the component's data to perform actions, such as teleporting an entity.

```java
// A hypothetical PortalSystem processing a chunk
ChunkStore targetChunk = world.getChunkStore(chunkPosition);
PortalDevice portal = targetChunk.getComponent(PortalDevice.getComponentType());

if (portal != null && portal.getDestinationWorld() != null) {
    World destination = portal.getDestinationWorld();
    // Logic to teleport an entity to the destination world...
}
```

### Anti-Patterns (Do NOT do this)
- **Long-Lived References:** Do not retrieve a **World** object via **getDestinationWorld** and store it for an extended period. The destination world can be unloaded at any time, making the stored reference invalid. Always re-query the destination world just before it is needed.
- **Unmanaged Instantiation:** Do not create a **PortalDevice** with *new* and fail to attach it to a **ChunkStore**. An unattached component is inert and will be garbage collected without ever being used by the game systems.
- **Cross-Thread Modification:** Never call **setDestinationWorld** or modify other state from an asynchronous task or worker thread. All mutations must be marshaled back to the main server thread.

## Data Pipeline
The **PortalDevice** is a data payload, not a processing stage. Its primary data flow relates to persistence (serialization and deserialization).

> **Serialization Flow:**
> **PortalDevice** (In-Memory Object) -> **BuilderCodec** -> Serialized Data (Binary/NBT) -> **ChunkStore** -> World Save File (Disk)

> **Deserialization Flow:**
> World Save File (Disk) -> **ChunkStore** -> Serialized Data (Binary/NBT) -> **BuilderCodec** -> **PortalDevice** (In-Memory Object)

