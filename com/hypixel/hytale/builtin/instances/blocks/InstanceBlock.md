---
description: Architectural reference for InstanceBlock
---

# InstanceBlock

**Package:** com.hypixel.hytale.builtin.instances.blocks
**Type:** Data Component

## Definition
```java
// Signature
public class InstanceBlock implements Component<ChunkStore> {
```

## Architecture & Concepts
The InstanceBlock component is the data-driven core of Hytale's world instancing system. It serves as a persistent link between a physical block entity in a primary world and an entirely separate, dynamically managed world, often referred to as an "instance" or "dungeon".

This component does not contain any active logic itself. Instead, it acts as a data container, holding the unique identifier (worldUUID) of the target world. The critical lifecycle logic, particularly the cleanup of the instanced world, is managed by the associated **InstanceBlock.OnRemove** system. This separation of data (the component) from behavior (the system) is a fundamental pattern in Hytale's Entity-Component-System (ECS) architecture.

When attached to a block entity, this component effectively turns that block into a "portal" or "gateway". The engine and related systems can then query for this component to manage player transitions and instance loading.

### Lifecycle & Ownership
The lifecycle of an InstanceBlock is inextricably tied to the entity it is attached to. Understanding this relationship is critical to prevent orphaned worlds or premature instance deletion.

-   **Creation:** An InstanceBlock is created and attached to a block entity when an instanced area is generated or placed by a designer. This is typically done via a `CommandBuffer` as part of a larger world-generation or event-handling process.
-   **Scope:** The component persists for the entire lifetime of its host block entity. It is serialized as part of the chunk data and reloaded when the chunk is loaded.
-   **Destruction:** The component is destroyed when its host entity is removed from the world (e.g., the block is mined by a player). This event is the primary trigger for the **InstanceBlock.OnRemove** system, which will then initiate the cleanup of the associated world if the `closeOnRemove` flag is set.

## Internal State & Concurrency
-   **State:** The component's state is mutable. It primarily consists of the `worldUUID` which identifies the target world, and the `closeOnRemove` flag which dictates cleanup behavior. It also contains a transient `worldFuture` field, which is used at runtime for asynchronous loading operations and is not persisted to storage.
-   **Thread Safety:** This component is **not thread-safe**. As with all ECS components, it must only be accessed and modified from the main world thread. All mutations should be queued through a `CommandBuffer` to ensure deterministic execution and prevent race conditions. The use of `CompletableFuture` for `worldFuture` is managed internally by engine systems to bridge asynchronous world loading with the synchronous game loop.

## API Surface
The public API is minimal, focusing on data access. The primary interaction with this component is through the ECS query and command systems, not direct method calls.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the unique type identifier for this component, used for ECS queries. |
| getWorldUUID() | UUID | O(1) | Returns the unique identifier of the instanced world this block links to. |
| getWorldFuture() | CompletableFuture | O(1) | Returns a future that completes with the World object. Used for async loading. |
| isCloseOnRemove() | boolean | O(1) | Determines if the linked world should be destroyed when this block is removed. |

## Integration Patterns

### Standard Usage
The correct way to create an instance link is to add an InstanceBlock component to a new or existing block entity using a `CommandBuffer`.

```java
// Creating a new instanced world and linking it to a block entity
UUID newWorldId = createNewInstancedWorld(); // Hypothetical API call
CommandBuffer<ChunkStore> commands = ...;
Ref<ChunkStore> blockEntityRef = ...;

InstanceBlock instanceLink = new InstanceBlock(newWorldId, true);
commands.addComponent(blockEntityRef, instanceLink);
```

### Anti-Patterns (Do NOT do this)
-   **State Modification Outside CommandBuffer:** Modifying an InstanceBlock's state directly from an asynchronous callback or a different thread will lead to state corruption and unpredictable behavior. All changes must be funneled through the world's `CommandBuffer`.
-   **Mismanaging closeOnRemove:** Setting `closeOnRemove` to true for instances that should persist beyond the lifecycle of their entry block (e.g., player housing) will result in data loss. Conversely, setting it to false for temporary dungeons will lead to orphaned worlds and memory leaks.
-   **Blocking on worldFuture:** Calling `worldFuture.get()` on the main server thread will block all world processing and cause severe performance degradation. Use `thenAccept` or other asynchronous patterns to handle the completed future.

## Data Pipeline
The most critical data flow involving this component is the destruction pipeline, which is managed by the `InstanceBlock.OnRemove` system.

> Flow:
> Block Entity Removal Event -> ECS Engine dispatches to **InstanceBlock.OnRemove** system -> System reads `worldUUID` from component -> System calls `InstancesPlugin.safeRemoveInstance(worldUUID)` -> Instance Manager queues world for unloading and deletion.

