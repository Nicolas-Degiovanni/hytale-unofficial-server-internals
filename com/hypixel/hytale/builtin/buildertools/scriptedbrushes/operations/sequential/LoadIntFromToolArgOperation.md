---
description: Architectural reference for LoadIntFromToolArgOperation
---

# LoadIntFromToolArgOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential
**Type:** Transient / Data-Driven Operation

## Definition
```java
// Signature
public class LoadIntFromToolArgOperation extends SequenceBrushOperation {
```

## Architecture & Concepts
The LoadIntFromToolArgOperation is a specific, data-driven command within the Scripted Brush framework. It serves as a critical bridge between a player's persistent tool configuration and the transient state of a brush during its execution.

Its primary architectural function is to dynamically parameterize a brush's behavior based on arguments stored on the player's active BuilderTool ItemStack. For example, a player might have a "sculpting" tool with a configurable "size" argument. This operation reads that "size" value at runtime and injects it into the BrushConfig's width or height field, allowing a single scripted brush to behave differently based on player-defined settings.

This class is not intended for direct instantiation in game logic. Instead, it is defined declaratively within a brush's asset files. The static CODEC field is the key to this system, providing the deserialization logic that transforms a configuration block into a live operation instance. This pattern makes the brush system highly extensible, allowing designers to compose complex behaviors without writing new Java code.

## Lifecycle & Ownership
-   **Creation:** Instances are created exclusively by the Hytale Codec system during the loading and parsing of a Builder Tool asset file. The static CODEC member defines the mapping between the configuration format and the object's fields.
-   **Scope:** The lifetime of an instance is tied to the loaded brush asset definition. It is effectively a stateless configuration object that is executed when a brush is used. It persists in memory as part of the brush definition but holds no per-player or per-use state.
-   **Destruction:** The object is eligible for garbage collection when the parent brush asset is unloaded from memory.

## Internal State & Concurrency
-   **State:** The internal state consists of its configuration fields: argNameArg, targetFieldArg, relativeArg, and negateArg. This state is configured once at creation by the Codec system and should be considered **immutable** thereafter. The class is stateless with respect to its execution; it reads its own configuration to modify an *external* BrushConfig object passed to it.
-   **Thread Safety:** This class is not thread-safe by design and contains no internal synchronization mechanisms. The modifyBrushConfig method performs direct lookups on entity components and modifies a mutable BrushConfig object. It is strictly intended for execution on the main server thread during a player-initiated action.

    **WARNING:** Calling modifyBrushConfig from a worker thread will lead to race conditions, data corruption, and server instability. All brush operations must be synchronized with the main game loop.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBrushConfig(...) | void | O(1) | Reads an integer from the active BuilderTool's arguments and applies it to the BrushConfig. Sets an error flag on the BrushConfig if the tool, item, or argument is missing or of an incorrect type. |

## Integration Patterns

### Standard Usage
This class is not invoked directly by developers. It is configured within a tool's asset definition and executed by the brush system. The engine iterates through a list of SequenceBrushOperation objects and calls modifyBrushConfig on each one to progressively build the final brush parameters.

A conceptual view of the engine's usage:
```java
// Engine-level code (conceptual)
BrushConfig config = new BrushConfig();
Player player = ...; // The player using the brush

// The engine iterates over operations defined in the tool's asset file
for (SequenceBrushOperation operation : brushAsset.getOperations()) {
    // This call executes the logic of LoadIntFromToolArgOperation
    operation.modifyBrushConfig(player.getRef(), config, ...);

    if (config.hasError()) {
        // Halt execution and report error to player
        break;
    }
}

// The now-modified 'config' is used for world edits
performWorldEdit(player, config);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new LoadIntFromToolArgOperation()`. The object's fields will be uninitialized, and it bypasses the entire data-driven asset pipeline. All operations must be defined in configuration and loaded via the CODEC.
-   **State Mutation:** Do not modify the public fields (argNameArg, etc.) at runtime after the object has been deserialized. This can lead to unpredictable behavior for a shared brush asset. Treat the instance as read-only.
-   **Incorrect Context:** Do not call modifyBrushConfig without a valid Player context. The method asserts that a Player component is present and will fail if it is not found. It is fundamentally tied to a player action.

## Data Pipeline
The operation facilitates a data flow from persistent item data to the transient execution state of a brush.

> Flow:
> ItemStack NBT Data (Tool Args) -> Player Component -> **LoadIntFromToolArgOperation.modifyBrushConfig** -> Mutable BrushConfig State -> Subsequent World Modification Operations

