---
description: Architectural reference for the Instruction class, the core node of the NPC behavior tree.
---

# Instruction

**Package:** com.hypixel.hytale.server.npc.instructions
**Type:** Component / Node

## Definition
```java
// Signature
public class Instruction implements RoleStateChange, IAnnotatedComponentCollection {
```

## Architecture & Concepts

The Instruction class is the fundamental building block of the server-side NPC Behavior Tree system. It represents a single, conditional unit of logic that combines a sensory precondition with a set of resulting actions or motions. This class embodies the **Composite design pattern**, allowing Instructions to be arranged in a tree-like hierarchy.

An Instruction can function in two distinct capacities:
1.  **Leaf Node:** A terminal instruction that contains direct orders. It is composed of a **Sensor** (the condition) and one or more actions, such as a **BodyMotion**, **HeadMotion**, or an **ActionList**. When its Sensor evaluates to true, it executes these defined behaviors.
2.  **Branch Node:** A non-terminal instruction that acts as a container and controller for a list of child Instructions. Its purpose is to orchestrate the flow of logic. When its own Sensor evaluates to true, it begins evaluating its children to find a valid behavior to execute.

The entire behavior of an NPC is defined by a root Instruction, which contains the complete tree of all possible states and actions. The NPC's governing **Role** component is responsible for traversing this tree on each server tick, evaluating nodes via the **matches** method and executing them with the **execute** method.

Specialized boolean flags like *treeMode* and *continueAfter* provide sophisticated control over the execution flow, enabling complex, stateful sequences and fallback behaviors without requiring a separate state machine.

### Lifecycle & Ownership
-   **Creation:** Instruction objects are not intended for manual instantiation in game logic. They are constructed exclusively during the server's asset loading phase. A `BuilderInstruction`, which is a direct representation of the data in an NPC asset file, is processed by a `BuilderSupport` service to create a fully-realized Instruction tree. The static factory `createRootInstruction` is used to assemble the final tree for a given NPC Role.
-   **Scope:** An Instruction tree is owned by an NPC's `Role`. Its lifetime is bound to the `Role` itself, persisting as long as the NPC exists and its behavioral configuration is loaded in memory. The structure of the tree is considered immutable after creation.
-   **Destruction:** The object and its entire subtree are eligible for garbage collection when the parent `Role` is unloaded or the NPC entity is permanently removed from the world. The `unloaded` and `removed` callbacks, propagated via the `RoleStateChange` interface, ensure that all components in the tree can release resources or deregister from event listeners gracefully.

## Internal State & Concurrency
-   **State:** The structural state of an Instruction (its children, actions, motions, and configuration flags) is **immutable** after construction. However, its operational state is mutable. The underlying `Sensor` component contains transient state, such as `once` flags, which are modified during the evaluation tick. This design allows the static behavior tree to manage dynamic, short-term state without altering its core definition.
-   **Thread Safety:** **This class is not thread-safe.** All interactions with an Instruction tree, including lifecycle callbacks and evaluation methods like `matches` and `execute`, must occur on the main server thread that owns the NPC's world. Unsynchronized access from other threads will lead to severe race conditions, inconsistent behavior, and potential server corruption.

## API Surface

The public API is designed for traversal and execution by the parent `Role` system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | O(S) | Evaluates the internal Sensor to determine if this Instruction's conditions are met. S is the complexity of the Sensor logic. |
| execute(ref, role, dt, store) | void | O(N) | Executes the Instruction. For a leaf, this runs its actions. For a branch, it evaluates its N children. |
| loaded(role) | void | O(C) | Propagates the `loaded` lifecycle event to all C components in its subtree. Part of the RoleStateChange contract. |
| spawned(role) | void | O(C) | Propagates the `spawned` lifecycle event to all C components in its subtree. |
| unloaded(role) | void | O(C) | Propagates the `unloaded` lifecycle event to all C components in its subtree. |
| createRootInstruction(instructions, support) | static Instruction | O(1) | Factory method to create the top-level root node for a behavior tree. |

## Integration Patterns

### Standard Usage

Direct interaction with the Instruction class is rare. The system is designed to be driven by the NPC's `Role` component, which holds the reference to the root of the Instruction tree and manages its evaluation during the server tick. The `Role` is the sole intended user of the `matches` and `execute` methods.

```java
// This logic is representative of the internal loop within an NPC's Role component.
// Developers do not typically write this code.

// Assume 'rootInstruction' is the entry point for an NPC's behavior tree.
if (rootInstruction.matches(entityRef, thisRole, deltaTime, entityStore)) {
    rootInstruction.execute(entityRef, thisRole, deltaTime, entityStore);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new Instruction()` in game logic. Behavior trees must be defined in NPC asset files and loaded through the engine's asset pipeline. Bypassing this process will result in uninitialized and non-functional components.
-   **State Caching:** Do not cache the result of a `matches()` call across ticks. The underlying `Sensor` state is dynamic and must be re-evaluated each tick to ensure the NPC reacts correctly to changes in the world.
-   **External Modification:** Do not attempt to modify the `instructionList` or other fields after the object has been constructed. The behavior tree is designed to be immutable configuration data.

## Data Pipeline

The Instruction class is a key transformation point, converting static configuration data into dynamic in-game behavior.

> Flow:
> NPC Asset File (JSON) -> Asset Loader -> `BuilderInstruction` -> **`Instruction` Tree (In Memory)** -> `Role` Tick Evaluation -> `MotionController` Update -> Entity Component State Change -> Network Packet to Client

