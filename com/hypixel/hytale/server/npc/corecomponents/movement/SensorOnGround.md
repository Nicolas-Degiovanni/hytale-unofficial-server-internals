---
description: Architectural reference for SensorOnGround
---

# SensorOnGround

**Package:** com.hypixel.hytale.server.npc.corecomponents.movement
**Type:** Transient Component

## Definition
```java
// Signature
public class SensorOnGround extends SensorBase {
```

## Architecture & Concepts
The SensorOnGround class is a fundamental component within the server-side NPC Artificial Intelligence framework. It functions as a *predicate* or a *condition checker*, designed to be integrated into higher-level AI constructs like Behavior Trees or Finite State Machines.

Its sole responsibility is to determine if an NPC entity is physically in contact with the ground. This is a critical state check for controlling movement-based behaviors. For example, an NPC's decision to jump, attack, or flee might be gated by the condition that it must first be on solid ground.

This sensor acts as a bridge between the abstract AI decision-making layer and the concrete physics state of an NPC, which is managed by the NPC's active MotionController. It is a leaf node in the AI logic tree, providing a simple boolean state that other, more complex nodes can use to alter an NPC's behavior.

### Lifecycle & Ownership
- **Creation:** SensorOnGround is not instantiated directly. It is constructed by the NPC configuration system, specifically via a BuilderSensorBase, during the parsing and assembly of an NPC's behavior profile from asset files.
- **Scope:** The object's lifetime is bound to the NPC's loaded AI definition. It is effectively an immutable, stateless component that persists as long as its parent AI configuration is in memory.
- **Destruction:** The object is eligible for garbage collection when the NPC entity is destroyed or its AI definition is unloaded. It requires no explicit cleanup.

## Internal State & Concurrency
- **State:** This class is **stateless**. It contains no mutable fields and its evaluation logic depends entirely on the arguments passed to its methods, primarily the Role object representing the NPC.
- **Thread Safety:** The class is inherently thread-safe due to its stateless nature. However, it is **not safe** to invoke its methods from arbitrary threads. The underlying Role and MotionController objects it queries are part of the server's main game loop.

    **WARNING:** All invocations of SensorOnGround must occur on the primary server thread responsible for the NPC's update tick. Accessing it from other threads will lead to race conditions and inconsistent world state.

## API Surface
The public contract is minimal, focused on the evaluation logic inherited and implemented from SensorBase.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | O(1) | Evaluates to true if the NPC represented by the Role is on the ground. This check is delegated to the NPC's active MotionController. |

## Integration Patterns

### Standard Usage
A developer or designer will typically not interact with this class in Java code. Instead, it is declared within an NPC's data-driven behavior definition (e.g., a JSON file) and invoked automatically by the AI engine.

Conceptually, its usage within a Behavior Tree would look like this:

```java
// PSEUDO-CODE: Conceptual use in a Behavior Tree
// This code does not exist; it illustrates the design pattern.

SelectorNode root = new SelectorNode();

// Sequence: Only execute if the condition is met
SequenceNode jumpSequence = new SequenceNode();
jumpSequence.addCondition(new SensorOnGround()); // The sensor acts as a gate
jumpSequence.addAction(new ActionJump());

root.addChild(jumpSequence);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new SensorOnGround()`. The object lacks the necessary context when created this way and bypasses the AI's configuration pipeline. It must be created via its corresponding builder.
- **Stateful Logic:** Do not extend this class to add state. Sensors are designed to be reusable, stateless predicates. If you need to store information, use the AI's blackboard or memory system.
- **External Invocation:** Do not call the `matches` method from systems outside the NPC's own AI update loop. The result is only guaranteed to be valid during that specific phase of the game tick.

## Data Pipeline
SensorOnGround does not process a stream of data. Instead, it participates in a control flow by answering a question about the game state.

> Flow:
> NPC Behavior Tree Tick -> Evaluates Condition Node -> **SensorOnGround.matches(role)** -> Accesses role.getActiveMotionController() -> Calls onGround() -> Returns boolean -> Behavior Tree alters execution path

