---
description: Architectural reference for ActionCompleteTask
---

# ActionCompleteTask

**Package:** com.hypixel.hytale.builtin.adventure.npcobjectives.npc
**Type:** Transient

## Definition
```java
// Signature
public class ActionCompleteTask extends ActionPlayAnimation {
```

## Architecture & Concepts
ActionCompleteTask is a specialized **Action** within the Hytale NPC AI framework. It functions as a concrete implementation of the Command Pattern, encapsulating the logic required for an NPC to mark a player's objective or task as complete.

Architecturally, this class serves as a critical bridge between two distinct systems:
1.  The low-level **NPC Behavior System**, which orchestrates an entity's moment-to-moment actions.
2.  The high-level **Quest and Objective System**, managed by the NPCObjectivesPlugin.

By inheriting from ActionPlayAnimation, it extends the base functionality of playing an animation with quest-specific logic. This is a common engine pattern where generic behaviors are specialized for gameplay purposes. This action is designed to be configured within an NPC's behavior assets and is executed by the server-side AI state machine when specific conditions are met, typically upon successful interaction with a player.

### Lifecycle & Ownership
-   **Creation:** This object is not instantiated directly. It is constructed by its corresponding builder, BuilderActionCompleteTask, during the server's asset loading phase. The engine parses NPC configuration files (e.g., JSON behavior trees) and uses builders to create the graph of potential actions for an NPC's Role.
-   **Scope:** An instance of ActionCompleteTask is stateless and reusable. Its lifetime is tied to the NPC's loaded Role definition. It persists as long as the NPC's behavior configuration is active in memory and is re-executed whenever the AI logic dictates.
-   **Destruction:** The object is eligible for garbage collection when the parent NPC entity is unloaded or its Role definition is replaced. There is no manual destruction method.

## Internal State & Concurrency
-   **State:** This class is effectively immutable after its initial construction. The primary configuration, such as the boolean flag playAnimation, is final. The animation to be played is determined dynamically within the execute method and is not persistent state. This design ensures that the same action instance can be safely re-used across multiple game ticks without risk of state corruption.
-   **Thread Safety:** **This class is not thread-safe and must not be considered as such.** All methods, particularly canExecute and execute, perform unsynchronized reads and writes on the core Entity Component System (ECS) Store. Execution is strictly bound to the main server thread that processes NPC AI updates.

    **WARNING:** Accessing this object from any worker thread will lead to severe data corruption, race conditions, and unpredictable server behavior.

## API Surface
The public contract is defined by the base Action class and is intended for consumption by the NPC Role and AI systems only.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| canExecute(...) | boolean | O(1) | Checks if the action can run. Verifies the interaction target exists and is not dead. |
| execute(...) | boolean | O(T) | Executes the core logic. Updates player tasks via NPCObjectivesPlugin. T is the number of active tasks. |

## Integration Patterns

### Standard Usage
This class is not intended to be invoked directly in gameplay code. A game designer or developer configures it declaratively within an NPC's behavior asset file. The engine's AI system is then responsible for its execution.

A conceptual configuration might look like this:
```yaml
# Example NPC Behavior Asset (Conceptual)
# This is NOT a real code example.
behaviors:
  - onPlayerInteract:
      sequence:
        - action: "GreetPlayer"
        - action: "ActionCompleteTask"
          # The engine maps this string to the correct builder
          # and instantiates the class.
          playAnimation: true
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new ActionCompleteTask()`. The object is complex to construct and requires a builder context provided by the asset loading system. Direct instantiation will fail or result in a non-functional object.
-   **External State Modification:** Do not call `setAnimationId` from outside the class. This method is part of the internal execution flow and is managed by the parent class contract. Modifying it externally will break the action's logic.
-   **Off-Thread Execution:** As stated previously, invoking any method from a separate thread is strictly forbidden and will destabilize the server.

## Data Pipeline
The flow of data and control for this action is linear and synchronous, occurring within a single server tick.

> Flow:
> 1.  **NPC AI Tick:** The NPC's behavior tree or state machine determines it is time to complete a task for the interacting player.
> 2.  **Role System:** The NPC's current Role invokes `ActionCompleteTask.execute()`.
> 3.  **ECS Read:** The `execute` method reads the target player's entity reference from the Role's state and fetches the player's active tasks.
> 4.  **External System Call:** It calls `NPCObjectivesPlugin.updateTaskCompletion`, passing the NPC UUID and player data. This is the primary side-effect, mutating the player's quest state.
> 5.  **Data Return:** The plugin returns a potential animation name to play as feedback.
> 6.  **ECS Write (Animation):** If an animation is requested and returned, the action calls its parent `super.execute()`, which writes a new animation component or message to the ECS for the NPC entity.
> 7.  **Result:** The player's task state is updated, and the NPC may begin playing a confirmation animation.

