---
description: Architectural reference for PortalInvalidDestinationSystem
---

# PortalInvalidDestinationSystem

**Package:** com.hypixel.hytale.builtin.portals.systems
**Type:** System Service

## Definition
```java
// Signature
public class PortalInvalidDestinationSystem extends RefSystem<ChunkStore> {
```

## Architecture & Concepts
The **PortalInvalidDestinationSystem** is a reactive, server-side system responsible for maintaining the integrity of portal block states. It acts as a data validation and cleanup mechanism within the Entity Component System (ECS) framework. Its primary function is to automatically detect and deactivate portal blocks whose destination worlds are no longer valid or have been deleted.

This system operates at the intersection of world state management and the portal gameplay feature. It ensures that players do not encounter portals leading to unloaded or non-existent worlds, preventing potential errors and improving data consistency. It achieves this by listening to entity lifecycle events and providing a bulk operation utility for other systems to invoke during world management tasks.

The system is designed to be stateless and operates based on a specific query for entities possessing both a **PortalDevice** and a **BlockStateInfo** component.

## Lifecycle & Ownership
- **Creation:** An instance of **PortalInvalidDestinationSystem** is created by the core ECS framework when a world is initialized. It is automatically discovered and registered as part of the server's system startup sequence. Developers do not manually instantiate this class.
- **Scope:** The system's lifecycle is bound to the **World** instance it operates on. It remains active as long as the world is loaded and running on the server.
- **Destruction:** The system is destroyed and garbage collected when its parent world is unloaded or the server shuts down.

## Internal State & Concurrency
- **State:** This system is entirely **stateless**. It does not contain any member variables that store data between invocations. All necessary information is retrieved from components passed into its methods via the **Store** and **CommandBuffer**. This design makes the system highly predictable and robust.

- **Thread Safety:** This system is thread-safe.
    - The static method **turnOffPortalsInWorld** utilizes a parallel entity query (**forEachEntityParallel**), enabling high-performance iteration over large numbers of entities.
    - **CRITICAL:** All operations that mutate world state, such as changing a block, are not executed directly. Instead, they are scheduled for execution on the target world's dedicated thread via the **world.execute** method. This pattern is essential for preventing race conditions and maintaining the integrity of world data, which is not designed for concurrent modification.

## API Surface
The primary public contract is a static utility method. Framework-overridden methods like **onEntityAdded** are not intended for direct invocation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| turnOffPortalsInWorld(originWorld, destinationWorld) | static void | O(N) | Iterates all portal entities in the *originWorld*. Any portal pointing to the *destinationWorld* will be turned off. N is the number of entities with a **PortalDevice** component. |

## Integration Patterns

### Standard Usage
The most common use case is invoking the static utility method from another system or command handler, typically during world deletion or unloading procedures. This ensures all links to the removed world are severed.

```java
// Example: In a world deletion command handler
World worldToRemove = server.getWorldManager().getWorldByName("old_world");
World mainWorld = server.getWorldManager().getWorldByName("main");

// Before deleting worldToRemove, turn off all portals in the main world that point to it.
PortalInvalidDestinationSystem.turnOffPortalsInWorld(mainWorld, worldToRemove);

// Proceed with world deletion...
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new PortalInvalidDestinationSystem()`. The ECS framework manages its lifecycle. Attempting to create it manually will result in a non-functional object that is not registered with the engine.
- **Bypassing the World Executor:** Do not attempt to replicate the logic of the private **turnOffPortalBlock** method. Directly modifying a **WorldChunk** from an arbitrary thread will lead to data corruption, server instability, and crashes.

## Data Pipeline
This system has two primary data flows: a reactive flow triggered by entity loading and a proactive flow triggered by explicit calls.

> **Reactive Flow (On Chunk Load):**
> Chunk Load → ECS adds Entity with **PortalDevice** → **onEntityAdded** hook triggered → **PortalInvalidDestinationSystem** checks `destinationWorld` → If `null`, schedules task → World Executor → **turnOffPortalBlock** → **WorldChunk** state is modified.

> **Proactive Flow (On World Deletion):**
> World Management Event → System calls **turnOffPortalsInWorld** → **PortalInvalidDestinationSystem** iterates all portal entities in parallel → For each matching portal, schedules task → World Executor → **turnOffPortalBlock** → **WorldChunk** state is modified.

