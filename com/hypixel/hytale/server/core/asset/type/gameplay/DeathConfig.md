---
description: Architectural reference for DeathConfig
---

# DeathConfig

**Package:** com.hypixel.hytale.server.core.asset.type.gameplay
**Type:** Configuration Model

## Definition
```java
// Signature
public class DeathConfig {
```

## Architecture & Concepts

The DeathConfig class is a data-driven configuration model that defines the rules and consequences of entity death within a game world. It is not a service or a manager; rather, it is a passive data structure that represents a deserialized game asset.

Its primary architectural role is to decouple gameplay logic from hard-coded values. Instead of a developer writing `if (playerDies) drop(allItems)`, the logic queries the active DeathConfig for the appropriate rules. This allows game designers to modify death penalties, item loss percentages, and respawn behaviors by simply editing asset files, without requiring code changes.

The static **CODEC** field is the cornerstone of this class. It is a Hytale `BuilderCodec` that acts as a contract for serialization and deserialization. It defines how raw data from an asset file (e.g., JSON or HOCON) is mapped to the fields of a DeathConfig instance, including type conversion, validation, and documentation. The use of `appendInherited` within the codec signifies that DeathConfig participates in a hierarchical asset system, where specific configurations can inherit and override values from a more generic parent configuration.

### Lifecycle & Ownership
- **Creation:** A DeathConfig instance is never created directly using its constructor. It is instantiated exclusively by the Hytale Asset Loading framework during server or world initialization. The framework uses the static `CODEC` to parse a corresponding asset file and construct the object.
- **Scope:** The lifetime of an instance is bound to the asset it represents. A global death configuration will persist for the entire server session. A zone-specific configuration will be loaded when its zone becomes active and unloaded when it is no longer needed.
- **Destruction:** The object is managed by the Java garbage collector. It is eligible for destruction once the Asset Manager releases all references to it, typically upon server shutdown or when an asset scope is unloaded.

## Internal State & Concurrency
- **State:** DeathConfig is **effectively immutable**. Its state is populated once during deserialization by the `CODEC`. While its fields are not marked as final, there is no public API to modify them post-creation. It holds static configuration data, not dynamic runtime state.
- **Thread Safety:** The class is **thread-safe for reads**. As its internal state does not change after initialization, multiple gameplay systems can safely access its configuration properties from different threads without external synchronization. Any attempt to mutate its state via reflection or other means is a severe anti-pattern and will lead to unpredictable server behavior.

## API Surface

The public API is limited to read-only accessors for its configured properties.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getRespawnController() | RespawnController | O(1) | Retrieves the controller responsible for determining an entity's respawn location. |
| getItemsLossMode() | ItemsLossMode | O(1) | Returns the configured mode for handling item loss upon death. |
| getItemsAmountLossPercentage() | double | O(1) | Returns the percentage of items lost, applicable only when ItemsLossMode is CONFIGURED. |
| getItemsDurabilityLossPercentage() | double | O(1) | Returns the percentage of durability lost from all items in an entity's inventory upon death. |

## Integration Patterns

### Standard Usage

Gameplay systems should retrieve the active DeathConfig from a context-aware service or asset provider. The configuration is then used to direct the death logic.

```java
// Example from a hypothetical PlayerDeathSystem
void handlePlayerDeath(Player player) {
    // Retrieve the correct config for the player's context (e.g., current zone)
    DeathConfig activeConfig = assetService.get(DeathConfig.class, player.getZoneId());

    // Use the config to apply rules
    ItemsLossMode mode = activeConfig.getItemsLossMode();
    if (mode == ItemsLossMode.ALL) {
        player.inventory().dropAll();
    } else if (mode == ItemsLossMode.CONFIGURED) {
        double lossPercentage = activeConfig.getItemsAmountLossPercentage();
        player.inventory().dropItemsByPercentage(lossPercentage);
    }

    // ... other logic for durability loss and respawning
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new DeathConfig()`. This bypasses the entire asset loading pipeline, including data validation and inheritance. The resulting object will contain only default values and will not reflect the intended game design.
- **State Mutation:** Do not attempt to modify the fields of a DeathConfig object after it has been loaded. This violates its immutability contract and can cause inconsistent behavior across different systems that share the same configuration instance.

## Data Pipeline

The DeathConfig class is the terminal point of an asset loading pipeline. It translates raw data into a type-safe, in-memory representation for consumption by server logic.

> Flow:
> Asset File (e.g., world/zone.json) -> Server AssetManager -> **DeathConfig.CODEC** -> **DeathConfig Instance** -> Gameplay System (e.g., Death Processor)

