---
description: Architectural reference for Teleport
---

# Teleport

**Package:** com.hypixel.hytale.server.core.modules.entity.teleport
**Type:** Transient

## Definition
```java
// Signature
public class Teleport implements Component<EntityStore> {
```

## Architecture & Concepts
The Teleport class is not a persistent property of an entity; rather, it is an **imperative command component** within the server's Entity-Component-System (ECS) architecture. Its sole purpose is to represent a request to move an entity to a new location, potentially in a different world.

This class acts as a data-driven instruction for a dedicated processing system. When game logic needs to teleport an entity, it constructs a Teleport object, configures it with the target destination and parameters, and attaches it to the entity. A system, likely operating within the main server tick, will detect the presence of this component, execute the physical state change on the entity, and then immediately remove the Teleport component.

This design decouples the *intent* to teleport from the *mechanism* of teleportation, allowing various game systems (portals, commands, scripted events) to trigger teleports without needing to know the low-level details of modifying entity state.

## Lifecycle & Ownership
- **Creation:** A Teleport instance is created on-demand by any system-level logic that needs to initiate a teleport. This is a transient, short-lived object.

- **Scope:** The component is designed to exist for less than a single game tick. It is attached to an entity, processed, and immediately discarded. It should never persist across multiple ticks or be serialized as part of an entity's state.

- **Destruction:** The system responsible for processing teleports is also responsible for removing the component from the entity once the operation is complete. The Java Garbage Collector then reclaims the memory.

## Internal State & Concurrency
- **State:** The Teleport component is a mutable state container. It is instantiated with initial coordinates and can be further configured using its builder-style methods like withHeadRotation and withoutVelocityReset. All state is self-contained and does not reference external mutable objects other than the target World.

- **Thread Safety:** This class is **not thread-safe**. It is designed to be created, attached, and processed exclusively on the main server thread within the scope of a single, synchronous game update cycle. Any concurrent modification will result in race conditions and undefined server behavior.

## API Surface
The public API is designed as a fluent builder to facilitate easy construction and configuration of the teleport command.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the unique type identifier for this component from the EntityModule. |
| withHeadRotation(headRotation) | Teleport | O(1) | Sets an optional, specific rotation for the entity's head. Returns the instance for chaining. |
| withResetRoll() | Teleport | O(1) | Modifies the internal rotation state to set the roll axis to zero. |
| withoutVelocityReset() | Teleport | O(1) | Configures the command to preserve the entity's velocity after teleportation. |
| getWorld() | World | O(1) | Returns the target world for the teleport, or null for an intra-world teleport. |
| getPosition() | Vector3d | O(1) | Returns the target world-space position. |
| getRotation() | Vector3f | O(1) | Returns the target body rotation. |

## Integration Patterns

### Standard Usage
The correct pattern is to instantiate, configure, and attach the component to an entity. The underlying system handles the execution.

```java
// Assume 'entity' is a valid entity and 'targetWorld' is the destination
Vector3d destination = new Vector3d(100.0, 64.0, 250.0);
Vector3f orientation = new Vector3f(0.0f, 90.0f, 0.0f);

// 1. Create the teleport command object
Teleport teleportCommand = new Teleport(targetWorld, destination, orientation)
    .withoutVelocityReset(); // Preserve momentum

// 2. Attach the component to the entity
// The TeleportSystem will process this during the next entity update phase.
entity.set(teleportCommand);
```

### Anti-Patterns (Do NOT do this)
- **Reusing Instances:** Do not create a Teleport object, hold a reference to it, and attach it to multiple entities or re-attach it for a subsequent teleport. Each teleport operation must be represented by a new Teleport instance to prevent state conflicts.

- **Persisting the Component:** Never attempt to save an entity's state while it has a Teleport component attached. This is a transient command, not persistent data. Doing so will lead to unexpected behavior on world load, such as teleport loops.

- **Manual State Management:** Do not get the component, modify its fields, and expect a change. The component is typically processed and removed in the same tick it is added.

## Data Pipeline
The Teleport component initiates a control flow rather than a traditional data processing pipeline. It serves as a message to the entity processing loop.

> Flow:
> Game Logic (e.g. Portal System) -> Instantiates **Teleport** object -> Attaches to Entity -> Entity Update System -> Detects **Teleport** component -> Modifies Entity Transform & World components -> Removes **Teleport** component

