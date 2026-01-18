---
description: Architectural reference for BlockMapMarker
---

# BlockMapMarker

**Package:** com.hypixel.hytale.server.core.universe.world.meta.state
**Type:** Data Component

## Definition
```java
// Signature
public class BlockMapMarker implements Component<ChunkStore> {
```

## Architecture & Concepts
The BlockMapMarker is a pure data component within the server's Entity Component System (ECS). It serves a single purpose: to attach world map metadata (a name and an icon) to a specific block entity within the game world.

This class itself contains no logic. Its behavior is entirely driven by two associated systems that observe its presence on an entity:
1.  **OnAddRemove System:** This system acts as a controller that listens for the creation and destruction of entities possessing a BlockMapMarker. Upon creation, it extracts the marker data and the block's world position, writing this information into a centralized, world-global resource named BlockMapMarkersResource. This is a critical optimization; instead of querying every block in the world for a marker, the system aggregates the data into a single, efficient lookup structure.
2.  **MarkerProvider System:** This system reads from the aggregated BlockMapMarkersResource. It is periodically invoked by the WorldMapManager to generate and transmit map marker network packets to clients. This decouples the client update logic from the block entity lifecycle, ensuring that map updates are efficient and batched.

In essence, BlockMapMarker is the trigger and data source for a one-way data flow that makes block-specific information available to the global world map system.

### Lifecycle & Ownership
-   **Creation:** A BlockMapMarker is not instantiated directly. It is added as a component to a block entity, typically during world generation, through a gameplay script, or when a player places a specific type of block. The ECS framework manages its instantiation. The OnAddRemove system is immediately notified of its addition.
-   **Scope:** The component's lifetime is strictly bound to the block entity it is attached to. It persists as long as the block exists in the world.
-   **Destruction:** When the parent block entity is destroyed (e.g., mined by a player), the BlockMapMarker component is removed. The OnAddRemove system intercepts this event and issues a command to remove the corresponding entry from the central BlockMapMarkersResource, ensuring the world map is kept in sync.

## Internal State & Concurrency
-   **State:** The component holds immutable state after its initial creation. The *name* and *icon* fields are set once and are not intended to be modified during the component's lifetime. The authoritative state for all map markers is maintained externally in the BlockMapMarkersResource.
-   **Thread Safety:** This component is not thread-safe and must not be accessed from multiple threads. All interactions are designed to occur on the main server thread within the ECS update cycle. Modifications and reads are mediated by systems using a CommandBuffer, which guarantees sequential, deterministic execution.

## API Surface
The public API is minimal, reflecting its role as a simple data container.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getName() | String | O(1) | Returns the display name for the map marker. |
| getIcon() | String | O(1) | Returns the asset path or identifier for the marker's icon. |
| getComponentType() | ComponentType | O(1) | **Critical:** Static method to retrieve the unique type identifier for this component. |
| clone() | Component | O(1) | Creates a deep copy of the component's data. |

## Integration Patterns

### Standard Usage
Developers do not interact with BlockMapMarker instances directly. Instead, they add the component to a block entity definition or attach it at runtime via a CommandBuffer. The associated systems handle all subsequent logic automatically.

```java
// Example: Adding a marker to a block entity via a command buffer
// This is a conceptual example of how a system would add the component.

Ref<ChunkStore> blockEntityRef = ... // Reference to a specific block entity
CommandBuffer<ChunkStore> commands = ... // The current command buffer

BlockMapMarker newMarker = new BlockMapMarker("Ancient Tomb", "icons/dungeon_entrance");
commands.addComponent(blockEntityRef, newMarker);

// The OnAddRemove system will now automatically process this addition.
```

### Anti-Patterns (Do NOT do this)
-   **State Modification:** Do not get a BlockMapMarker component from an entity and attempt to change its name or icon. The OnAddRemove system only triggers on component addition and removal, not on state changes. Such modifications will not be reflected on the world map. To update a marker, the old component must be removed and a new one added.
-   **Direct Instantiation:** Creating an instance with `new BlockMapMarker()` without adding it to an entity has no effect on the game world. The instance will be garbage collected without ever being processed by the map system.

## Data Pipeline
The flow of data from block creation to client UI is a clear, one-way pipeline orchestrated by the ECS.

> Flow:
> Block Entity Creation -> **BlockMapMarker** Component Added -> OnAddRemove System -> BlockMapMarkersResource (Aggregate Write) -> WorldMapManager Tick -> MarkerProvider System -> BlockMapMarkersResource (Aggregate Read) -> WorldMapTracker -> Network Packet -> Client World Map UI

