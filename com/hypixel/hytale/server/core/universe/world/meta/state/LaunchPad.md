---
description: Architectural reference for LaunchPad
---

# LaunchPad

**Package:** com.hypixel.hytale.server.core.universe.world.meta.state
**Type:** Component (Data)

## Definition
```java
// Signature
public class LaunchPad implements Component<ChunkStore> {
```

## Architecture & Concepts
The LaunchPad class is a server-side data component that encapsulates the state of a launch pad block within the world. As a component in Hytale's Entity Component System (ECS) architecture, its sole responsibility is to hold data; it contains no game logic for applying forces. The actual physics simulation is handled by a separate *System* that queries for entities possessing this component.

This component is intrinsically linked to the world's chunk-based storage system, as indicated by its implementation of Component<ChunkStore>. This signifies that LaunchPad instances are owned and managed by a ChunkStore, persisting as part of a chunk's data.

Serialization and deserialization are managed through the static CODEC field, a BuilderCodec instance. This mechanism is fundamental to the engine's world-saving and loading process, allowing LaunchPad properties to be written to and read from disk.

A critical feature is the nested LaunchPadSettingsPage class. This inner class provides the server-side logic for a player-facing UI, allowing in-game modification of a LaunchPad's properties. This pattern tightly couples the component's data model with its configuration interface, creating a self-contained, editable block entity.

## Lifecycle & Ownership
- **Creation:** A LaunchPad instance is created under two primary conditions:
    1.  **Deserialization:** When a WorldChunk is loaded from disk, the chunk's data is decoded, and the CODEC is used to instantiate LaunchPad components for any corresponding blocks. This is the most common creation path.
    2.  **Programmatic:** A new LaunchPad can be instantiated and attached to a block entity at runtime, for example, when a player places a new launch pad block in the world.

- **Scope:** The component's lifetime is strictly bound to the block entity it is attached to. It persists as long as the block exists in a loaded chunk.

- **Destruction:** The component is marked for garbage collection when the associated block is destroyed by a player or when the WorldChunk containing it is unloaded from server memory.

## Internal State & Concurrency
- **State:** The internal state of LaunchPad is **mutable**. It consists of three float values for velocity and a boolean flag. This state is designed to be modified at runtime, primarily through the integrated LaunchPadSettingsPage UI.

- **Thread Safety:** This class is **not thread-safe**. It contains no internal locking or synchronization mechanisms. All modifications and reads of its state must be performed on the main server game loop thread. Accessing a LaunchPad instance from an asynchronous task or worker thread without external synchronization will lead to race conditions, data corruption, and server instability. The engine's ECS framework ensures this safety by design, processing components within a single-threaded tick loop.

## API Surface
The public API is minimal, focusing on data access. Logic is intentionally excluded.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | ComponentType | O(1) | Statically retrieves the unique type identifier for this component from the BlockModule. |
| getVelocityX() | float | O(1) | Returns the configured velocity on the X-axis. |
| getVelocityY() | float | O(1) | Returns the configured velocity on the Y-axis. |
| getVelocityZ() | float | O(1) | Returns the configured velocity on the Z-axis. |
| isPlayersOnly() | boolean | O(1) | Returns true if the launch pad should only affect player entities. |

## Integration Patterns

### Standard Usage
A physics or collision system retrieves the LaunchPad component from a block entity during a collision event to apply the configured velocity to the colliding entity.

```java
// Conceptual example within a server-side System
void onEntityCollisionWithBlock(Entity collidingEntity, Ref<ChunkStore> blockRef, ChunkStore store) {
    LaunchPad launchPad = store.getComponent(blockRef, LaunchPad.getComponentType());
    if (launchPad != null) {
        // Do not apply force if playersOnly is true and entity is not a player
        if (launchPad.isPlayersOnly() && !isPlayer(collidingEntity)) {
            return;
        }

        // Apply velocity to the colliding entity's physics component
        PhysicsComponent physics = getPhysicsComponent(collidingEntity);
        physics.addVelocity(launchPad.getVelocityX(), launchPad.getVelocityY(), launchPad.getVelocityZ());
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Asynchronous Modification:** Never modify LaunchPad fields from a separate thread. All state changes must be queued and executed on the main server thread to prevent data corruption.

- **State Modification without Persistence:** When modifying a LaunchPad programmatically, you must also retrieve its parent WorldChunk component and call markNeedsSaving(). Failure to do so will result in the changes being lost when the chunk is unloaded or the server restarts. The built-in LaunchPadSettingsPage handles this correctly.

- **Detached Instantiation:** Avoid creating instances with `new LaunchPad()` that are not immediately attached to a block entity via a Store. A disconnected component serves no purpose and will be garbage collected without ever affecting the game state.

## Data Pipeline
The flow of data for configuring a LaunchPad via its in-game UI is a round-trip from client to server and back to storage.

> Flow:
> Player interacts with block -> Server creates **LaunchPadSettingsPage** -> UI definition sent to client -> Player changes values and clicks save -> Client sends **LaunchPadSettingsPageEventData** to server -> `handleDataEvent` updates **LaunchPad** state -> `WorldChunk.markNeedsSaving()` is called -> ChunkStore serializes the updated component to disk.

