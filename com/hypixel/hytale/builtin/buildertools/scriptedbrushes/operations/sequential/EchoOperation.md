---
description: Architectural reference for EchoOperation
---

# EchoOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential
**Type:** Transient Command Object

## Definition
```java
// Signature
public class EchoOperation extends SequenceBrushOperation {
```

## Architecture & Concepts
The EchoOperation class is a concrete implementation of the Command design pattern, representing a single, serializable action within the Scripted Brush toolset. It serves as a fundamental building block for creating complex brush behaviors by providing a simple, user-facing feedback mechanism.

Its primary role is to send a pre-configured text message to the chat of the player executing the brush operation. This is typically used for debugging brush sequences or providing explicit notifications to the user at specific points in a multi-stage operation.

Architecturally, EchoOperation is designed to be data-driven. The static CODEC field is the most critical feature, integrating the class with Hytale's serialization framework. This allows instances to be defined declaratively in external configuration files (e.g., JSON) and instantiated at runtime by the brush system. The operation is executed entirely on the server, interacting with the Entity-Component System (ECS) to locate and message the correct player entity.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale `Codec` deserialization engine. The static `CODEC` field provides the blueprint for constructing an EchoOperation from a data source, such as a brush configuration file loaded by the server. Manual instantiation is an anti-pattern.
- **Scope:** The lifetime of an EchoOperation instance is strictly bound to its parent `BrushConfig` object. It exists in memory only as long as the brush definition is loaded. It should be treated as an immutable value object within that configuration.
- **Destruction:** The object is marked for garbage collection when its parent `BrushConfig` is unloaded or otherwise dereferenced. There are no explicit cleanup or `close` methods.

## Internal State & Concurrency
- **State:** The class holds a single, mutable state field: `messageArg`. This string is populated once during deserialization via the `CODEC`. After initialization, it is considered *effectively immutable*, and any external modification will lead to undefined behavior.
- **Thread Safety:** This class is not thread-safe and must not be accessed from multiple threads. The Scripted Brush system guarantees that the `modifyBrushConfig` method is invoked synchronously within the server's main game loop (tick). All interactions with an EchoOperation instance must be confined to this thread.

## API Surface
The public API is minimal, consisting only of the inherited execution method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBrushConfig(...) | void | O(1) | Executes the command. Retrieves the PlayerRef component for the executing entity and dispatches the configured message to their chat. Throws an assertion error if the target entity does not have a PlayerRef component. |

## Integration Patterns

### Standard Usage
EchoOperation is not intended to be used directly in Java code. Instead, it is defined declaratively as part of a brush configuration file. The server's brush system parses this file and executes the operation as part of a sequence.

```json
// Example: brush_definition.json
{
  "name": "Debug Brush",
  "operations": [
    {
      "type": "hytale:echo",
      "Message": "Starting brush operation..."
    },
    {
      "type": "hytale:set_block",
      "block": "hytale:stone"
    },
    {
      "type": "hytale:echo",
      "Message": "Operation complete!"
    }
  ]
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new EchoOperation()`. The system relies on the `CODEC` to correctly populate the object from a data source. Direct instantiation will result in an object with a default, unconfigured message.
- **State Mutation:** Do not attempt to modify the `messageArg` field via reflection or other means after the brush has been loaded. This violates the data-driven design and can cause inconsistent behavior.
- **Client-Side Execution:** This is a server-side-only operation. It depends on server-core components like `PlayerRef` and `EntityStore` and will fail if executed in a client context.

## Data Pipeline
The flow of data for this component begins with a static configuration file and ends with a UI update on the client.

> Flow:
> Brush Configuration File -> Server Codec Parser -> **EchoOperation Instance** -> Scripted Brush Executor -> `modifyBrushConfig` -> Server Message System -> Network Packet -> Client Chat UI

