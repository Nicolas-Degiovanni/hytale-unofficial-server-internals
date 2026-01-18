---
description: Architectural reference for ReplaceOperation
---

# ReplaceOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential
**Type:** Transient Operation Component

## Definition
```java
// Signature
public class ReplaceOperation extends SequenceBrushOperation {
```

## Architecture & Concepts

The ReplaceOperation is a fundamental, data-driven component within the Scripted Brush system. It is not a standalone service but rather a single, configurable step in a sequence of world modification actions. Its architectural purpose is to perform a highly specific, conditional replacement of one block type with a potentially complex block pattern.

Architecturally, this class embodies the Command pattern. Each instance is a reified command containing both the logic (the `modifyBlocks` method) and the data (the `blockTypeKeyToReplace` and `replacementBlocks` fields) required for its execution.

The most critical design aspect is its integration with the Hytale **Codec** system. The static `CODEC` field makes the class self-describing, allowing the game engine to dynamically instantiate and configure it from asset files (e.g., JSON or a similar format) without hardcoded factories. This decouples the operation's logic from its configuration, enabling content creators to define complex brush behaviors entirely through data.

It operates on a transactional buffer, the `BrushConfigEditStore`, ensuring that its modifications are not applied directly to the world but are staged as part of a larger brush application.

### Lifecycle & Ownership
-   **Creation:** An instance of ReplaceOperation is not created directly via its constructor. It is deserialized and instantiated by the Hytale `Codec` system when a parent Scripted Brush asset is loaded into memory. The static `CODEC` field serves as the factory.
-   **Scope:** The object's lifetime is bound to its parent `BrushConfig`. It persists as a configured component of the brush for as long as that brush is active. It holds no state between individual brush applications.
-   **Destruction:** The instance is marked for garbage collection when its parent `BrushConfig` is unloaded or reconfigured, such as when a player selects a different brush. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
-   **State:** The internal state consists of the `blockTypeKeyToReplace` and `replacementBlocks` fields. This state is **mutable**, but it is designed to be configured once at load time by the Codec system. It should be treated as read-only during execution. The class holds no session-specific or world-related state.

-   **Thread Safety:** This class is **not thread-safe**. The Scripted Brush execution engine guarantees that all operations for a single brush stroke are executed serially on a single, designated world-processing thread. Concurrently modifying its configuration fields from another thread during a `modifyBlocks` call will lead to undefined behavior and world corruption.

## API Surface

The public API is minimal, designed to be invoked exclusively by the Scripted Brush engine.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBlocks(...) | boolean | O(1) | Core execution method. Called for each block in the brush's volume. Checks the existing block and, if it matches, stages a replacement in the `BrushConfigEditStore`. |
| modifyBrushConfig(...) | void | O(1) | No-op. This operation does not alter the parent brush's configuration during execution. |

## Integration Patterns

### Standard Usage

This class is not used directly in procedural code. It is defined declaratively within a brush asset file. The engine then loads this configuration to build the brush's behavior.

A conceptual brush definition might look like this:

```yaml
# Example: my_cool_brush.hbrush (Conceptual)
name: "Stone to Grass Replacer"
operations:
  - type: "ReplaceOperation"
    FromBlockType: "Rock_Stone"
    ToBlockPattern: "Grass"
  - type: "AnotherOperation"
    # ... other operation configs
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new ReplaceOperation()`. This bypasses the Codec system and results in a default, unconfigured component that will likely replace stone with stone, which is useless. Operations must be configured via data assets.
-   **External Invocation:** Do not call `modifyBlocks` from outside the Scripted Brush engine. The method relies on a very specific context, including a valid `BrushConfigEditStore`, which is only provided by the engine during a brush application cycle.
-   **Runtime State Mutation:** Modifying the `blockTypeKeyToReplace` or `replacementBlocks` fields after the brush has been loaded is an unsupported and dangerous pattern. It can lead to inconsistent brush behavior within a single game session.

## Data Pipeline

The ReplaceOperation functions as a transformation step within the broader Brush system data flow. It receives coordinates and reads from a world data buffer, conditionally writing its changes back to that same buffer.

> Flow:
> Brush Asset File -> **Codec Deserializer** -> In-Memory `ReplaceOperation` Instance -> Scripted Brush Engine -> `modifyBlocks(x,y,z)` -> `BrushConfigEditStore` (Staging Buffer) -> World Storage Commit

