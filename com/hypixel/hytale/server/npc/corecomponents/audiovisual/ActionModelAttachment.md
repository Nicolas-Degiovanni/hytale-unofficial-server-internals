---
description: Architectural reference for ActionModelAttachment
---

# ActionModelAttachment

**Package:** com.hypixel.hytale.server.npc.corecomponents.audiovisual
**Type:** Transient

## Definition
```java
// Signature
public class ActionModelAttachment extends ActionBase {
```

## Architecture & Concepts
The ActionModelAttachment class is a concrete implementation of the ActionBase command pattern, operating within the server-side NPC Behavior System. Its sole responsibility is to dynamically alter the visual composition of an NPC entity by attaching, replacing, or detaching a sub-model to a designated slot on the NPC's primary model.

This class acts as a direct manipulator of an entity's **ModelComponent**. It is a data-driven, single-purpose command that encapsulates the logic for reconstructing the model state. Actions like this are the fundamental building blocks of complex NPC behaviors, allowing designers to script visual changes (e.g., an NPC equipping a weapon, putting on a hat, or showing damage) without writing bespoke code.

Configuration for this action is loaded from asset definitions via its corresponding builder, BuilderActionModelAttachment, making it a key component in the bridge between static game data and live entity state.

### Lifecycle & Ownership
- **Creation:** Instantiated by the server's asset loading and NPC behavior systems when an NPC's definition is parsed. It is constructed from a BuilderActionModelAttachment, which reads configuration from a data file (e.g., a JSON behavior tree). It is **not** created per-execution.
- **Scope:** The object is stateless and reusable. Its lifetime is bound to the lifecycle of the NPC's behavior definition (e.g., its Role). It persists in memory as part of a behavior graph as long as that NPC type is active in the world.
- **Destruction:** The object is eligible for garbage collection when its parent behavior definition is unloaded, typically when a game zone is shut down or a server-wide asset reload occurs.

## Internal State & Concurrency
- **State:** This object is **immutable** after construction. The target *slot* and *attachment* asset names are stored in final fields. It holds no runtime state specific to any single NPC entity instance.
- **Thread Safety:** This class is **not thread-safe** by design. The execute method is expected to be called exclusively from the main server thread as part of the entity update tick. It performs direct, non-atomic mutations on an entity's components via a ComponentAccessor. Concurrent calls on the same entity reference would lead to race conditions and corrupted component state.

## API Surface
The public contract is limited to the execute method inherited from ActionBase.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(ref, role, sensorInfo, dt, store) | boolean | O(1) | Modifies the target entity's ModelComponent. Always returns true, signifying the action is instantaneous and completes in a single tick. |

## Integration Patterns

### Standard Usage
This class is not intended to be invoked directly by most game logic developers. It is automatically executed by an NPC's Role as part of a behavior tree or state machine. The system retrieves the pre-configured action and executes it on the target entity.

```java
// Conceptual engine-level usage within an NPC's Role update loop

// 'action' is a pre-loaded ActionModelAttachment instance
// 'entityRef' is the reference to the NPC being updated
boolean complete = action.execute(entityRef, this, sensorInfo, dt, componentStore);

// Since it always returns true, the behavior tree would immediately
// transition to the next action.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new ActionModelAttachment()`. The object must be configured and created by the asset pipeline via its builder to ensure correctness.
- **Empty Slot Name:** Configuring this action with an empty or null slot name in the source data file will result in a runtime `IllegalArgumentException`. The system enforces that all attachments must have a target slot.
- **Stateful Subclassing:** Do not extend this class to add per-instance state. Actions must remain stateless to be safely reused across multiple NPC instances of the same type.

## Data Pipeline
ActionModelAttachment sits in the middle of the flow from data definition to entity state mutation. It translates a static asset name into a live component modification.

> Flow:
> NPC Definition File (JSON) -> Asset Parser -> **ActionModelAttachment** (Instance created) -> NPC Role Execution -> `execute()` -> ComponentAccessor writes new `ModelComponent` -> Entity state is updated for the next tick.

