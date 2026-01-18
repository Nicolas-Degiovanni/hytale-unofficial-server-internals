---
description: Architectural reference for Teleporter
---

# Teleporter

**Package:** com.hypixel.hytale.builtin.adventure.teleporter.component
**Type:** Transient

## Definition
```java
// Signature
public class Teleporter implements Component<ChunkStore> {
```

## Architecture & Concepts
The Teleporter is a data-only component within Hytale's Entity Component System. It does not execute teleportation logic itself; its sole responsibility is to define the parameters of a potential destination. It serves as a configuration object that is attached to game entities, typically block entities, to make them function as teleporters.

This component defines a destination in one of two primary ways:
1.  **Symbolic Reference:** By storing the name of a globally registered Warp. This decouples the teleporter block from a hardcoded location, allowing administrators to update warp points centrally.
2.  **Absolute or Relative Coordinates:** By storing a specific Transform (position and rotation) and an optional world UUID. A byte mask, *relativeMask*, can be used to make parts of the destination Transform relative to the teleporting entity's current state.

The class is fundamentally a serialization contract, defined by its static CODEC field. This allows Teleporter instances to be persisted within world data as part of a block entity's state and rehydrated when a chunk is loaded.

## Lifecycle & Ownership
- **Creation:** A Teleporter instance is almost exclusively created by the engine's deserialization systems when loading world data. When a chunk containing a block entity with a Teleporter component is loaded, the static CODEC is used to instantiate and populate the object from the stored data. Manual instantiation is rare and reserved for in-memory programmatic creation of teleporters.
- **Scope:** The lifetime of a Teleporter is strictly bound to its parent component container, such as a block entity. It persists in memory only as long as the parent entity is active in the world.
- **Destruction:** The object is eligible for garbage collection as soon as its parent entity is unloaded or destroyed. It manages no external resources and has no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** The Teleporter is a highly mutable Plain Old Java Object (POJO). Its fields represent the raw configuration data for a teleport destination and can be freely modified after instantiation. It holds no derived or cached state.
- **Thread Safety:** **This class is not thread-safe.** It is designed to be owned and manipulated exclusively by the main server thread responsible for the world in which its parent entity resides. Any off-thread access or modification will result in undefined behavior, including data corruption and race conditions. All interactions must be synchronized with the main game loop.

## API Surface
The public API is focused on configuration and the final resolution of the destination into an actionable command object.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toTeleport(currentPosition, currentRotation, blockPosition) | Teleport | O(1) | **Core Method.** Resolves the component's data into a concrete Teleport object. Throws no exceptions but may return null if the destination is invalid or unresolvable. |
| isValid() | boolean | O(1) | Performs a validation check to determine if the configured destination points to a valid, existing warp or world. |
| getComponentType() | ComponentType | O(1) | Static accessor for retrieving the component's unique type identifier within the engine's component registry. |
| clone() | Component | O(1) | Creates a deep copy of the Teleporter component's data. |

## Integration Patterns

### Standard Usage
The standard pattern involves a higher-level system retrieving this component from an entity (e.g., a block a player stands on) and using it to generate a Teleport command for the core teleportation service.

```java
// A system reacting to a player interacting with a teleporter block
Entity blockEntity = world.getEntityAt(blockPosition);
Teleporter teleporterData = blockEntity.getComponent(Teleporter.getComponentType());

if (teleporterData != null && teleporterData.isValid()) {
    // Resolve the destination based on the player's current state
    Teleport teleportCommand = teleporterData.toTeleport(
        player.getPosition(),
        player.getRotation(),
        blockPosition
    );

    if (teleportCommand != null) {
        // Submit the command to the core teleportation system
        TeleportPlugin.get().teleportEntity(player, teleportCommand);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation for Configuration:** Avoid using `new Teleporter()` to configure and pass around teleportation data. The correct pattern is to use the dedicated Teleport object for commands and the Teleporter component for persistent entity state.
- **Caching Resolved Teleport:** Do not call `toTeleport` once and cache the result. If the destination is relative, or if it references a named Warp that is later modified, the cached Teleport object will become stale. Always resolve the destination just-in-time.
- **Modification from Off-Thread:** Never modify a Teleporter component from a thread other than the main server thread for its world. This will break thread-safety guarantees of the game engine.

## Data Pipeline
The Teleporter component acts as the originating source of configuration data in the teleportation process. It does not process data itself but provides the input for the core teleportation systems.

> Flow:
> World Data on Disk -> Engine Deserializer -> **Teleporter** (in-memory component) -> Game Logic calls `toTeleport()` -> Teleport Command Object -> Core Teleport Service -> Entity State Update

