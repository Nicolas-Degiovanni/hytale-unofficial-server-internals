---
description: Architectural reference for ActionSetMarkedTarget
---

# ActionSetMarkedTarget

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity
**Type:** Transient Command

## Definition
```java
// Signature
public class ActionSetMarkedTarget extends ActionBase {
```

## Architecture & Concepts
ActionSetMarkedTarget is a concrete implementation of the Command Pattern within the server-side NPC AI framework. It serves as a fundamental building block in an NPC's decision-making logic, such as a Behavior Tree or a Finite State Machine.

Its primary function is to bridge an NPC's real-time sensory input with its persistent memory. The action captures the current target entity identified by the NPC's sensor system (via InfoProvider) and stores a reference to it in a designated memory slot within the NPC's Role. This "marking" of a target allows subsequent, decoupled actions to operate on a consistent entity without needing to perform their own sensory checks.

For example, a sequence of actions could be:
1.  *FindEnemy* (Sensor Action)
2.  **ActionSetMarkedTarget** (Memory Action)
3.  *ActionMoveToMarkedTarget* (Movement Action)
4.  *ActionAttackMarkedTarget* (Combat Action)

This component is entirely data-driven. Its behavior, specifically which memory slot to use, is configured in external NPC asset files and instantiated by the engine through its corresponding builder, BuilderActionSetMarkedTarget.

### Lifecycle & Ownership
-   **Creation:** Instances are not created directly via code. They are instantiated by the server's asset loading pipeline when an NPC's behavior definition is parsed. The BuilderActionSetMarkedTarget reads the configuration (e.g., from a JSON file) and constructs the immutable ActionSetMarkedTarget object.
-   **Scope:** The object is stateless and reusable. A single instance is created for a given NPC asset definition and is shared by all NPC instances of that type. Its lifetime is tied to the loaded asset collection.
-   **Destruction:** The object is eligible for garbage collection when the server unloads or reloads the associated NPC asset definitions.

## Internal State & Concurrency
-   **State:** Effectively immutable. The single internal field, targetSlot, is final and set during construction. This class holds no per-NPC instance state, making it a pure behavioral component.
-   **Thread Safety:** This class is inherently thread-safe due to its immutable state. However, it is designed for synchronous execution within the main server's entity update loop for a single NPC at a time. The objects it receives in the execute method, such as Role and EntityStore, are not guaranteed to be thread-safe and must not be manipulated from other threads during execution.

## API Surface
The public contract is minimal, consisting only of the core execution method inherited from ActionBase.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(ref, role, sensorInfo, dt, store) | boolean | O(1) | Writes the current sensor target to the configured slot in the NPC's Role. Always returns true, indicating immediate completion. |

## Integration Patterns

### Standard Usage
This action is not intended to be invoked directly from Java code. It is designed to be declared within an NPC's asset definition file as a step in a larger AI behavior.

The following conceptual example shows how it might be used in a YAML-based behavior tree definition.

```yaml
# Conceptual NPC Behavior Asset
# This is NOT a real file format, but illustrates the design pattern.
behaviorTree:
  - sequence:
      - description: "Find, Mark, and Approach an enemy"
      - action: FindNearestEnemy # A sensor action that populates the sensorInfo target
      - action: SetMarkedTarget
        # Configures this action to use memory slot 0
        targetSlot: 0
      - action: MoveToMarkedTarget
        # A subsequent action that reads from memory slot 0
        targetSlot: 0
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call new ActionSetMarkedTarget(). The object's configuration is critical and must be supplied by the asset pipeline via its builder. Direct instantiation will result in an unconfigured and non-functional component.
-   **Stateful Modification:** Do not extend this class to add mutable, per-NPC state. All runtime state must be stored and managed in the Role or other state-bearing components passed into the execute method. This action must remain a stateless, reusable command.

## Data Pipeline
The primary purpose of this class is to act as a conduit, moving data from the sensory system into the NPC's working memory.

> Flow:
> NPC Sensor System → InfoProvider.getTarget() → **ActionSetMarkedTarget.execute()** → Role.getMarkedEntitySupport().setMarkedEntity() → Downstream AI Actions

