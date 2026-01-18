---
description: Architectural reference for AppendMaskFromToolArgOperation
---

# AppendMaskFromToolArgOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential.masks
**Type:** Transient Data-Driven Component

## Definition
```java
// Signature
public class AppendMaskFromToolArgOperation extends SequenceBrushOperation {
```

## Architecture & Concepts
The AppendMaskFromToolArgOperation is a specialized component within the Scripted Brush framework. It functions as a single, configurable step in a brush's execution sequence. Its primary architectural role is to bridge the gap between a player's dynamic tool configuration and the static definition of a brush script.

This operation dynamically constructs a BlockMask by introspecting the arguments of the player's currently equipped BuilderTool. This allows for the creation of highly versatile brushes that can adapt their behavior based on the material or pattern the player has selected in the tool's user interface. For example, a single "replace" brush script can use this operation to determine which block to replace by reading the "material" argument from the player's active tool.

It effectively decouples the brush's logic from hardcoded block types, promoting reusable and context-aware building tools.

### Lifecycle & Ownership
- **Creation:** Instances are not created directly via the new keyword. They are deserialized and instantiated by the Hytale Codec system when a brush script configuration is loaded by the server. The static CODEC field defines the schema for this deserialization, mapping configuration keys (e.g., ArgName, FilterType) to the class fields.
- **Scope:** The object's lifetime is tied to the parent brush definition. It is effectively a stateless, transient component used during the setup phase of a brush stroke. It exists to execute its logic once per brush application via the modifyBrushConfig method.
- **Destruction:** The object is managed by the Java garbage collector. It is eligible for collection when the brush definition it belongs to is unloaded, for instance, during a script reload or server shutdown. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** The internal state of this operation is defined by its public fields: argNameArg, invertArg, filterTypeArg, and additionalBlocksArg. This state is configured once at creation time by the CODEC and is treated as **immutable** thereafter. The modifyBrushConfig method reads this state but does not alter it.
- **Thread Safety:** This class is **not thread-safe** and is not designed to be. All interactions with this operation are expected to occur on the main server thread as part of the game tick loop. The modifyBrushConfig method accesses and modifies thread-unsafe game state objects like Player and BrushConfig, and its correct execution relies on the single-threaded processing model of the brush system.

## API Surface
The public contract is exclusively for internal engine use, driven by the brush execution system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBrushConfig(ref, brushConfig, ...) | void | O(N) | Reads the configured tool argument from the active player's tool, constructs a BlockMask, and appends it to the current BrushConfig. N is the total number of block names in the tool argument and the additionalBlocksArg string. Throws assertion errors if critical components like the Player are missing. Sets an error flag on the BrushConfig for recoverable failures (e.g., missing tool arg). |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in Java code. It is designed to be configured declaratively within a Scripted Brush definition file (e.g., a JSON file). The engine parses this file and uses the defined operations to build the brush's behavior.

A typical configuration might look like this:

```json
{
  "name": "Operations",
  "value": [
    {
      "type": "AppendMaskFromToolArg",
      "value": {
        "ArgName": "Material",
        "FilterType": "TargetBlock",
        "Invert": false,
        "AdditionalBlocks": "hytale:air"
      }
    }
    // ... other operations
  ]
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new AppendMaskFromToolArgOperation()`. This bypasses the CODEC system, leaving all configuration fields in their default, non-functional state. The object would be useless.
- **Manual Invocation:** Do not call modifyBrushConfig from external systems. This method relies on a very specific context provided by the BrushConfigCommandExecutor, including a valid player entity reference and an active BrushConfig. Calling it outside of this lifecycle will result in unpredictable behavior and likely server errors.
- **State Mutation:** Do not attempt to modify the public fields (e.g., argNameArg) after the object has been instantiated by the codec. The system treats these as immutable configuration values.

## Data Pipeline
This operation processes data by reading from the player's state and writing to the brush's configuration context. The flow is unidirectional and synchronous within a single game tick.

> Flow:
> Player Entity -> Active BuilderTool -> ItemStack ArgData -> **AppendMaskFromToolArgOperation** reads `BlockPattern` -> Parses block names -> Constructs `BlockFilter` and `BlockMask` -> Appends mask to `BrushConfig` for subsequent operations.

