---
description: Architectural reference for BrushConfigCommandExecutor
---

# BrushConfigCommandExecutor

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes
**Type:** Transient

## Definition
```java
// Signature
public class BrushConfigCommandExecutor {
```

## Architecture & Concepts

The BrushConfigCommandExecutor is the runtime engine for Hytale's scripted brush system. It acts as an interpreter that translates a declarative BrushConfig object—a script composed of sequential and global operations—into imperative world modifications. This class is the central component responsible for managing the state and execution flow of a single brush stroke.

Its primary architectural role is to bridge player input with the complex logic of a scripted brush. When a player uses a tool, this executor is instantiated to manage the entire lifecycle of that single interaction. It iterates through a list of SequenceBrushOperation objects, applying each one in order. It also processes GlobalBrushOperation objects, which perform pre-execution modifications to the brush's configuration.

The executor maintains a state machine that tracks the current operation index, manages variables, handles control flow (jumps), and facilitates debugging features like step-through execution and breakpoints. It does not modify the world directly; instead, it populates a BrushConfigEditStore with pending changes, which are then flushed to the world by the parent BuilderToolsPlugin system.

## Lifecycle & Ownership

-   **Creation:** An instance is created by the BuilderToolsPlugin each time a player initiates an action with a scripted brush. It is initialized with the specific BrushConfig that defines the tool's behavior.
-   **Scope:** The object's lifetime is ephemeral, scoped exclusively to a single brush execution. It is created, the primary `execute` method is called, and upon completion, the instance is no longer used.
-   **Destruction:** The object becomes eligible for garbage collection as soon as the `execute` method returns and all references to it are dropped. There is no persistent state held within this class between distinct player interactions; persistent variables are managed by this class but their lifetime is tied to the executor's own short scope.

## Internal State & Concurrency

-   **State:** This class is highly stateful and mutable. It maintains critical execution state including the `currentOperationIndex`, collections of stored variables (`persistentStoredVariables`), brush configuration snapshots (`brushConfigStoredSnapshots`), and debugging flags. The entire execution process is dependent on the careful management of this internal state.

-   **Thread Safety:** **This class is not thread-safe and must not be accessed from multiple threads.** All operations are designed to be executed synchronously on the main server thread. Any concurrent access to its methods or internal state will result in race conditions, state corruption, and unpredictable server behavior. The lack of any synchronization primitives (locks, atomics) is intentional, as it is architected for single-threaded access within the server's game loop.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(...) | void | O(N*M) | The primary entry point. Orchestrates the entire execution of all sequential operations. N is the number of operations, M is the complexity of each operation. |
| step(...) | BrushConfig.BCExecutionStatus | O(M) | Executes a single operation from the sequence. Used for debug stepping. |
| resetInternalState() | void | O(1) | Clears all transient execution state, preparing the instance for a new run. |
| storeOperatingIndex(name, index) | void | O(1) | Stores the current execution index under a given name, enabling future jumps. |
| loadOperatingIndex(name) | void | O(1) | Jumps the execution pointer to a previously stored index. This is a GOTO-like operation. |
| setPersistentVariable(name, value) | void | O(1) | Stores a key-value pair that can be accessed by subsequent operations within the same execution. |
| storeBrushConfigSnapshot(name) | void | O(1) | Creates a deep copy of the current BrushConfig state and stores it. |
| loadBrushConfigSnapshot(name, flags) | void | O(1) | Restores parts of the BrushConfig from a previously saved snapshot. |

## Integration Patterns

### Standard Usage

The executor is designed to be instantiated and used by a higher-level tool management system. The caller provides the context of the player interaction, and the executor handles the procedural generation logic defined in the BrushConfig.

```java
// A tool handler retrieves the appropriate BrushConfig for the player's tool
BrushConfig config = getBrushConfigForTool();

// A new executor is created for this specific interaction
BrushConfigCommandExecutor executor = new BrushConfigCommandExecutor(config);

// The executor is run with the full context of the player's action
executor.execute(
    playerEntityRef,
    world,
    interactionOrigin,
    isHoldDown,
    interactionType,
    componentAccessor
);
// The executor instance is now discarded.
```

### Anti-Patterns (Do NOT do this)

-   **Instance Re-use:** Do not re-use an executor instance for a new brush stroke without calling `resetInternalState`. The recommended and safest pattern is to create a new instance for every distinct user interaction. Failure to do so will leak state between operations.
-   **Asynchronous Execution:** Do not invoke `execute` or `step` from a separate thread. All interactions with this class must be on the main server thread to prevent world corruption and concurrency exceptions.
-   **External State Mutation:** Do not modify the internal state, such as `currentOperationIndex`, directly. Use the provided API methods like `loadOperatingIndex` or `setCurrentlyExecutingActionIndex` to manipulate control flow. Direct field manipulation will break the interpreter's logic.

## Data Pipeline

The BrushConfigCommandExecutor is a central processing unit in the scripted brush data flow. It consumes a configuration and context, and produces a set of world edits.

> Flow:
> Player Input -> BuilderToolsPlugin -> **BrushConfigCommandExecutor** (interprets BrushConfig) -> Iterates over `SequenceBrushOperation`s -> Populates `BrushConfigEditStore` -> BuilderToolsPlugin queues world modification task.

