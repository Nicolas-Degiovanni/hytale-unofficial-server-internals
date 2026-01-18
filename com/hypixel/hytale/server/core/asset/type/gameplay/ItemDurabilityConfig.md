---
description: Architectural reference for ItemDurabilityConfig
---

# ItemDurabilityConfig

**Package:** com.hypixel.hytale.server.core.asset.type.gameplay
**Type:** Configuration Model

## Definition
```java
// Signature
public class ItemDurabilityConfig {
```

## Architecture & Concepts
ItemDurabilityConfig is a data-only configuration object that represents the durability parameters for a game item. It is a fundamental component of Hytale's data-driven design philosophy, where game mechanics are defined in external asset files rather than being hardcoded in the engine.

This class does not contain any logic. Its sole purpose is to act as a structured container for data parsed from an asset file. The key architectural feature is the static `CODEC` field. This field makes the class self-describing to the engine's serialization framework, allowing the `AssetManager` to automatically deserialize a corresponding configuration block into a fully-formed Java object. This decouples the asset loading system from the specific data structures it manages, enabling developers and designers to add new configuration parameters by modifying only the asset file and this model class.

This object is typically embedded within a larger asset definition, such as an `Item`, and is accessed by gameplay systems to determine the consequences of an item breaking.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale `BuilderCodec` framework during the asset loading phase. The framework invokes the constructor and populates the fields based on the structure of the source asset file (e.g., a JSON or HOCON file). Direct manual instantiation is an unsupported and dangerous pattern.
- **Scope:** The lifetime of an ItemDurabilityConfig instance is bound to the lifecycle of its parent asset. It persists in memory as long as the corresponding item definition is loaded by the server.
- **Destruction:** The object is eligible for garbage collection when the server unloads or reloads the asset bundle containing the parent item definition. There are no explicit destruction or cleanup methods.

## Internal State & Concurrency
- **State:** The internal state is mutable during deserialization but should be treated as **effectively immutable** post-creation. The `brokenPenalties` field is populated once by the codec system. Subsequent modification is not intended and can lead to unpredictable behavior.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be populated on a single asset-loading thread and then read concurrently by game logic threads. As long as the object is treated as read-only after its initial creation, no concurrency issues will arise.

**WARNING:** Do not modify the state of this object after it has been loaded. All configuration changes must be made in the source asset files and reloaded through the proper engine mechanisms.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getBrokenPenalties() | BrokenPenalties | O(1) | Retrieves the configured penalties for when an item's durability is fully depleted. |

## Integration Patterns

### Standard Usage
This object is not typically retrieved directly. Instead, it is accessed through a parent asset, such as an `ItemDefinition`, by a system responsible for managing item state.

```java
// Hypothetical example within a DurabilitySystem
void applyBrokenState(Item item) {
    ItemDefinition definition = item.getDefinition();
    ItemDurabilityConfig durabilityConfig = definition.getDurabilityConfig();

    if (durabilityConfig != null) {
        BrokenPenalties penalties = durabilityConfig.getBrokenPenalties();
        // ... apply penalties to the player or item
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new ItemDurabilityConfig()`. The object's state will be uninitialized and invalid. It must be created by the asset loader via its `CODEC`.
- **State Mutation:** Do not attempt to modify the `brokenPenalties` field via reflection or other means after the asset has been loaded. This breaks the data-driven contract and will cause inconsistent game state, especially in a multiplayer context.

## Data Pipeline
ItemDurabilityConfig is a terminal node in the asset loading pipeline. It serves as a structured representation of raw configuration data for consumption by server-side gameplay systems.

> Flow:
> Asset File (e.g., `stone_sword.json`) -> Asset Loading Service -> Hytale Codec Framework -> **ItemDurabilityConfig Instance** -> Parent ItemDefinition -> Gameplay Systems

