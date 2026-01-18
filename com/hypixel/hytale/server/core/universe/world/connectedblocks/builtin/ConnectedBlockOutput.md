---
description: Architectural reference for ConnectedBlockOutput
---

# ConnectedBlockOutput

**Package:** com.hypixel.hytale.server.core.universe.world.connectedblocks.builtin
**Type:** Transient Data Object

## Definition
```java
// Signature
public class ConnectedBlockOutput {
```

## Architecture & Concepts
The ConnectedBlockOutput class is a data structure that represents the declarative outcome of a connected block rule. It is not a service or manager, but rather a configuration object that defines which block should be placed in the world when its corresponding rule is successfully evaluated.

This class acts as a critical bridge between the declarative asset system and the imperative world update logic. Its primary role is to translate a high-level, human-readable block definition (e.g., "oak_log" with state "axis=y") into a low-level, performance-optimized integer index used by the world renderer and game logic.

Instances of this class are created by the asset loading system using the provided static CODEC, which deserializes definitions from configuration files. This design allows game designers to specify complex block connection behaviors without modifying engine code.

## Lifecycle & Ownership
- **Creation:** Instantiated exclusively by the Hytale `Codec` system during the server's asset loading phase. The protected constructor prevents direct instantiation. It is typically defined as part of a larger connected block rule asset.
- **Scope:** The object's lifetime is bound to the connected block rule that contains it. It is loaded at server startup and persists in memory for the entire server session.
- **Destruction:** De-referenced and garbage collected when the server shuts down and all asset registries are cleared.

## Internal State & Concurrency
- **State:** **Mutable**. The internal state, particularly the blockTypeKey field, is modified by the resolve method upon its first successful execution. This behavior serves as a form of lazy resolution and memoization, caching the final block key string.

- **Thread Safety:** **This class is not thread-safe.** The mutation of internal state within the resolve method introduces a severe risk of race conditions if a single instance is accessed by multiple threads concurrently.

    **Warning:** All operations on a ConnectedBlockOutput instance must be performed within a single, synchronized context, such as the main server thread or a dedicated, single-threaded world generation worker.

## API Surface
The public contract is minimal, focused entirely on the resolution of the block definition.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| resolve(baseBlockType, assetMap) | int | O(1) | Resolves the configured block and state against the asset map to produce a final integer block index. Returns -1 if resolution fails. **Warning:** This method mutates the object's internal state upon success. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by most developers. It is an internal component of the connected blocks system. A higher-level rule evaluator uses it to determine the final block ID after a rule has been matched.

```java
// Hypothetical usage within a rule evaluation system
ConnectedBlockOutput output = matchedRule.getOutput();
BlockType baseType = world.getBlockTypeAt(pos);
BlockTypeAssetMap assetMap = server.getAssetMap(BlockType.class);

// Resolve the declarative output into a concrete block index
int finalBlockIndex = output.resolve(baseType, assetMap);

if (finalBlockIndex != -1) {
    world.setBlockIndexAt(pos, finalBlockIndex);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to create instances via reflection. Objects must be defined in asset files and loaded via the engine's CODEC system.
- **Concurrent Resolution:** Do not call the resolve method on the same instance from multiple threads. This will lead to unpredictable behavior and data corruption due to internal state mutation.
- **State Re-evaluation:** The resolve method is not idempotent if it succeeds, as it caches its result. Do not rely on it to re-evaluate logic based on external changes after a successful first call.

## Data Pipeline
ConnectedBlockOutput sits at the end of the rule evaluation pipeline, acting as the final translation step before a world modification is committed.

> Flow:
> Block Update Event -> Connected Block System -> Rule Matching Logic -> **ConnectedBlockOutput.resolve()** -> Final Block Index -> World Chunk Data Update

