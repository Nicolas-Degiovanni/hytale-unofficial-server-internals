---
description: Architectural reference for DebugBrushOperation
---

# DebugBrushOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.global
**Type:** Transient

## Definition
```java
// Signature
public class DebugBrushOperation extends GlobalBrushOperation {
```

## Architecture & Concepts
The DebugBrushOperation is a meta-operation within the Scripted Brush framework. Unlike most brush operations that directly manipulate world data (e.g., placing blocks, modifying terrain), this class modifies the *execution environment* itself. It acts as a configuration provider for the BrushConfigCommandExecutor, enabling powerful in-game debugging capabilities for complex brush scripts.

Its primary role is to intercept the operation pipeline and inject debugging flags into the executor. This allows content creators to step through their scripts, enable breakpoints, and control logging verbosity without altering the core logic of other operations. This separation of concerns is critical, as it keeps debugging logic isolated from world-modification logic.

The class leverages Hytale's powerful Codec system for deserialization. A static CODEC field defines the schema for this operation, including documentation for each parameter. This allows the engine to automatically parse it from a brush configuration file and instantiate it with user-defined settings.

## Lifecycle & Ownership
- **Creation:** An instance is created exclusively by the engine's Codec deserializer when a brush script containing a "Debug" operation is loaded and executed. It is defined declaratively in a configuration file, not instantiated programmatically.

- **Scope:** Ephemeral. The object's lifetime is confined to a single execution of a brush script. It exists only long enough for its configuration data to be transferred to the BrushConfigCommandExecutor.

- **Destruction:** The object becomes eligible for garbage collection immediately after the BrushConfigCommandExecutor invokes its modifyBrushConfig method. It holds no persistent state and is not referenced after its initial application.

## Internal State & Concurrency
- **State:** The object is a mutable data container for five debugging flags (e.g., printOperations, stepThrough). Its state is populated by the Codec system upon creation and is considered effectively immutable afterward. Any subsequent changes to its fields will have no effect on the system.

- **Thread Safety:** **This class is not thread-safe.** It is designed to be created, used, and discarded entirely within the server's main thread during a single game tick. Concurrent access from other threads is an unsupported and dangerous pattern that will lead to unpredictable behavior in the brush execution environment.

## API Surface
The public API is minimal, exposing only the contract necessary for the BrushConfigCommandExecutor to retrieve its configuration.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBrushConfig(ref, brushConfig, executor, accessor) | void | O(1) | Applies the operation's internal state to the BrushConfigCommandExecutor, configuring its debugging behavior for all subsequent operations in the script. |

## Integration Patterns

### Standard Usage
A developer or content creator does not interact with this class via Java code. Instead, they declare it within a scripted brush configuration file. The system handles its lifecycle automatically.

```yaml
# Conceptual example of a scripted brush definition
# The engine parses this and creates a DebugBrushOperation instance.

operations: [
    {
        type: "Debug",
        StepThrough: true,
        PrintOperations: true,
        OutputTarget: "Console"
    },
    {
        type: "SetBlock",
        block: "hytale:stone"
    },
    # ... other operations that will now be affected by the debug settings
]
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new DebugBrushOperation()`. The object is meaningless outside the context of the Codec-driven execution pipeline. The system must be responsible for its creation to ensure proper initialization.

- **State Mutation After Creation:** Do not attempt to get a reference to this object and modify its fields after it has been processed. The configuration is applied once and is not re-read.

- **Asynchronous Execution:** Do not pass this object or the executor it configures to another thread. The entire brush execution system is synchronous with the server tick.

## Data Pipeline
The DebugBrushOperation acts as a transient data carrier that configures a stateful executor.

> Flow:
> Brush Script File -> Engine Codec Deserializer -> **DebugBrushOperation Instance** -> BrushConfigCommandExecutor::modifyBrushConfig -> Executor Internal State Update -> Execution of Subsequent Operations

