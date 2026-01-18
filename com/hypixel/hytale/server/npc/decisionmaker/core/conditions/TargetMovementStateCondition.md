---
description: Architectural reference for TargetMovementStateCondition
---

# TargetMovementStateCondition

**Package:** com.hypixel.hytale.server.npc.decisionmaker.core.conditions
**Type:** Transient Component

## Definition
```java
// Signature
public class TargetMovementStateCondition extends SimpleCondition {
```

## Architecture & Concepts
The TargetMovementStateCondition is a specific, data-driven predicate used within the server-side NPC Decision Maker framework. It functions as a leaf node in a behavior tree, answering a simple boolean question: "Is the entity I am targeting currently in a specific movement state?".

This class acts as a bridge between the abstract decision-making logic and the concrete state of the game world, specifically the NPC movement system. It is not intended for direct manual instantiation by developers. Instead, it is deserialized from NPC behavior definition assets (e.g., JSON files) via its static **CODEC** field. This allows game designers to construct complex AI behaviors declaratively, without writing Java code.

Its core responsibility is to query the **MotionController** of a target entity to determine if its current state matches the configured **MovementState** (e.g., WALKING, IDLE, SPRINTING).

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale **Codec** system during the loading and parsing of NPC behavior assets. The static **CODEC** field defines the deserialization logic, including how to read the desired **MovementState** from the asset file.
- **Scope:** The lifetime of a TargetMovementStateCondition instance is bound to the lifecycle of the parent behavior tree or decision node that contains it. These are typically loaded once and cached for the server's lifetime or until a hot-reload occurs.
- **Destruction:** The object is managed by the Java Garbage Collector. It is eligible for cleanup when the containing behavior asset is unloaded and no longer referenced.

## Internal State & Concurrency
- **State:** The internal state consists of a single field, **movementState**, which is set once during deserialization. After this initial hydration, the object's state is effectively immutable. It holds no runtime-generated data or caches.

- **Thread Safety:** This class is stateless and therefore inherently re-entrant. However, the **evaluate** method interacts with the core Entity Component System (ECS) via the **ArchetypeChunk** and **CommandBuffer**.

    **WARNING:** Invoking the **evaluate** method is only safe from the main server game-loop thread which has exclusive write access to the ECS. Calling it from an asynchronous task or a network thread will lead to race conditions, data corruption, or concurrency exceptions.

## API Surface
The primary interaction point is the overridden **evaluate** method from the **SimpleCondition** base class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| evaluate(...) | boolean | O(1) | Checks if the provided target entity exists and is in the configured MovementState. Returns false if the target is null or invalid. |

## Integration Patterns
This component is designed to be used declaratively within behavior assets, not imperatively in code.

### Standard Usage
A game designer would define this condition within an NPC's behavior file. The system then loads and executes it.

**Conceptual Asset Definition (e.g., JSON):**
```json
{
  "type": "TargetMovementStateCondition",
  "state": "WALKING"
}
```

**System-Level Invocation (Hypothetical):**
```java
// This code is representative of how the Decision Maker engine would use the object.
// You would not typically write this yourself.
SimpleCondition condition = loadedBehavior.getCondition(); // Deserialized instance

boolean result = condition.evaluate(
    npcEntityIndex,
    worldChunk,
    targetRef,
    commandBuffer,
    evaluationContext
);

if (result) {
    // Target is walking, proceed with action...
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use **new TargetMovementStateCondition()**. This bypasses the **CODEC** and results in an unconfigured object where **movementState** is null, causing NullPointerExceptions during evaluation.

- **State Mutation:** Do not attempt to use reflection or other means to change the **movementState** field after creation. These objects are expected to be immutable.

- **Off-Thread Execution:** Do not call the **evaluate** method from any thread other than the main server thread. This will violate the threading constraints of the ECS and cause severe, unpredictable bugs.

## Data Pipeline
The primary flow for this component is from a static data definition into a runtime evaluation that queries live game state.

> Flow:
> NPC Behavior Asset File -> **CODEC** Deserializer -> **TargetMovementStateCondition Instance** -> Decision Maker Engine -> evaluate() -> MotionController Query -> Boolean Result

