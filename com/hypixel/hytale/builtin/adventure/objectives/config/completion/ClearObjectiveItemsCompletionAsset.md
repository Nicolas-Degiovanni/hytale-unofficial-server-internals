---
description: Architectural reference for ClearObjectiveItemsCompletionAsset
---

# ClearObjectiveItemsCompletionAsset

**Package:** com.hypixel.hytale.builtin.adventure.objectives.config.completion
**Type:** Data Transfer Object / Asset

## Definition
```java
// Signature
public class ClearObjectiveItemsCompletionAsset extends ObjectiveCompletionAsset {
```

## Architecture & Concepts
The ClearObjectiveItemsCompletionAsset is a stateless, declarative configuration object within the Adventure Mode objective system. It represents a specific type of completion action: clearing all items associated with an objective from a player's inventory upon that objective's successful completion.

This class embodies the **Strategy Pattern** for objective completion logic. It does not contain any executable code for modifying player inventory. Instead, its presence and type within an objective's configuration data signals to a higher-level **ObjectiveCompletionSystem** that the "clear items" strategy should be executed for the player.

Its primary role is to serve as a data-driven marker, allowing game designers to define complex quest behaviors in external configuration files (e.g., JSON) without modifying engine code. The engine's asset loading pipeline uses the associated static CODEC to deserialize these definitions into in-memory asset objects.

## Lifecycle & Ownership
- **Creation:** Instantiated exclusively by the engine's asset loading and deserialization pipeline. The static CODEC field is the entry point for constructing this object from a data file. It is never created programmatically during active gameplay.
- **Scope:** The object is immutable and lives for the entire game session once its containing asset pack is loaded. It is part of the static game data that defines objective behaviors.
- **Destruction:** Marked for garbage collection when the server or client unloads the asset pack it belongs to, typically during a shutdown sequence or a major game state change.

## Internal State & Concurrency
- **State:** Immutable. This class has no internal fields and its state cannot be changed after deserialization. It acts as a pure data marker.
- **Thread Safety:** Fully thread-safe. As an immutable object, it can be safely read and referenced by any system (e.g., game logic thread, asset management thread) without locks or other synchronization primitives.

## API Surface
This asset's contract is defined by its type, not by a traditional method-based API. The engine uses type-checking to identify and act upon it.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| (Implicit Contract) | Type Marker | N/A | The existence of this asset type within an objective's completion configuration signals the engine to execute the item-clearing logic. |
| CODEC | BuilderCodec | N/A | **Engine Internal.** A static definition used by the asset pipeline for serialization. Direct interaction is an anti-pattern. |

## Integration Patterns

### Standard Usage
This class is not used directly in Java code by developers. Instead, it is declared within an objective's JSON or HOCON configuration file. The engine interprets this configuration at runtime.

**Conceptual Example (in a JSON asset file):**
```json
{
  "id": "com.example.adventure.collect_gems",
  "type": "COLLECT_ITEMS",
  "completionActions": [
    {
      "type": "ClearObjectiveItems"
    }
  ]
}
```
In the above example, the `type: "ClearObjectiveItems"` would be deserialized by the engine into a ClearObjectiveItemsCompletionAsset instance.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new ClearObjectiveItemsCompletionAsset()`. These objects must only be created by the asset deserializer to ensure they are correctly integrated into the game's configuration data.
- **Subclassing for Logic:** Do not extend this class to add gameplay logic. It is a pure data container. All corresponding logic must reside in dedicated processing systems that interpret these assets.

## Data Pipeline
The flow of data represented by this asset begins as a text definition and ends as an in-memory object that triggers a game system.

> Flow:
> Objective Definition File (JSON) -> AssetManager Deserializer (using CODEC) -> **ClearObjectiveItemsCompletionAsset** (in-memory) -> ObjectiveCompletionSystem -> Player Inventory Update

