---
description: Architectural reference for IsInStateCondition
---

# IsInStateCondition

**Package:** com.hypixel.hytale.server.npc.decisionmaker.core.conditions
**Type:** Transient Data Object

## Definition
```java
// Signature
public class IsInStateCondition extends SimpleCondition {
```

## Architecture & Concepts
The **IsInStateCondition** is a fundamental predicate component within the server-side NPC Decision Maker framework. It functions as a leaf node in an NPC's behavior tree, answering a simple, critical question: "Is the NPC currently in a specific state?".

This class acts as a bridge between the declarative world of NPC behavior configuration (defined in asset files) and the imperative, runtime state of an entity. The engine's codec system deserializes a configuration block into an instance of this class. The **DecisionMaker** system then invokes the **evaluate** method during its update tick to determine if this condition is met, which in turn influences which branch of the behavior tree is executed.

Its sole responsibility is to query the **StateSupport** component of an **NPCEntity**, abstracting the low-level state-checking logic away from the higher-level behavior tree structure.

## Lifecycle & Ownership
- **Creation:** Instances of **IsInStateCondition** are not created directly using the constructor. They are exclusively instantiated and configured by the Hytale codec system via the static **CODEC** field. This process occurs when the server loads NPC behavior definitions from game assets.

- **Scope:** The object's lifetime is bound to the loaded NPC behavior definition it is part of. It is effectively a stateless, reusable configuration object that persists as long as the parent NPC definition is held in memory.

- **Destruction:** The object is eligible for garbage collection when its parent NPC behavior definition is unloaded, for instance, during a server shutdown or a hot-reload of game assets. There is no manual destruction mechanism.

## Internal State & Concurrency
- **State:** The internal state, consisting of the target **state** and **subState** strings, is set once during deserialization. After this initial configuration, the object is effectively immutable. The **evaluate** method is a pure function with respect to the object's own state; it does not modify any internal fields.

- **Thread Safety:** This class is inherently thread-safe. Its immutable nature after creation prevents data races. The **evaluate** method can be safely called from any thread, provided the passed-in ECS parameters (**ArchetypeChunk**, **CommandBuffer**, etc.) are handled according to the engine's threading model. The method itself performs no locking.

## API Surface
The primary contract is the inherited **evaluate** method, which is invoked by the NPC decision-making system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| evaluate(...) | boolean | O(1) | Executes the state check against the target NPC. Returns true if the NPC's current state matches the configured state and sub-state. |
| getState() | String | O(1) | Returns the configured primary state string. |

## Integration Patterns

### Standard Usage
This component is not intended to be used directly in procedural Java code. Instead, it is defined declaratively within an NPC's behavior asset file. The engine's **DecisionMaker** system consumes this configuration.

*Conceptual NPC Behavior Asset:*
```yaml
# This is a conceptual example of how IsInStateCondition would be defined in an asset file.
# The actual format may vary.

behaviorTree:
  root:
    type: Selector
    children:
      - type: Sequence
        children:
          - type: IsInStateCondition # The engine maps this type to our class
            State: "Combat"
            SubState: "Enraged"
          - type: PerformAttackAction
            target: "@player"
      - type: Sequence
        children:
          - type: IsInStateCondition
            State: "Idle"
          - type: WanderAction
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call **new IsInStateCondition()**. The resulting object will be invalid as its internal state fields (**state**, **subState**) will be null, leading to a **NullPointerException** during evaluation. Always define conditions in data assets.

- **External Invocation:** Do not call the **evaluate** method from outside the engine's **DecisionMaker** or a related AI system. The method requires a valid and correctly populated ECS context (**ArchetypeChunk**, **selfIndex**, etc.) which is managed exclusively by the engine's entity processing loop.

## Data Pipeline
The primary flow for this component is from a static data definition to a runtime boolean evaluation.

> Flow:
> NPC Behavior Asset (e.g., JSON/YAML) -> Server Asset Loader -> **BuilderCodec** -> **IsInStateCondition Instance** -> DecisionMaker System -> **evaluate()** -> Boolean Result -> Behavior Tree Execution

