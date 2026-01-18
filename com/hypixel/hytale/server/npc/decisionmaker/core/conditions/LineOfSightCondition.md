---
description: Architectural reference for LineOfSightCondition
---

# LineOfSightCondition

**Package:** com.hypixel.hytale.server.npc.decisionmaker.core.conditions
**Type:** Transient Component

## Definition
```java
// Signature
public class LineOfSightCondition extends SimpleCondition {
```

## Architecture & Concepts
The LineOfSightCondition is a fundamental component within the server-side NPC Decision Maker framework. It functions as a declarative, boolean test used in behavior trees, state machines, or other AI logic structures to determine if an NPC has an unobstructed view of its current target.

This class embodies the principle of separation of concerns. It does **not** perform the complex geometric calculations for line-of-sight itself. Instead, it acts as a high-level bridge between the abstract decision-making layer and the concrete world-querying systems. During an evaluation, it retrieves the NPC's stateful PositionCache and delegates the actual raycasting or voxel traversal logic to it.

The presence of a static CODEC field is a critical architectural indicator. It signifies that this condition is not intended to be hard-coded but rather defined in external data files, such as JSON behavior assets. This allows game designers to construct and modify complex NPC AI without requiring engine recompilation. The getSimplicity method provides a cost metric, enabling the Decision Maker engine to optimize its evaluation path by executing cheaper conditions first.

### Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale serialization system via the static CODEC. This occurs when an NPC behavior asset is loaded from disk and its logic graph is constructed in memory.
- **Scope:** The object is stateless and effectively immutable after creation. Its lifetime is bound to the parent NPC behavior definition it is part of. A single instance is shared and reused across all evaluation ticks for any NPC that utilizes this specific behavior asset.
- **Destruction:** The object is marked for garbage collection when the server unloads the parent NPC behavior asset, for instance, during a zone shutdown or a hot-reload of game data.

## Internal State & Concurrency
- **State:** This class is entirely stateless. It contains no instance fields and its behavior is determined solely by the runtime arguments passed to the `evaluate` method.
- **Thread Safety:** The class is inherently thread-safe. The `evaluate` method is a pure function with respect to the instance itself. However, callers must be aware that the components it interacts with, particularly the CommandBuffer and the underlying world data accessed by PositionCache, are subject to strict threading models. The Decision Maker engine is responsible for ensuring that `evaluate` is invoked within a valid, thread-safe context, typically the main server tick for that NPC.

## API Surface
The public contract is minimal, designed for internal use by the Decision Maker engine.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getSimplicity() | int | O(1) | Returns a static cost value used by the AI engine to prioritize condition evaluation. |
| evaluate(...) | boolean | Variable | Executes the line-of-sight check. Complexity depends on the underlying PositionCache and world state, involving a potentially expensive world query. |

## Integration Patterns

### Standard Usage
This component is not intended for direct invocation by developers. It is configured within an NPC behavior asset and executed automatically by the Decision Maker engine.

A designer would define this condition declaratively in a data file:
```json
// Example NPC Behavior Asset (pseudo-JSON)
{
  "type": "BehaviorTree",
  "root": {
    "type": "Selector",
    "children": [
      {
        "type": "Sequence",
        "decorators": [
          {
            "type": "LineOfSightCondition"
          }
        ],
        "tasks": [
          // ... tasks to execute if target is visible
        ]
      }
    ]
  }
}
```

The engine then invokes the condition during the NPC's update cycle:
```java
// Simplified engine-level invocation
boolean canSeeTarget = condition.evaluate(
    npcEntityIndex,
    archetypeChunk,
    targetReference,
    commandBuffer,
    evaluationContext
);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new LineOfSightCondition()`. The object will lack proper integration with the engine's configuration and AI systems. It must be loaded via the asset pipeline.
- **Manual Invocation:** Do not call the `evaluate` method from outside the Decision Maker engine. The context, archetype chunk, and command buffer arguments are highly specific to the server's ECS update loop and cannot be safely constructed manually.

## Data Pipeline
The LineOfSightCondition acts as a node in the NPC decision-making data flow. It translates a high-level AI query into a concrete world query.

> Flow:
> Decision Maker Engine Tick -> **LineOfSightCondition.evaluate()** -> NPCEntity.getRole() -> PositionCache.hasLineOfSight() -> World Storage Raycast -> boolean Result -> Decision Maker Engine State Transition

