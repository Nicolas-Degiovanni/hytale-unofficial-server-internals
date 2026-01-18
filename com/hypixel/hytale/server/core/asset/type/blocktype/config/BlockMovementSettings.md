---
description: Architectural reference for BlockMovementSettings
---

# BlockMovementSettings

**Package:** com.hypixel.hytale.server.core.asset.type.blocktype.config
**Type:** Data Object

## Definition
```java
// Signature
public class BlockMovementSettings implements NetworkSerializable<com.hypixel.hytale.protocol.BlockMovementSettings> {
```

## Architecture & Concepts
BlockMovementSettings is a data-driven configuration object that defines the physical interaction properties for a specific block type. It is not a service or manager, but rather a passive data container that dictates how entities are affected when moving on or through a block. For example, it specifies if a block is climbable like a ladder, bouncy like a slime block, or slippery like ice.

The core of this class is its static CODEC field. This `BuilderCodec` is the primary mechanism for its instantiation, enabling the system to deserialize block movement properties directly from asset configuration files (e.g., JSON). This design decouples physics behavior from hard-coded logic, allowing designers and creators to define block interactions declaratively.

By implementing the NetworkSerializable interface, this class guarantees that server-side physics configurations can be reliably transmitted to the client. The `toPacket` method facilitates this by converting the server-side data object into a network-ready protocol message, ensuring simulation consistency between the server and all connected clients.

## Lifecycle & Ownership
- **Creation:** Instances are created by the Hytale asset loading framework during server initialization or asset hot-reloading. The static CODEC is invoked to deserialize a section of a block's definition file into a BlockMovementSettings object. It is never instantiated directly via its constructor in game logic.
- **Scope:** The object's lifetime is bound to the parent BlockType asset it configures. It is loaded into memory and persists as long as its corresponding BlockType is registered in the server's asset registry.
- **Destruction:** The object is marked for garbage collection when the asset registry is cleared or reloaded, typically during a server shutdown or a major content update.

## Internal State & Concurrency
- **State:** The object is effectively immutable. After being deserialized from a configuration file, its state is not intended to change. All public fields are exposed via getters only, enforcing a read-only pattern at runtime. This object serves as a canonical source of truth for a block's movement properties.
- **Thread Safety:** Inherently thread-safe for all read operations. As an immutable data container, it can be safely accessed by multiple threads simultaneously without locks or other synchronization primitives. This is critical for performance in the multithreaded server environment, where different systems (e.g., physics, AI) may need to query block properties concurrently.

## API Surface
The public API is composed exclusively of accessor methods to read the configured physics properties.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toPacket() | com.hypixel.hytale.protocol.BlockMovementSettings | O(1) | Converts the object into its network protocol equivalent for client synchronization. |
| isClimbable() | boolean | O(1) | Returns true if the block allows entity climbing. |
| isBouncy() | boolean | O(1) | Returns true if the block imparts an upward force on entities. |
| getDrag() | float | O(1) | Returns the movement drag coefficient for an entity on this block. |
| getFriction() | float | O(1) | Returns the friction coefficient for an entity on this block. |

## Integration Patterns

### Standard Usage
This object is not meant to be retrieved directly. It is accessed as a property of a higher-level BlockType asset. Game logic queries the world for a block's type, then retrieves its movement settings to inform the physics simulation.

```java
// Hypothetical usage within the physics engine
BlockType blockUnderPlayer = world.getBlockTypeAt(player.getPosition().down());
BlockMovementSettings settings = blockUnderPlayer.getMovementSettings();

if (settings.isBouncy()) {
    player.applyVerticalForce(settings.getBounceVelocity());
}

// Apply friction and drag based on settings
player.getPhysicsState().setFriction(settings.getFriction());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new BlockMovementSettings()`. All instances must be defined in and loaded from asset files via the engine's asset pipeline. Manual creation bypasses the data-driven design and can lead to desynchronization between the server and game content.
- **State Modification:** Do not use reflection or other unsupported mechanisms to alter the fields of a BlockMovementSettings object at runtime. The engine expects this data to be constant and consistent throughout its lifecycle.

## Data Pipeline
BlockMovementSettings is a data source, not a data processor. Its primary role is to inject static configuration data into the runtime.

> **Asset Loading Flow:**
> Block Definition File (JSON) -> Server Asset Loader -> **BuilderCodec** -> **BlockMovementSettings Instance** -> Caching within parent BlockType asset

> **Network Synchronization Flow:**
> Server-side **BlockMovementSettings Instance** -> `toPacket()` -> Network Protocol Message -> Network Layer -> Client -> Client-side Physics Engine

