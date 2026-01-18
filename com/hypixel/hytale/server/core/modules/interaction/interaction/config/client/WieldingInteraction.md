---
description: Architectural reference for WieldingInteraction
---

# WieldingInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.client
**Type:** Configuration Object

## Definition
```java
// Signature
public class WieldingInteraction extends ChargingInteraction {
```

## Architecture & Concepts
The WieldingInteraction class defines the server-side behavior for continuous, stateful actions such as blocking with a shield or parrying. It is a specialized type of ChargingInteraction, designed to be held indefinitely by a player or NPC until an action is cancelled, stamina is depleted, or the interaction is otherwise interrupted.

This class is not a service or a manager; it is a data-centric configuration object loaded from game assets. Each instance represents a specific "wielding" move defined in JSON files, containing all the logic and data needed to modify an entity's defensive properties while the interaction is active.

Its primary architectural role is to act as a temporary state container that attaches to an entity's DamageDataComponent. When an entity initiates a wielding action, a reference to the corresponding WieldingInteraction instance is stored in its DamageDataComponent via the `setCurrentWielding` method. The core damage calculation pipeline then queries this component on the defending entity. If an active WieldingInteraction is present, its configured damage and knockback modifiers are applied to the incoming attack, effectively reducing or altering the damage taken.

The class also manages stamina consumption, defines effects to be triggered upon a successful block, and determines subsequent interactions, making it a critical node in the server's combat state machine.

## Lifecycle & Ownership
- **Creation:** Instances are deserialized from asset configuration files (e.g., JSON) at server startup by the `BuilderCodec`. They are not intended to be instantiated dynamically during gameplay. The `afterDecode` hook is critical for post-processing, transforming raw string-based maps into optimized, integer-keyed maps for high-performance lookups during combat calculations.

- **Scope:** An instance of WieldingInteraction is effectively a singleton for its specific asset definition. It persists for the entire server session once loaded into the asset registry.

- **Destruction:** Instances are garbage collected when the server shuts down and the asset registries are cleared. There is no manual destruction logic.

## Internal State & Concurrency
- **State:** The object is effectively immutable after its creation and the execution of the `afterDecode` hook. All its properties, such as `damageModifiers` and `staminaCost`, are defined in the source asset files. The `transient` fields `knockbackModifiers` and `damageModifiers` are write-once caches populated during deserialization to improve runtime performance.

- **Thread Safety:** The class is inherently thread-safe for read operations. Because its state is fixed after loading, multiple threads from the server's game loop can safely access a shared instance to calculate damage for different entities simultaneously without requiring locks or synchronization. All mutation is confined to the entity's own components, not the shared interaction object.

## API Surface
The public API is designed for read-only access by other game systems, primarily the damage calculation pipeline.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getKnockbackModifiers() | Int2DoubleMap | O(1) | Returns the cached map of knockback modifiers, keyed by DamageCause index. |
| getDamageModifiers() | Int2FloatMap | O(1) | Returns the cached map of damage modifiers, keyed by DamageCause index. |
| getAngledWielding() | AngledWielding | O(1) | Provides configuration for angle-dependent blocking mechanics. |
| getBlockedEffects() | DamageEffects | O(1) | Returns the effects to apply to the attacker or defender on a successful block. |
| getStaminaCost() | StaminaCost | O(1) | Provides the configuration for calculating stamina drain upon taking damage. |
| getBlockedInteractions() | String | O(1) | Returns the asset name of the interaction to trigger after a successful block. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use in typical gameplay logic. Instead, systems query an entity's components to see if a WieldingInteraction is active. The primary integration point is reading the instance from the DamageDataComponent during damage processing.

```java
// Example from within a hypothetical damage processing system
void applyDamage(Ref<EntityStore> victim, DamageInfo damage) {
    CommandBuffer<EntityStore> commands = getCommandBuffer();
    DamageDataComponent damageData = commands.getComponent(victim, DamageDataComponent.getComponentType());

    // This is the correct integration point
    WieldingInteraction activeWielding = damageData.getCurrentWielding();

    if (activeWielding != null) {
        int causeIndex = DamageCause.getAssetMap().getIndex(damage.cause);
        float modifier = activeWielding.getDamageModifiers().getOrDefault(causeIndex, 1.0f);
        damage.amount *= modifier;
    }

    // ... continue damage application
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new WieldingInteraction()`. The object is complex and requires its `BuilderCodec` to correctly initialize its state from an asset file. Manual creation will result in a non-functional object with null fields.

- **State Mutation:** Do not attempt to modify the state of a WieldingInteraction instance after it has been loaded. These objects are shared across the entire server; runtime mutation would cause unpredictable and global changes to game mechanics.

- **Misinterpreting `tick0`:** The `tick0` method is part of the internal interaction state machine. Calling it directly will bypass the state management provided by the InteractionContext and lead to inconsistent entity state.

## Data Pipeline
The WieldingInteraction acts as a data source and state-check within the server's combat loop. Its primary flow involves being activated by player input and subsequently influencing the damage calculation pipeline.

> Flow:
> Player Input -> Interaction System (`tick0`) -> Sets `WieldingInteraction` on `DamageDataComponent` -> Incoming Damage Event -> Damage Calculation Pipeline reads `DamageDataComponent` -> **WieldingInteraction** provides modifiers -> Final Damage Calculated

