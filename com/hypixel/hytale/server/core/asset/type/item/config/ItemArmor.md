---
description: Architectural reference for ItemArmor
---

# ItemArmor

**Package:** com.hypixel.hytale.server.core.asset.type.item.config
**Type:** Transient Data Model

## Definition
```java
// Signature
public class ItemArmor implements NetworkSerializable<com.hypixel.hytale.protocol.ItemArmor> {
```

## Architecture & Concepts

The ItemArmor class is a server-side data model that encapsulates all gameplay-related properties of an armor asset. It is not a live game object but rather a static template, deserialized from configuration files at server startup.

Its primary architectural role is to serve as a bridge between human-readable asset definitions and the high-performance data structures required by the core game simulation. For example, asset files may define damage resistance using string identifiers like "fire" or "magic". ItemArmor, during its initialization, resolves these strings into optimized, type-safe representations like the DamageCause enum or integer-indexed maps used by the EntityStatsModule.

This class prominently features a **"Raw vs. Processed"** internal data pattern. Fields suffixed with *Raw* (e.g., damageResistanceValuesRaw) hold the direct, unprocessed data from the asset file. The private `processConfig` method, invoked immediately after deserialization, transforms this raw data into optimized, runtime-ready fields (e.g., damageResistanceValues). All public accessors expose only the final, processed data, ensuring consumers interact with a consistent and efficient representation.

As an implementation of NetworkSerializable, this class is also responsible for converting its state into a network-ready packet format for transmission to game clients.

## Lifecycle & Ownership

-   **Creation:** ItemArmor instances are almost exclusively created by the Hytale asset loading system via the static `CODEC` field. This `BuilderCodec` defines the complex process of deserializing an asset file into an ItemArmor object. The `afterDecode` hook within the codec definition is critical, as it triggers the `processConfig` method to transform the raw data into its final, usable state. Manual instantiation is strongly discouraged.
-   **Scope:** An instance of ItemArmor persists for the entire server session. It is loaded once when assets are parsed and is stored in a central asset registry. It acts as a shared, immutable template for a specific type of armor. It is **not** created on a per-player or per-item-stack basis.
-   **Destruction:** The object is marked for garbage collection only when the server's asset registry is cleared, which typically occurs during server shutdown.

## Internal State & Concurrency

-   **State:** The object's state is highly mutable during the brief `afterDecode` phase of its lifecycle. Once the `processConfig` method completes, the object's state is fixed and should be treated as **effectively immutable**. The public API enforces this by only providing getters.
-   **Thread Safety:** ItemArmor is **not thread-safe** during its creation and initialization. This phase is expected to occur within a single-threaded context during server boot. After initialization, the object is safe for concurrent reads from multiple game threads (e.g., combat calculations for different players wearing the same armor type). This read-only nature after startup obviates the need for explicit locking.

## API Surface

The public API exclusively provides access to the processed, runtime-ready state of the armor configuration.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toPacket() | com.hypixel.hytale.protocol.ItemArmor | O(N) | Serializes the object into a network packet. N is the total number of modifiers. |
| getArmorSlot() | ItemArmorSlot | O(1) | Returns the designated equipment slot for this armor piece. |
| getBaseDamageResistance() | double | O(1) | Returns the base, untyped damage resistance value. |
| getStatModifiers() | Int2ObjectMap | O(1) | Returns the processed map of entity stat modifiers, keyed by stat index. |
| getDamageResistanceValues() | Map | O(1) | Returns the processed map of damage resistances, keyed by DamageCause. |
| getDamageEnhancementValues() | Map | O(1) | Returns the processed map of damage enhancements, keyed by DamageCause. |
| getInteractionModifier(String Key) | Int2ObjectMap | O(1) | Retrieves a map of modifiers for a specific interaction type. |

## Integration Patterns

### Standard Usage

ItemArmor is not meant to be queried directly. It is a component of a larger item asset. Game systems retrieve the item asset first, then access its ItemArmor configuration to apply effects to an entity.

```java
// A game system (e.g., CombatSystem) applies armor stats to an entity.
// Assume 'itemAsset' is the configuration for the equipped item.

ItemArmor armorConfig = itemAsset.getComponent(ItemArmor.class);
if (armorConfig != null) {
    EntityStatsModule stats = entity.getStatsModule();
    stats.addModifiers(armorConfig.getStatModifiers());
    // ... apply other effects like damage resistances
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never use `new ItemArmor()`. The object is invalid without the data and processing provided by the `CODEC` during asset loading.
-   **Accessing Raw Fields:** Do not attempt to access any field ending in *Raw* via reflection. These are internal implementation details of the deserialization pipeline and may not be populated or accurate after the `processConfig` step.
-   **State Mutation:** Do not modify the object's state after it has been loaded. It is shared across the entire server as a static template, and any mutation would have unpredictable global consequences.

## Data Pipeline

The creation and processing of an ItemArmor instance follows a clear, sequential data pipeline.

> Flow:
> Armor Asset File (JSON/Binary) -> Hytale Codec System -> **ItemArmor (Raw State)** -> `processConfig()` -> **ItemArmor (Processed State)** -> Game Systems (Combat, Stats)

For client-server communication, a secondary pipeline is used:

> Flow:
> **ItemArmor (Processed State)** -> `toPacket()` -> `com.hypixel.hytale.protocol.ItemArmor` -> Network Layer -> Game Client

