---
description: Architectural reference for FloodOperation
---

# FloodOperation

**Package:** com.hypixel.hytale.builtin.buildertools.tooloperations
**Type:** Transient

## Definition
```java
// Signature
public class FloodOperation extends ToolOperation {
```

## Architecture & Concepts
The FloodOperation class is a server-side implementation of the Command design pattern. It encapsulates all the information required to perform a single, atomic "flood fill" or "paint bucket" world edit, initiated by a player using the builder tools.

Its primary architectural role is to act as a translator and executor. It receives a raw network packet, `BuilderToolOnUseInteraction`, and transforms it into a high-level world modification command. It does not contain the flood fill algorithm itself; instead, it gathers the necessary context—such as the target block, the fill pattern, and the player's state—and delegates the core logic to the state management system within the `BuilderToolsPlugin`.

This class is a fundamental component of the builder tools feature, enabling large-scale, contiguous block replacement operations. It operates within the server's world transaction system, ensuring that the entire fill operation is applied atomically.

### Lifecycle & Ownership
- **Creation:** Instantiated by the `BuilderToolsPlugin` packet handler upon receiving a `BuilderToolOnUseInteraction` packet from a game client. The creation is a direct result of a player's in-game action.
- **Scope:** Extremely short-lived. A FloodOperation object exists only for the duration of a single server tick to process one user interaction. It is a single-use object.
- **Destruction:** The object becomes eligible for garbage collection as soon as its `execute` method completes. It holds no persistent state and is not referenced outside the scope of the transaction in which it is executed.

## Internal State & Concurrency
- **State:** FloodOperation is stateful upon creation, containing all parameters for the operation (e.g., origin coordinates, player reference, fill pattern). This state is derived from its constructor arguments and is treated as immutable for the object's lifetime. It does not modify its own state after initialization.
- **Thread Safety:** This class is **not thread-safe** and must not be used concurrently. It is designed to be created, executed, and discarded exclusively within the server's main game loop thread. All its interactions with world data via the `ComponentAccessor` and `edit` context assume single-threaded access.

## API Surface
The public API is minimal, consisting of the execution entry point inherited from its parent.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(accessor) | void | O(V) | Executes the flood fill operation. Complexity is proportional to V, the volume of blocks in the filled region. Delegates the core algorithm to the BuilderToolsPlugin. |
| execute0(x, y, z) | boolean | O(1) | Inherited method. Returns true and performs no action, as this operation is executed atomically in the `execute` method, not on a per-block basis. |

## Integration Patterns

### Standard Usage
A developer would not typically instantiate this class directly. It is created and managed internally by the builder tools system. The following example illustrates its conceptual usage within the plugin.

```java
// Hypothetical usage inside a BuilderToolsPlugin network handler

// On receiving a BuilderToolOnUseInteraction packet...
FloodOperation operation = new FloodOperation(entityStoreRef, player, packet, componentAccessor);

// The operation is then executed, often as part of a larger world transaction.
operation.execute(componentAccessor);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new FloodOperation()` in general game logic. Its creation is tightly coupled to the builder tool's network event lifecycle.
- **State Modification:** Do not attempt to modify the internal state of a FloodOperation after it has been created. It is designed as an immutable command.
- **Asynchronous Execution:** Never invoke the `execute` method from an asynchronous task or a different thread. Doing so will lead to world corruption and severe concurrency exceptions.

## Data Pipeline
The FloodOperation serves as a key processing stage in the data flow from player input to world modification.

> Flow:
> Player Input (Client) -> `BuilderToolOnUseInteraction` Packet -> Server Network Layer -> `BuilderToolsPlugin` Packet Handler -> **FloodOperation Instantiation** -> `FloodOperation.execute()` -> `BuilderToolsPlugin.getState().flood()` -> World Edit Session Update -> World Storage Commit

