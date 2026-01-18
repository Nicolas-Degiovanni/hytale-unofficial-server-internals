---
description: Architectural reference for CombatConfig
---

# CombatConfig

**Package:** com.hypixel.hytale.server.core.asset.type.gameplay
**Type:** Configuration Asset

## Definition
```java
// Signature
public class CombatConfig {
```

## Architecture & Concepts
The CombatConfig class is a passive data structure that represents a deserialized combat configuration asset. It is a cornerstone of Hytale's data-driven design, allowing game designers to define and tune core combat mechanics through external asset files (e.g., JSON) without modifying or recompiling server code.

The central architectural feature of this class is the static **CODEC** field. This `BuilderCodec` instance defines the serialization contract, specifying how raw asset data is mapped to the fields of a CombatConfig object. It handles data type conversion, validation, and crucially, property inheritance from parent configurations. This allows for a hierarchical asset system where specific combat profiles can override a base set of rules.

This class does not contain any logic. Its sole responsibility is to hold validated configuration values that are consumed by active game systems, such as a `DamageSystem` or `EntityHealthManager`, to enforce combat rules during gameplay.

## Lifecycle & Ownership
-   **Creation:** CombatConfig instances are **never** created directly using the constructor. They are instantiated exclusively by the server's asset loading framework during the deserialization process. The framework reads an asset file from disk and uses the static CODEC to build the corresponding in-memory CombatConfig object.

-   **Scope:** An instance of CombatConfig is typically loaded at server startup and persists for the entire server session. These objects are cached by the asset management system and are effectively singletons relative to their source asset file.

-   **Destruction:** The object is eligible for garbage collection only when the server's asset cache is cleared or reloaded, which typically occurs during a server shutdown or a full asset hot-reload event.

## Internal State & Concurrency
-   **State:** The object is **effectively immutable** post-creation. While its fields are not marked as final, there is no public API to modify its state after it has been deserialized. All properties are populated once by the CODEC. The `staminaBrokenEffectIndex` field is a derived value, calculated and cached during the `afterDecode` lifecycle hook for improved lookup performance.

-   **Thread Safety:** CombatConfig is **thread-safe for reads**. Because its state is fixed after initialization, multiple game threads (e.g., world simulation threads) can safely access its configuration properties concurrently without locks or other synchronization primitives. The asset loading process that creates these objects is responsible for ensuring safe publication.

## API Surface
The public API consists entirely of non-blocking, O(1) accessor methods for retrieving configuration values.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getOutOfCombatDelay() | Duration | O(1) | Returns the configured time an entity must be out of combat before state changes. |
| getStaminaBrokenEffectIndex() | int | O(1) | Returns the pre-calculated asset index for the stamina broken effect. |
| isDisplayHealthBars() | boolean | O(1) | Returns true if the server should suggest displaying health bars. |
| isDisplayCombatText() | boolean | O(1) | Returns true if the server should suggest displaying damage numbers. |
| isNpcIncomingDamageDisabled() | boolean | O(1) | Returns true if all incoming damage to NPCs should be ignored. |
| isPlayerIncomingDamageDisabled() | boolean | O(1) | Returns true if all incoming damage to players should be ignored. |

## Integration Patterns

### Standard Usage
Game logic systems should retrieve the relevant CombatConfig instance from the appropriate context or asset registry. Never cache the values locally; always read directly from the config object to respect potential asset hot-reloading.

```java
// Example within a hypothetical DamageProcessingSystem
public void applyDamage(Entity target, DamageSource source) {
    // Retrieve the active combat configuration for the current context
    CombatConfig config = world.getActiveCombatConfig();

    if (target.isNpc() && config.isNpcIncomingDamageDisabled()) {
        return; // Abort damage calculation
    }

    // ... proceed with damage logic
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new CombatConfig()`. This bypasses the entire asset loading and data-driven configuration system. The resulting object will contain only default, hardcoded values and will not reflect the actual game design specified in asset files. This is a critical error that can lead to subtle and difficult-to-diagnose bugs.

-   **State Modification:** Do not use reflection or other means to modify the fields of a CombatConfig instance after it has been loaded. The server architecture assumes these objects are immutable, and changing their state at runtime will lead to unpredictable behavior and race conditions.

## Data Pipeline
The CombatConfig class is the terminal point of the asset loading pipeline and the starting point for consumption by game logic.

> Flow:
> Asset File on Disk (e.g., `default_combat.json`) -> Server Asset Loader -> **CombatConfig.CODEC** (Deserialization & Validation) -> In-Memory **CombatConfig** Instance -> Game Systems (e.g., DamageSystem)

