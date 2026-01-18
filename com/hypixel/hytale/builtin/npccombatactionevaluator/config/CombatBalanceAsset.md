---
description: Architectural reference for CombatBalanceAsset
---

# CombatBalanceAsset

**Package:** com.hypixel.hytale.builtin.npccombatactionevaluator.config
**Type:** Configuration Data Object

## Definition
```java
// Signature
public class CombatBalanceAsset extends BalanceAsset {
```

## Architecture & Concepts
The **CombatBalanceAsset** is a data-driven configuration object that defines the combat behavior and memory parameters for a Non-Player Character (NPC). It serves as a critical bridge between game design data and the server-side AI engine.

This class is not a service or an active component. Instead, it is a passive data container, deserialized from asset files (e.g., JSON) at runtime. Its primary architectural purpose is to decouple NPC combat logic from the core engine code, allowing designers and modders to define and tune complex AI behaviors without modifying Java source.

It extends the base **BalanceAsset**, inheriting common balancing properties, and specializes it by adding configuration for the **CombatActionEvaluator** system. This system is responsible for utility-based AI decision-making, where an NPC evaluates a list of potential actions against a set of conditions to select the optimal behavior in a given combat scenario. The **CombatBalanceAsset** provides the raw data—the actions, conditions, and memory settings—that fuels this evaluator.

## Lifecycle & Ownership
- **Creation:** Instances of **CombatBalanceAsset** are exclusively created by the Hytale **Codec** system during the server's asset loading phase. The static **CODEC** field defines the schema for parsing the corresponding asset file. Direct instantiation via the constructor is a critical anti-pattern.
- **Scope:** The object's lifetime is tied to the asset management system. It is loaded into memory and typically persists as long as the world or zone requiring this asset is active. It is effectively a read-only singleton for a given asset definition.
- **Destruction:** The object is eligible for garbage collection when the **AssetManager** unloads the corresponding asset package, typically upon server shutdown or when transitioning to a world that no longer references it.

## Internal State & Concurrency
- **State:** The state of a **CombatBalanceAsset** is considered *effectively immutable* after deserialization. While its fields are not declared as final, they are populated once by the **Codec** system and are not intended to be modified thereafter. It acts as a read-only data source for the AI systems that consume it.
- **Thread Safety:** This class is inherently thread-safe for read operations. Because its state does not change after initialization, multiple NPC AI threads can safely access the same **CombatBalanceAsset** instance concurrently without locks or synchronization. All mutation is confined to the single-threaded asset loading pipeline at startup.

## API Surface
The public API is minimal, designed for read-only access to the configured data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getTargetMemoryDuration() | float | O(1) | Returns the duration in seconds an NPC will remember a target after losing line of sight. |
| getEvaluatorConfig() | CombatActionEvaluatorConfig | O(1) | Returns the comprehensive configuration for the utility-based AI decision engine. |

## Integration Patterns

### Standard Usage
The intended pattern is to retrieve a pre-loaded instance from a registry or asset provider. The NPC's AI controller then uses this configuration to initialize its internal state.

```java
// Example from within an NPC AI initialization method
CombatBalanceAsset balance = assetProvider.get(CombatBalanceAsset.class, "hytale:orc_brute");
this.combatAI.configure(balance.getEvaluatorConfig());
this.targetMemory.setDuration(balance.getTargetMemoryDuration());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use **new CombatBalanceAsset()**. This bypasses the **Codec** system and results in a default, unconfigured object that will cause predictable AI failures. The object is useless without being populated from an asset file.
- **State Mutation:** Do not attempt to modify the state of a **CombatBalanceAsset** instance after it has been loaded. This violates its contract as a read-only data source and can lead to unpredictable, non-deterministic behavior across all NPCs that share this asset.

## Data Pipeline
The **CombatBalanceAsset** is a key stage in the data flow from design files to live AI behavior.

> Flow:
> Asset File (JSON) -> Server Asset Loader -> **CombatBalanceAsset.CODEC** -> **CombatBalanceAsset Instance** -> NPC AI Component -> In-Game Combat Decision

