---
description: Architectural reference for PlacedByInteractionComponent
---

# PlacedByInteractionComponent

**Package:** com.hypixel.hytale.server.core.modules.interaction.components
**Type:** Transient

## Definition
```java
// Signature
public class PlacedByInteractionComponent implements Component<ChunkStore> {
```

## Architecture & Concepts
The PlacedByInteractionComponent is a server-side data component within the engine's Entity-Component-System (ECS) architecture. Its sole purpose is to act as a data tag, permanently associating a world object (such as a block or a placed entity) with the unique identifier of the player who created or placed it.

As an implementation of Component<ChunkStore>, this component is designed to be persisted within the world's chunk data. This signifies its role in storing state that must survive server restarts and be loaded with the world.

The static CODEC field is a critical piece of its design. It leverages the engine's serialization framework (BuilderCodec) to define how the component's state—specifically the player's UUID—is written to and read from persistent storage. This integration is automatic and managed by the ChunkStore and world persistence systems. The component's type is registered and retrieved via the InteractionModule, ensuring a decoupled and modular design where game systems can request components by type without direct class dependencies.

## Lifecycle & Ownership
- **Creation:** An instance is created by a server-side game system, typically the InteractionModule, at the moment a player successfully places an object in the world. The constructor is invoked with the placing player's UUID.
- **Scope:** The component's lifetime is strictly bound to the lifetime of the host object it is attached to. It persists as long as the block or entity exists in the world.
- **Destruction:** The component is marked for garbage collection when its host object is destroyed (e.g., a block is broken) or the component is explicitly removed by a game system. There is no manual destruction method; its memory is managed by the JVM.

## Internal State & Concurrency
- **State:** The component holds a single, mutable state field: whoPlacedUuid. It is fundamentally a simple data container. While technically mutable, this field is intended to be treated as immutable after its initial construction.
- **Thread Safety:** This component is **not thread-safe**. It is a plain Java object with no internal locking mechanisms. All access and modification must be synchronized externally, typically by being confined to the main server thread or a specific world-processing thread that owns the corresponding game region. Unmanaged multi-threaded access will lead to data corruption and server instability.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the globally registered type for this component from the InteractionModule. |
| PlacedByInteractionComponent(UUID) | constructor | O(1) | Constructs a new component, initializing it with the placer's UUID. |
| getWhoPlacedUuid() | UUID | O(1) | Returns the UUID of the player who placed the associated object. |
| clone() | Component | O(1) | Creates a shallow copy of the component. Used by engine systems for duplication. |

## Integration Patterns

### Standard Usage
This component is attached to an entity or block to tag it with its creator. Game logic can later query for this component to implement ownership-based mechanics, such as protection or logging.

```java
// A server system processing a block placement event
UUID placingPlayerUuid = event.getPlayer().getUuid();
Block targetBlock = event.getWorld().getBlockAt(event.getPosition());

// Create and attach the component to the block
PlacedByInteractionComponent ownership = new PlacedByInteractionComponent(placingPlayerUuid);
targetBlock.addComponent(ownership);
```

### Anti-Patterns (Do NOT do this)
- **State Modification:** Do not modify the UUID after the component has been attached to an object. This can break ownership logic and lead to a desynchronized game state. The component should be treated as write-once.
- **Client-Side Logic:** This is a server-authoritative component. Client code should never create, modify, or rely on this component for gameplay logic, as it has no presence on the client.
- **Manual Serialization:** Never attempt to serialize or deserialize this component manually. The engine's world persistence layer handles this automatically via the defined CODEC. Bypassing this will lead to data corruption on world saves.

## Data Pipeline
This component is primarily data-at-rest, but it follows a clear pipeline during world persistence operations.

> **Flow (World Save):**
> Game Event (Player places block) -> **PlacedByInteractionComponent** (Instantiation & Attachment) -> ChunkStore (In-memory storage) -> World Save Trigger -> **CODEC** (Serialization to binary) -> Disk Storage

> **Flow (World Load):**
> Disk Storage -> World Load Trigger -> **CODEC** (Deserialization from binary) -> **PlacedByInteractionComponent** (Re-instantiation) -> ChunkStore (Restored in-memory) -> Game Logic (Component is now available for queries)

