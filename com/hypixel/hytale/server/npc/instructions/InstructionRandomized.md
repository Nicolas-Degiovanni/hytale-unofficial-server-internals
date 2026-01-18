---
description: Architectural reference for InstructionRandomized
---

# InstructionRandomized

**Package:** com.hypixel.hytale.server.npc.instructions
**Type:** Transient

## Definition
```java
// Signature
public class InstructionRandomized extends Instruction {
```

## Architecture & Concepts
The InstructionRandomized class is a composite control-flow node within the server-side NPC Behavior System. Its primary function is to introduce non-deterministic behavior by selecting and executing one of its child Instructions based on a weighted probability distribution. This allows for more dynamic and less predictable NPC actions.

It operates as a stateful selector. When its execution is triggered, it checks if a child Instruction is already active and if its allotted time has expired. If no instruction is active or the timer has run out, it performs a weighted random selection from its collection of child Instructions. The chosen child is then executed for a randomized duration, defined by a minimum and maximum time range.

This component is fundamental for creating lifelike AI. Instead of a rigid sequence of actions, an NPC can be configured to, for example, wander for a random period, then have a 70% chance to look at a point of interest and a 30% chance to play an idle animation.

## Lifecycle & Ownership
-   **Creation:** InstructionRandomized is not instantiated directly. It is constructed by the NPC asset loading pipeline, specifically via a BuilderInstructionRandomized. This builder reads the NPC's behavioral definition from an asset file, instantiates the child Instructions, and assembles them into the weighted map required by this class's constructor.
-   **Scope:** An instance of InstructionRandomized has its lifecycle tied directly to the NPC entity that owns it. It exists as a node within the behavior tree managed by the NPC's Role component and persists as long as that Role is active.
-   **Destruction:** The object is eligible for garbage collection when the parent NPC entity is unloaded from the world or when its behavior tree is replaced. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
-   **State:** This class is highly **mutable** and stateful.
    -   **weightedInstructionMap:** An immutable, weighted collection of child instructions, populated at creation time.
    -   **timeout:** A mutable countdown timer that dictates how long the currently selected child instruction will be executed.
    -   **current:** A mutable reference to the currently active child InstructionHolder. This field represents the primary state of the component.
    -   **resetOnStateChange:** An immutable boolean flag that controls whether the `current` instruction is cleared when the parent behavior state changes.

-   **Thread Safety:** This class is **not thread-safe** and must only be accessed from the main server thread responsible for ticking the NPC's world. Its internal state, particularly the `timeout` and `current` fields, is modified without any synchronization primitives. The use of ThreadLocalRandom further reinforces its design for single-threaded execution within the game loop. Concurrent access would lead to race conditions and corrupt the NPC's behavior logic.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(ref, role, dt, store) | void | O(log N) | The primary update method called each game tick. Manages the timeout, selects a new child instruction if necessary, and executes the current one. The complexity reflects the weighted random selection, which is typically logarithmic. |
| clearOnce() | void | O(1) | Conditionally resets the `current` active instruction based on the `resetOnStateChange` flag. Part of the state transition contract. |
| reset() | void | O(1) | Unconditionally resets the `current` active instruction to null, forcing a re-selection on the next `execute` call. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by developers. It is configured declaratively within an NPC's asset files. The game engine's asset loader is responsible for parsing this configuration and constructing the object graph.

The following conceptual example illustrates how the system invokes an already-constructed Instruction.

```java
// Within the server's NPC update loop for a specific Role

// 'instruction' is an instance of InstructionRandomized,
// configured and loaded from an NPC asset file.
instruction.execute(entityRef, npcRole, deltaTime, entityStore);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new InstructionRandomized()`. The constructor requires a fully resolved list of child instructions and a BuilderSupport context, which are only available during the asset loading phase. Manual creation will result in a misconfigured and non-functional behavior node.
-   **External State Modification:** Do not modify the public `current` or `timeout` fields from outside the class. The internal timing and selection logic is self-contained and will be broken by external interference.
-   **Concurrent Execution:** Never call the `execute` method from any thread other than the world's main update thread. Doing so will cause state corruption and unpredictable server behavior.

## Data Pipeline
The flow of control and configuration for this class begins with game design assets and ends with entity state modification in the game world.

> Flow:
> NPC Asset File (JSON) -> BuilderInstructionRandomized -> **InstructionRandomized Instance** -> Role Component -> Game Tick -> `execute()` -> Selected Child Instruction -> Entity State Update

