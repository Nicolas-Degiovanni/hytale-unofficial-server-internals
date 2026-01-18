---
description: Architectural reference for HasTargetCondition
---

# HasTargetCondition

**Package:** com.hypixel.hytale.server.npc.decisionmaker.core.conditions
**Type:** Transient

## Definition
```java
// Signature
public class HasTargetCondition extends SimpleCondition {
```

## Architecture & Concepts
The HasTargetCondition is a fundamental building block within the server-side NPC Artificial Intelligence engine, specifically the Decision Maker framework. It functions as a leaf node in a behavior tree or a simple predicate in a state machine, designed to answer a single, critical question: "Does this NPC currently have an entity assigned to a specific target slot?"

This class is not intended for direct, imperative use. Instead, it is designed to be defined in data files (e.g., JSON or HOCON) that describe an NPC's behavior. The static **CODEC** field is the key to this data-driven approach, enabling the game engine to deserialize an AI definition into a live, executable object graph.

During an AI tick, the Decision Maker system evaluates this condition for a given NPC. The condition reads the NPC's state from the Entity Component System (ECS) but never modifies it. Its sole responsibility is to return a boolean value that influences which branch of a behavior tree is executed.

## Lifecycle & Ownership
-   **Creation:** Instances are created exclusively by the Hytale serialization system via the public static **CODEC** field. This typically occurs when the server loads NPC behavior assets from disk at startup or during a resource reload. The constructor is protected to prevent direct instantiation.
-   **Scope:** The object's lifetime is bound to the loaded AI behavior definition it is part of. It is effectively a singleton within the context of that definition and is shared by all NPC entities that use the same behavior asset. It does not store per-NPC state.
-   **Destruction:** The object is marked for garbage collection when its parent AI behavior definition is unloaded by the server.

## Internal State & Concurrency
-   **State:** The internal state is immutable after creation. The **targetSlot** field is populated once during deserialization and is never modified thereafter. The class is stateless with respect to the game world; it queries world state via its method parameters but retains none of it.
-   **Thread Safety:** This class is inherently thread-safe. Its immutable internal state and lack of side effects in the **evaluate** method allow it to be safely executed by multiple threads in a parallel AI update system. Thread safety of the overall evaluation depends on the guarantees provided by the calling context and the underlying ECS data structures.

## API Surface
The primary interaction point is the **evaluate** method, inherited from its parent class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| evaluate(...) | boolean | O(1) | (Protected) The core logic. Retrieves the NPCEntity component for the current entity and checks if a target exists in the configured slot. |
| getTargetSlot() | String | O(1) | Returns the name of the target slot this condition is configured to check. |
| CODEC | BuilderCodec | N/A | A static factory responsible for creating HasTargetCondition instances from serialized data. Enforces validation rules, such as requiring a non-empty target slot. |

## Integration Patterns

### Standard Usage
This component is not invoked directly in Java code. It is configured within an NPC behavior asset file. The AI system then instantiates and executes it automatically.

*Example NPC Behavior Definition (Conceptual YAML)*
```yaml
# In an NPC's behavior tree definition file
- type: "Selector"
  children:
    - type: "Sequence"
      conditions:
        # The engine deserializes this block into a HasTargetCondition instance
        - type: "HasTargetCondition"
          TargetSlot: "primary_threat"
      actions:
        - type: "AttackTarget"
          TargetSlot: "primary_threat"
    - type: "Sequence"
      actions:
        - type: "Wander"
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new HasTargetCondition()`. The object will be in an invalid state with a null **targetSlot**, leading to runtime exceptions. Always define conditions in data files.
-   **Stateful Logic:** Do not attempt to subclass this condition to add state. Conditions in the Decision Maker framework are expected to be stateless and reusable across many NPC instances.

## Data Pipeline
The HasTargetCondition acts as a gate in the flow of AI logic. It consumes world state and produces a boolean signal that directs the subsequent behavior of an NPC.

> Flow:
> NPC Behavior Asset File -> Server Asset Loader -> **CODEC Deserialization** -> In-Memory Behavior Tree -> AI System Tick -> **HasTargetCondition.evaluate()** -> Boolean Result -> Behavior Tree Traversal (e.g., Attack or Wander)

