---
description: Architectural reference for DefaultModifiers
---

# DefaultModifiers

**Package:** com.hypixel.hytale.server.core.modules.entitystats.modifier
**Type:** Utility

## Definition
```java
// Signature
public interface DefaultModifiers {
```

## Architecture & Concepts
The DefaultModifiers interface serves as a centralized, canonical dictionary for the string identifiers of core stat modifier sources. Its primary architectural function is to eliminate the use of "magic strings" within the entity statistics system. By providing a single source of truth for these common identifiers, it ensures consistency and type safety across disparate modules that interact with entity stats, such as the combat, armor, and status effect systems.

This interface establishes a shared vocabulary. When the armor system grants a bonus, it tags the modifier with the ARMOR constant. When a potion effect is applied, it uses the EFFECT constant. This allows the core stat calculation engine to correctly attribute, aggregate, and potentially remove modifiers based on their source without needing to know the implementation details of the originating system.

## Lifecycle & Ownership
- **Creation:** As an interface composed entirely of static final fields, DefaultModifiers is never instantiated. Its constants are initialized by the JVM Class Loader when the interface is first referenced by any part of the server application.
- **Scope:** The constants are application-scoped. They are loaded once and persist for the entire lifetime of the server process.
- **Destruction:** The constants are unloaded only when the application's Class Loader is garbage collected, which effectively occurs at server shutdown.

## Internal State & Concurrency
- **State:** DefaultModifiers holds no instance state. Its members are compile-time constants (`public static final String`), making them deeply immutable.
- **Thread Safety:** The interface is inherently thread-safe. Its constants can be safely accessed from any thread without requiring synchronization primitives. This is a critical property, as the entity stat system may be accessed by multiple threads processing game logic concurrently.

## API Surface
The public contract consists solely of static string constants used as identifiers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| EFFECT | String | O(1) | Canonical key for modifiers originating from status effects (e.g., potions, debuffs). |
| ARMOR | String | O(1) | Canonical key for modifiers originating from equipped armor pieces. |
| WEAPON_MODIFIER_PREFIX | String | O(1) | A required prefix for all modifiers originating from weapon properties or enchantments. |
| UTILITY_MODIFIER_PREFIX | String | O(1) | A required prefix for all modifiers originating from non-weapon, non-armor utility items. |

## Integration Patterns

### Standard Usage
The constants within DefaultModifiers should be used to identify the source of a StatModifier upon its creation. This ensures that the stat system can correctly process and categorize the modifier.

```java
// Correctly applying a health bonus from an armor piece
StatModifier armorHealthBonus = new StatModifier(
    DefaultModifiers.ARMOR,
    "hytale:health",
    StatOperation.ADDITIVE,
    50.0
);
entity.getStatContainer().addModifier(armorHealthBonus);
```

### Anti-Patterns (Do NOT do this)
- **Implementing the Interface:** A class **must not** implement DefaultModifiers to gain access to its constants. This is a widely recognized anti-pattern that pollutes the class's public API and incorrectly implies an "is-a" relationship. Use static imports or fully qualified names instead (e.g., `DefaultModifiers.ARMOR`).
- **Hardcoding Identifiers:** Never use literal strings where a constant from this interface is available. Using `"Armor"` instead of `DefaultModifiers.ARMOR` introduces fragility and creates a high risk of typos that will not be caught at compile time, leading to difficult-to-diagnose bugs in stat calculations.

## Data Pipeline
This interface does not process data itself; rather, its constants are used to tag and categorize data flowing through the entity statistics pipeline.

> Flow:
> Game Event (e.g., Equip Armor) -> Armor System Logic -> Creates `StatModifier` tagged with **DefaultModifiers.ARMOR** -> StatContainer -> Final Stat Calculation

