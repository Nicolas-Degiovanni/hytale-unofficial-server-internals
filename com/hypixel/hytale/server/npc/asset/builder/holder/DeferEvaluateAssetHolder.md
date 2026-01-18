---
description: Architectural reference for DeferEvaluateAssetHolder
---

# DeferEvaluateAssetHolder

**Package:** com.hypixel.hytale.server.npc.asset.builder.holder
**Type:** Transient Data Model

## Definition
```java
// Signature
public class DeferEvaluateAssetHolder extends AssetHolder {
```

## Architecture & Concepts
The DeferEvaluateAssetHolder is a concrete implementation of the Strategy pattern, specializing the abstract AssetHolder. Its sole purpose is to act as a flag or marker within the NPC asset building system, signaling that the associated asset is not a static, predefined value. Instead, it represents a placeholder for an asset that must be resolved dynamically at runtime.

This component is critical for creating NPCs with variable appearances or properties based on in-game conditions, such as time of day, biome, or NPC state. It decouples the NPC definition from fixed asset paths, allowing designers to specify logic-driven asset selection.

For example, an NPC definition might specify a texture that changes based on whether it is day or night. The asset parser would create a DeferEvaluateAssetHolder to represent this conditional texture. At runtime, the asset resolution system will identify this holder via its `isStatic` method and invoke a separate evaluation pipeline to determine the correct texture to load.

## Lifecycle & Ownership
- **Creation:** Instantiated exclusively by the internal NPC asset definition parser when it encounters a dynamic or conditional asset reference within a configuration file (e.g., a HOCON or JSON file). It is never created directly by game systems or gameplay programmers.
- **Scope:** This object is short-lived and exists only during the asset pre-processing and building phase for a specific NPC type. Its lifetime is bound to the builder object that creates it.
- **Destruction:** It is marked for garbage collection immediately after the NPC's asset blueprint is finalized. It does not persist as a runtime object within a spawned NPC instance.

## Internal State & Concurrency
- **State:** This class is stateless. Its behavior is defined by its type, not by any internal fields. It inherits state from AssetHolder but adds none of its own. It can be considered immutable.
- **Thread Safety:** Inherently thread-safe. As a stateless and immutable object, it can be safely read by multiple threads during the asset loading process without requiring any synchronization or locks.

## API Surface
The public contract is minimal and consists of a single overridden method that defines its entire behavior.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isStatic() | boolean | O(1) | Signals that the held asset is dynamic. This implementation always returns false, differentiating it from a static holder. |

## Integration Patterns

### Standard Usage
This class is an internal implementation detail of the asset system and is not intended for direct use. The following example illustrates its conceptual creation within the asset pipeline.

```java
// PSEUDOCODE: Internal usage within an asset parser
// When the parser finds a dynamic value like "skin": "eval(some_logic)"
// it creates this holder to represent the unresolved asset.

AssetHolder dynamicHolder = new DeferEvaluateAssetHolder();
// The holder is then associated with a property in the NPC blueprint.
npcBlueprint.addAsset("skin", dynamicHolder);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never instantiate this class in gameplay code. The asset system relies on specific holder types being created by the correct parsing logic. Manually creating one can lead to unpredictable asset resolution failures.
- **Wrapping Static Assets:** Do not use this class to hold a known, static asset path. Doing so forces the system to engage the more expensive dynamic evaluation pipeline for no reason, introducing unnecessary performance overhead. Use a `StaticAssetHolder` for this purpose.

## Data Pipeline
This holder acts as a routing instruction within the asset resolution data flow, diverting dynamic assets to a specialized evaluation path.

> Flow:
> NPC Definition File -> Asset Parser -> **DeferEvaluateAssetHolder** (Instantiation) -> NPC Blueprint -> Runtime Asset Resolver -> Dynamic Logic Evaluator -> Final Asset Path -> AssetManager

