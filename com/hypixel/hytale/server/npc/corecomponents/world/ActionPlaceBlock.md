---
description: Architectural reference for ActionPlaceBlock
---

# ActionPlaceBlock

**Package:** com.hypixel.hytale.server.npc.corecomponents.world
**Type:** Transient

## Definition
```java
// Signature
public class ActionPlaceBlock extends ActionBase {
```

## Architecture & Concepts
The **ActionPlaceBlock** class is a concrete implementation of the **ActionBase** contract, operating within the server-side NPC artificial intelligence framework. It represents a single, atomic behavior: an NPC placing a specified block at a target location in the world.

Architecturally, this class serves as a leaf node within a larger AI structure, such as a Behavior Tree or a Finite-State Machine. Its purpose is to translate an NPC's high-level goal—provided by its **Role** and sensory input—into a direct, low-level world state mutation.

The design strictly separates precondition checking from execution. The **canExecute** method acts as a logical gate, evaluating all constraints (range, target validity, world rules) without causing side effects. Only if this check passes can the AI scheduler commit to the action by calling **execute**, which performs the irreversible world modification. This pattern is fundamental for enabling the AI system to evaluate and prioritize multiple potential actions each tick.

## Lifecycle & Ownership
- **Creation:** An **ActionPlaceBlock** is not instantiated directly. It is constructed via its corresponding builder, **BuilderActionPlaceBlock**, which is typically configured from static NPC definition assets. This process occurs when an NPC's behavior set is initialized.
- **Scope:** The object's lifetime is bound to the NPC entity that owns it. It is a persistent component of the NPC's configured abilities, not a temporary object created per-tick.
- **Destruction:** The instance is eligible for garbage collection when its parent NPC entity is unloaded from the world or when its AI configuration is fundamentally changed.

## Internal State & Concurrency
- **State:** The class holds a mix of immutable and mutable state. Configuration parameters like **range** and **allowEmptyMaterials** are final and set at construction. However, it contains a mutable **target** Vector3d field. This vector is reused across method calls to store the calculated position, avoiding per-tick object allocations and reducing garbage collector pressure.

- **Thread Safety:** **This class is not thread-safe and must not be treated as such.** It is designed to be invoked exclusively from the main server thread, which governs the game loop and world state. The mutable **target** field makes concurrent access inherently unsafe. Any attempt to call its methods from an asynchronous task or worker thread will result in world corruption, race conditions, or crashes.

## API Surface
The public contract is focused on the two-phase execute pattern common in game AI.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| canExecute(ref, role, sensorInfo, dt, store) | boolean | O(1) | Evaluates if the action can be performed. Checks range, world placement rules, and NPC state. Complexity is constant time but involves world data access which may have its own performance characteristics. |
| execute(ref, role, sensorInfo, dt, store) | boolean | O(1) | Executes the block placement, mutating the world state. This method assumes **canExecute** was recently called and returned true. Returns true on successful execution. |

## Integration Patterns

### Standard Usage
This action is intended to be managed by a higher-level AI scheduler or behavior tree. The standard invocation pattern is to check **canExecute** every tick where the action is relevant. If it returns true, the scheduler may then invoke **execute**.

```java
// Within an NPC's AI update loop
// Assume 'action' is a configured ActionPlaceBlock instance

if (action.canExecute(npcRef, npcRole, sensorInfo, dt, worldStore)) {
    // The AI has decided to commit to this action
    action.execute(npcRef, npcRole, sensorInfo, dt, worldStore);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use a constructor directly. The class must be configured and created via its corresponding **BuilderActionPlaceBlock** to ensure its state is valid.
- **State Caching:** Do not cache the result of **canExecute** across multiple ticks. The validity of a block placement can change at any moment due to other world events. It must be re-evaluated each time the action is considered.
- **Off-Thread Execution:** Invoking any method on this class from a thread other than the main server thread is strictly forbidden. All world interactions must be synchronized with the primary game loop.

## Data Pipeline
The flow of data through this component involves receiving high-level intent and sensory data, processing it against world state, and finally outputting a world mutation.

> Flow:
> NPC Sensor System -> Target Position -> **ActionPlaceBlock.canExecute** -> NPC Role (Provides Block Type) -> World State (Validation) -> **ActionPlaceBlock.execute** -> WorldChunk.setBlock (Mutation)

