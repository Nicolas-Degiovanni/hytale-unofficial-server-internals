---
description: Architectural reference for TrackedPlacement
---

# TrackedPlacement

**Package:** com.hypixel.hytale.server.core.modules.interaction.blocktrack
**Type:** Data Component

## Definition
```java
// Signature
public class TrackedPlacement implements Component<ChunkStore> {
```

## Architecture & Concepts
The **TrackedPlacement** component is a data-only "tag" within the server's Entity-Component-System (ECS) architecture. Its sole purpose is to mark a block entity for inclusion in a global counting mechanism. It does not contain any logic itself; instead, it serves as a trigger for the nested **OnAddRemove** system.

This component is central to the block tracking feature, which allows the server to maintain a real-time count of specific block types in the world. When a **TrackedPlacement** component is attached to a block entity, the **OnAddRemove** system is invoked by the ECS engine. The system then inspects the actual block in the world, identifies its type, and increments a global **BlockCounter** resource. The reverse occurs upon the component's removal.

This design cleanly separates state from logic. **TrackedPlacement** holds the state (the name of the block being tracked), while **OnAddRemove** contains the logic for how to react to changes in that state's lifecycle. It is a fundamental building block for game mechanics that might limit the number of certain blocks a player can place or that track resources for a faction.

### Lifecycle & Ownership
- **Creation:** Instances are not created directly via the constructor. They are attached to block entities programmatically through a **CommandBuffer** by other game systems, typically in response to a player action or world generation event. The component is added first, and its internal *blockName* field is populated subsequently by the **OnAddRemove** system during the same tick.
- **Scope:** The component's lifetime is strictly tied to the entity it is attached to. It persists as long as the block entity exists in the world.
- **Destruction:** The component is destroyed when its parent entity is removed from the **ChunkStore**. This action automatically triggers the **onEntityRemove** callback in the **OnAddRemove** system, ensuring the global block count is correctly decremented before the component is garbage collected.

## Internal State & Concurrency
- **State:** The component's state is mutable. It contains a single field, *blockName*, which is null upon initial attachment and is populated by the **OnAddRemove** system. This field is critical for the removal logic, as it allows the system to know which counter to decrement without re-querying the world state, which may no longer be valid.
- **Thread Safety:** This component is **not thread-safe** and must not be accessed or modified from outside the main server thread that processes the ECS update loop. All interactions must be mediated through a **CommandBuffer** to guarantee transactional integrity and prevent race conditions with the ECS engine.

## API Surface
The public API is minimal, as interaction is intended to occur through the ECS engine rather than direct method calls.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the unique type identifier for this component, used to register and query it within the ECS. |
| clone() | Component | O(1) | Creates a shallow copy of the component. Used internally by the ECS for operations like entity duplication. |

## Integration Patterns

### Standard Usage
Developers should never instantiate or manage this component directly. The correct pattern is to add the component type to an entity using a **CommandBuffer** within a system. The **OnAddRemove** system will handle the rest of the logic.

```java
// Correct: Add the component to a block entity reference within a system.
// The engine will instantiate it and trigger the OnAddRemove logic.

Ref<ChunkStore> blockEntityRef = ...;
CommandBuffer<ChunkStore> commands = ...;

commands.addComponent(blockEntityRef, TrackedPlacement.getComponentType());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new TrackedPlacement()`. The ECS engine manages the component's lifecycle. Direct creation bypasses critical engine hooks and will lead to a non-functional component.
- **Manual State Management:** Do not manually set the *blockName* field. This field is owned and managed by the **OnAddRemove** system. Manually altering it can desynchronize the global **BlockCounter** and cause severe state corruption.
- **External Modification:** Do not get a reference to this component and modify it from an external thread. All interactions must be queued via a **CommandBuffer**.

## Data Pipeline
The data flow for this component is reactive, triggered by its addition to or removal from an entity.

> **Addition Flow:**
> Game Logic System -> CommandBuffer.addComponent -> ECS Engine attaches **TrackedPlacement** -> **OnAddRemove.onEntityAdded** -> Reads BlockStateInfo & BlockChunk -> Updates global BlockCounter resource

> **Removal Flow:**
> Game Logic System -> CommandBuffer.removeComponent -> ECS Engine detaches **TrackedPlacement** -> **OnAddRemove.onEntityRemove** -> Reads *blockName* from component -> Updates global BlockCounter resource

