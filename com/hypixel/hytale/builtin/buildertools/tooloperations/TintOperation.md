---
description: Architectural reference for TintOperation
---

# TintOperation

**Package:** com.hypixel.hytale.builtin.buildertools.tooloperations
**Type:** Transient (Command Object)

## Definition
```java
// Signature
public class TintOperation extends ToolOperation {
```

## Architecture & Concepts
The TintOperation class is a concrete implementation of the Command design pattern. It encapsulates a single, atomic world-modification action: applying a color tint to a region of the game world. Its primary role is to translate the data from a network packet, specifically a BuilderToolOnUseInteraction, into a direct, executable call on the server's world state.

This class acts as a bridge between the network protocol layer and the core world-editing logic. By encapsulating the operation's parameters (color, position, shape) and logic, it decouples the packet-handling system from the specifics of how a tint is applied. The system that processes player tool usage creates an instance of TintOperation, populates it with context from the player and the network packet, and then invokes its `execute` method to apply the change.

The constructor is a critical part of its design, performing immediate data validation and transformation. It parses a hexadecimal color string into an integer representation, failing fast and notifying the player if the format is invalid. This ensures that by the time `execute` is called, the operation is guaranteed to have valid parameters.

### Lifecycle & Ownership
- **Creation:** Instantiated by a server-side builder tool handler when it processes a `BuilderToolOnUseInteraction` packet corresponding to a tint action. It is created on-demand for each distinct use of the tool.
- **Scope:** Ephemeral. The object's lifetime is extremely short, confined to the scope of a single tool-use event within a single server tick.
- **Destruction:** The object holds no persistent state or external references and is eligible for garbage collection immediately after its `execute` method completes.

## Internal State & Concurrency
- **State:** The internal state of a TintOperation instance is immutable after construction. Its core state, the `tintColor`, is stored in a final field. All other operational parameters like coordinates and shape are inherited from the `ToolOperation` base class and are also fixed during instantiation. The object is effectively a read-only container for the parameters of a single command.
- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed to be created, used, and discarded within a single, sequential execution context, such as the main server world thread. Concurrent calls to `execute` would lead to unpredictable world state corruption.

## API Surface
The public API is minimal, exposing only the command execution entry point.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(accessor) | void | O(N) | Executes the tint operation. Delegates to the BuilderState to modify the world. Complexity is proportional to the number of blocks in the target shape. |
| execute0(x, y, z) | boolean | O(1) | Inherited from ToolOperation. This is a no-op for this class, as the entire operation is performed atomically within the main execute method. |

## Integration Patterns

### Standard Usage
A TintOperation is not intended to be used directly by most systems. It is created and executed by the server's internal builder tool framework in response to player input.

```java
// Hypothetical usage within a tool handling system
void handleToolUse(Player player, BuilderToolOnUseInteraction packet) {
    // The framework determines the operation type is "Tint"
    Ref<EntityStore> storeRef = ...;
    ComponentAccessor<EntityStore> accessor = ...;

    try {
        ToolOperation operation = new TintOperation(storeRef, player, packet, accessor);
        operation.execute(accessor);
    } catch (NumberFormatException e) {
        // The constructor has already notified the player. Log the error.
        log.warn("Failed to create TintOperation due to invalid color", e);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **State Re-use:** Do not hold a reference to a TintOperation instance after its `execute` method has been called. It is a one-shot object and is not designed for re-use.
- **Manual Instantiation:** Avoid creating a TintOperation manually. It should only be constructed by the designated framework that handles builder tool packets to ensure all context (Player, Packet, Accessors) is correctly supplied.
- **External State Modification:** Do not attempt to modify the state of a TintOperation via reflection or other means after it has been constructed. Its immutability is key to ensuring predictable behavior.

## Data Pipeline
The flow of data for a tinting action begins with a client packet and ends with a modification to the server's world data. The TintOperation is a critical processing step in this pipeline.

> Flow:
> BuilderToolOnUseInteraction Packet -> Server Network Handler -> Builder Tool Framework -> **TintOperation** (Instantiation & Validation) -> BuilderState.tint() -> World Data Modification

