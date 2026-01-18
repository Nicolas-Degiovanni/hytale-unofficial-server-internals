---
description: Architectural reference for SensorBlockChange
---

# SensorBlockChange

**Package:** com.hypixel.hytale.server.npc.corecomponents.world
**Type:** Transient Component

## Definition
```java
// Signature
public class SensorBlockChange extends SensorEvent {
```

## Architecture & Concepts

The SensorBlockChange is a specialized component within the server-side NPC artificial intelligence framework. It functions as a reactive sensor that allows an NPC to detect and respond to block modification events—such as breaking or placing blocks—initiated by players or other NPCs within a configurable radius.

Architecturally, this class is not a proactive world-scanner. It does not poll world state or perform expensive region queries. Instead, it operates as a passive listener within a message-driven system. It subscribes to a specific type of block event by registering for a pre-calculated *message slot* on its parent NPC's event support components.

This design decouples the AI sensory logic from the core world simulation. The world simulation is responsible for identifying a block change and dispatching a targeted message to the relevant event support components on nearby entities. The SensorBlockChange then acts as the final filter and consumer of this message during the NPC's AI update cycle, translating a low-level world event into a high-level AI stimulus.

## Lifecycle & Ownership

-   **Creation:** An instance of SensorBlockChange is never created directly via its constructor in game logic. It is instantiated by the server's asset loading pipeline, specifically through its corresponding builder, BuilderSensorBlockChange. This occurs when an NPC's behavior definition is parsed and its AI components are assembled.
-   **Scope:** The lifecycle of a SensorBlockChange instance is tightly bound to the lifecycle of the NPC entity to which it belongs. It persists as long as the parent NPC is active in the world.
-   **Destruction:** The object is marked for garbage collection when the parent NPC entity is destroyed or unloaded from the world. There is no manual destruction method.

## Internal State & Concurrency

-   **State:** This class is stateful. It holds configuration data loaded during its construction, including the detection range, search type, and integer identifiers for the player and NPC message slots. This state is considered immutable after initialization.
-   **Thread Safety:** This class is **not thread-safe**. Its methods are designed to be executed exclusively on the main server thread as part of the synchronous NPC update tick. The methods read from and modify ECS components (PlayerBlockEventSupport, NPCBlockEventSupport) which are not designed for concurrent access.

    **Warning:** Accessing instances of this class from any thread other than the main server thread will lead to race conditions, data corruption, or ConcurrentModificationExceptions within the entity-component store.

## API Surface

The primary interaction with this class is through methods inherited from SensorEvent, which in turn call the protected methods below.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getPlayerTarget(parent, store) | Ref<EntityStore> | O(1) | Checks the parent's PlayerBlockEventSupport component for a pending message in the configured slot. If a spatially valid message exists, it is consumed and the originating entity is returned. Returns null if no event is found. |
| getNpcTarget(parent, store) | Ref<EntityStore> | O(1) | Checks the parent's NPCBlockEventSupport component for a pending message. If a valid message exists, it is consumed and the originating entity is returned. Returns null if no event is found. |

## Integration Patterns

### Standard Usage

This component is not intended for direct invocation by developers. It is configured declaratively within an NPC's asset files and integrated into a behavior tree. A controlling node, such as a Decorator or Selector, would query this sensor on each AI tick to decide whether to alter the NPC's behavior.

```java
// Conceptual usage within a hypothetical Behavior Tree node
// This code would exist inside the AI framework, not user code.

// The sensor is a member variable of the behavior node
SensorBlockChange blockSensor; 

// During the AI tick:
Ref<EntityStore> target = blockSensor.update(parentNPC, entityStore);
if (target != null) {
    // A block event was detected.
    // Transition to a new behavior, e.g., "Investigate" or "Attack".
    blackboard.put("targetEntity", target);
    return BehaviorState.SUCCESS;
}

return BehaviorState.FAILURE;
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never use `new SensorBlockChange()`. The internal message slots will not be initialized correctly, rendering the sensor non-functional. It must be created via the asset loading system and its associated builder.
-   **State Mutation:** Do not attempt to modify the sensor's range or message slots after it has been constructed. This will cause a desynchronization with the underlying event messaging system, leading to missed or incorrect event detections.
-   **Asynchronous Polling:** Do not call `getPlayerTarget` or `getNpcTarget` from a separate worker thread to "check for events". All interactions must be synchronized with the main server tick to prevent catastrophic state corruption in the ECS.

## Data Pipeline

The flow of data from world event to AI reaction is a multi-stage process mediated by the ECS messaging system.

> Flow:
> 1. Player/NPC breaks a block.
> 2. World Server detects the block change and identifies affected entities in the vicinity.
> 3. A targeted message is dispatched to the `PlayerBlockEventSupport` or `NPCBlockEventSupport` component of the nearby NPC.
> 4. On its next AI update, the NPC's behavior tree ticks.
> 5. A node queries the **SensorBlockChange** instance.
> 6. The sensor polls the appropriate event support component, consuming the message.
> 7. The sensor returns a valid entity reference to the behavior tree.
> 8. The behavior tree triggers a state change in the NPC.

