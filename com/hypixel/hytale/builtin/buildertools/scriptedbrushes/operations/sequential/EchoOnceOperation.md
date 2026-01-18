---
description: Architectural reference for EchoOnceOperation
---

# EchoOnceOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential
**Type:** Transient

## Definition
```java
// Signature
public class EchoOnceOperation extends SequenceBrushOperation {
```

## Architecture & Concepts
The EchoOnceOperation is a stateful component within the Scripted Brushes framework, designed to execute a side-effect exactly once during a brush's lifecycle. It functions as a specialized step in a `SequenceBrushOperation` chain, primarily used to provide informational messages or one-time setup feedback to the player using the brush.

Architecturally, it embodies the Command pattern, where an operation is encapsulated as an object. Its key feature is its internal boolean flag, `hasBeenExecuted`, which makes it a single-use command within the context of a loaded brush instance. This prevents the associated action, sending a chat message, from repeating on subsequent applications of the brush.

This class integrates directly with the server's entity and messaging systems. During execution, it retrieves the `PlayerRef` associated with the brush user to dispatch a `Message` to their client. Its configuration, specifically the message content, is managed declaratively and hydrated at runtime by the engine's `Codec` system.

## Lifecycle & Ownership
-   **Creation:** EchoOnceOperation is not instantiated directly via its constructor. It is created by the `BuilderCodec` system during the deserialization of a `BrushConfig` from a data source (e.g., a JSON definition). The static `CODEC` field defines how the engine should build an instance and populate its `messageArg` field from the source data.
-   **Scope:** The object's lifetime is strictly bound to its parent `BrushConfig` instance. It persists in memory as long as the brush script is loaded and active.
-   **Destruction:** The object is eligible for garbage collection when the containing `BrushConfig` is unloaded or replaced. It has no explicit destruction or cleanup logic. Notably, the `resetInternalState` method is implemented as a no-op, which is a deliberate design choice to enforce the "once" behavior across the object's entire lifetime.

## Internal State & Concurrency
-   **State:** This class is mutable and highly stateful. It maintains two private fields:
    -   `messageArg`: A String configured during deserialization.
    -   `hasBeenExecuted`: A boolean flag, initialized to false, that is mutated to true after the first execution. This state is not persisted and resets if the brush is completely reloaded.
-   **Thread Safety:** This class is **not thread-safe**. The `modifyBrushConfig` method performs a non-atomic check-then-act operation on the `hasBeenExecuted` flag. Concurrent invocation on the same instance could result in the message being sent multiple times. However, the Scripted Brush execution model is assumed to be single-threaded, processing operations sequentially within a single game tick, thus mitigating this risk in practice.

    **Warning:** Any attempt to execute brush operations in parallel from a custom system will break the contract of this class.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBrushConfig(...) | void | O(1) | Executes the one-time message send. Mutates internal state to prevent subsequent executions. Asserts that a valid PlayerRef component exists. |
| resetInternalState() | void | O(1) | No-op. Intentionally left empty to preserve the `hasBeenExecuted` state for the object's lifetime. |

## Integration Patterns

### Standard Usage
This operation is not intended to be used programmatically. It should be defined declaratively within a scripted brush data file. The engine's `BrushConfig` loader will then instantiate and manage it.

```json
// Example pseudo-code for a brush definition
{
  "name": "My Informational Brush",
  "operations": [
    {
      "type": "EchoOnceOperation",
      "Message": "Welcome! This brush will now place stone blocks."
    },
    {
      "type": "SetBlockOperation",
      "block": "hytale:stone"
    }
  ]
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not call `new EchoOnceOperation()`. This bypasses the `Codec` system, leaving the `messageArg` field with its default value and breaking the declarative configuration pattern.
-   **External State Mutation:** Do not attempt to externally modify the `hasBeenExecuted` flag to force the message to re-send. This violates the core design contract of the operation. If a resettable echo is needed, a different operation class should be created.

## Data Pipeline
The data flow for this operation involves configuration data being transformed into a server-side action that results in a client-side UI update.

> Flow:
> Brush Definition (JSON) -> `BuilderCodec` Deserializer -> **EchoOnceOperation Instance** -> Brush Execution -> `modifyBrushConfig` Call -> PlayerRef Component -> Server Message System -> Network Packet -> Client Chat UI

