---
description: Architectural reference for ActionNothing
---

# ActionNothing

**Package:** com.hypixel.hytale.server.npc.corecomponents.utility
**Type:** Transient

## Definition
```java
// Signature
public class ActionNothing extends ActionBase {
```

## Architecture & Concepts
The ActionNothing class is a concrete implementation of the Null Object pattern within the server-side NPC behavior system. It represents a deliberate, non-operational action that an NPC can perform. Its primary role is to provide a safe, explicit "do nothing" state within complex behavior trees or finite state machines, eliminating the need for null checks and special-case handling for idle or terminal states.

Architecturally, ActionNothing serves as a leaf node in an NPC's behavior tree. When the behavior tree traversal selects this action for execution, the NPC controller effectively skips its action phase for that tick. This is crucial for designing behaviors such as waiting, pausing after an action, or defining a default state when no other action is applicable. By providing a concrete type for inaction, the system maintains structural integrity and avoids the fragility associated with null references in the action execution pipeline.

## Lifecycle & Ownership
- **Creation:** An ActionNothing instance is constructed by its corresponding builder, BuilderActionBase, during the assembly of an NPC's behavior tree. This typically occurs when the server loads NPC configurations from asset files. It is not intended for direct, dynamic instantiation during gameplay.
- **Scope:** The object's lifetime is tied to the lifecycle of the parent NPC behavior tree configuration. As it is stateless, it is a candidate for the Flyweight pattern, where a single, shared instance could be referenced by multiple nodes or even multiple NPC instances to conserve memory.
- **Destruction:** The instance is eligible for garbage collection when the NPC behavior tree it belongs to is unloaded, typically when an NPC type is no longer active on the server or during a server shutdown sequence.

## Internal State & Concurrency
- **State:** Inherently immutable. The ActionNothing class introduces no new state beyond what is inherited from ActionBase. Its defining characteristic is its lack of specific logic or data.
- **Thread Safety:** This class is unconditionally thread-safe. Due to its immutable nature, a single instance can be safely shared and executed by multiple NPC controllers across different threads without requiring any synchronization mechanisms.

## API Surface
The public contract of ActionNothing is entirely inherited from its parent, ActionBase. It introduces no new methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ActionNothing(builder) | Constructor | O(1) | Constructs the action. Restricted to be called by its corresponding builder. |

## Integration Patterns

### Standard Usage
ActionNothing is used declaratively within an NPC behavior tree definition to specify a period of inaction. It is a fundamental component for controlling the pacing and logic of AI.

```java
// Conceptual example of a behavior tree definition
Sequence root = new Sequence(
    new ActionFindTarget(),
    new ActionMoveToTarget(),
    new ActionAttackTarget(),
    new ActionNothing() // Represents a cooldown or pause after attacking
);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call new ActionNothing(). The NPC action system relies on the builder pattern to correctly initialize and wire up actions within the behavior tree. Bypassing the builder can lead to an improperly configured and non-functional action.
- **Subclassing:** Do not extend ActionNothing. Its purpose is to be a final, atomic no-op. Creating a subclass violates the Null Object pattern and suggests a design flaw; if behavior is needed, a different ActionBase implementation should be created instead.
- **Conditional Checks:** Avoid checks like *if (action instanceof ActionNothing)*. The system is designed for the action to be executed polymorphically. The correct approach is to let the ActionNothing execute its empty logic, rather than building special-case logic around it.

## Data Pipeline
ActionNothing does not process data; it halts a control flow pipeline for a single execution tick.

> Flow:
> NPC Behavior Tree Processor -> Selects **ActionNothing** as current action -> Executor calls execute() -> No operation occurs -> Control returns to Processor

