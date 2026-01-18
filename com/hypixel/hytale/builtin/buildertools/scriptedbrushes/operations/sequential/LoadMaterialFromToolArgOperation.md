---
description: Architectural reference for LoadMaterialFromToolArgOperation
---

# LoadMaterialFromToolArgOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential
**Type:** Transient

## Definition
```java
// Signature
public class LoadMaterialFromToolArgOperation extends SequenceBrushOperation {
```

## Architecture & Concepts
The LoadMaterialFromToolArgOperation is a specialized command object within the Scripted Brush framework. It acts as a bridge, connecting a player's dynamic tool configuration with the execution context of a brush. Its primary function is to read a material, specifically a BlockPattern, from a named argument on the player's currently held Builder Tool and inject it into the active BrushConfig.

This component is fundamental to creating interactive and user-configurable building tools. Instead of a brush having a hardcoded material, this operation allows the material to be selected by the player in-game via the tool's UI or commands. It effectively decouples the brush's logic from its material data, promoting reusable and flexible brush scripts.

Architecturally, it embodies the Command Pattern. It is a single, atomic step in a sequence of operations that collectively define the behavior of a scripted brush. The engine invokes it during the brush's execution cycle, passing it the current state (BrushConfig) to modify.

## Lifecycle & Ownership
- **Creation:** Instances are not created directly via the `new` keyword. They are deserialized and instantiated by the engine's Codec system when a scripted brush asset is loaded. The static `CODEC` field defines the schema for this process, mapping asset data to the `argNameArg` field.
- **Scope:** The object's lifetime is tied to the parent scripted brush asset. It is created once when the brush definition is loaded into memory and persists until that asset is unloaded. The same instance is reused for every execution of the brush.
- **Destruction:** The instance is marked for garbage collection when the server unloads the corresponding brush asset, for example, during a server shutdown or a hot-reload of game assets.

## Internal State & Concurrency
- **State:** The class has one configurable state field, `argNameArg`, which is set at creation time from the asset definition and is treated as immutable thereafter. During execution, the object is stateless; it reads from external game state (Player, BuilderTool, ItemStack) and writes to a passed-in BrushConfig object. It does not maintain any state between invocations.
- **Thread Safety:** This class is **not thread-safe** and is designed for single-threaded access. The `modifyBrushConfig` method directly accesses and manipulates live game state components like Player and EntityStore. It must only be invoked from the main server game thread. Any concurrent access will lead to race conditions, data corruption, and server instability.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBrushConfig(ref, brushConfig, ...) | void | O(1) | Modifies the BrushConfig by loading a BlockPattern from the player's active tool. Sets an error flag on the BrushConfig if the tool, item, argument, or argument type is invalid. |

## Integration Patterns

### Standard Usage
This operation is not intended to be called directly from Java code. It is defined declaratively within a scripted brush asset file (e.g., a JSON or HOCON file). The engine's brush system parses this definition and executes the operation as part of a sequence.

A conceptual asset definition might look like this:
```yaml
# In a hypothetical brush_script.yaml
name: "Dynamic Material Brush"
sequence:
  - type: "LoadMaterialFromToolArgOperation"
    ArgName: "primaryMaterial" # This value is deserialized into argNameArg
  - type: "AnotherBrushOperation"
    # ... uses the material loaded by the previous step
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new LoadMaterialFromToolArgOperation()`. The instance will be unconfigured with an empty `argNameArg`, causing it to fail every time. It must be instantiated by the engine's asset loading system.
- **Stateful Logic:** Do not modify this class to store state between calls. The system assumes each operation is an independent and stateless command.
- **Asynchronous Execution:** Never invoke `modifyBrushConfig` from an asynchronous task or a different thread. It must execute synchronously within the server's main game loop to prevent critical state corruption.

## Data Pipeline
The operation functions as a specific step in a larger data transformation pipeline, moving configuration from a player's tool into the brush's runtime context.

> Flow:
> Player's Builder Tool (with configured `primaryMaterial` argument) -> Player Inventory -> **LoadMaterialFromToolArgOperation** -> Reads `primaryMaterial` value -> Writes BlockPattern to BrushConfig -> Subsequent Brush Operations (e.g., PlaceBlockOperation) consume the pattern from BrushConfig.

