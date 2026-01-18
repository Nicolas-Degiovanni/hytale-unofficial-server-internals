---
description: Architectural reference for DefaultEntityStatTypes
---

# DefaultEntityStatTypes

**Package:** com.hypixel.hytale.server.core.modules.entitystats.asset
**Type:** Utility

## Definition
```java
// Signature
public abstract class DefaultEntityStatTypes {
```

## Architecture & Concepts
DefaultEntityStatTypes serves as a static, high-performance cache for the integer indices of core entity statistics. In the Hytale engine, entity stats like Health and Mana are defined as string-based assets but are referenced in performance-critical game logic using integer identifiers for efficiency. This class is the bridge between those two worlds.

Its primary architectural role is to eliminate repeated, costly string lookups within the asset system during the main game loop. By caching these essential stat indices at startup, systems like combat, AI, and status effect handlers can access them via direct, zero-overhead static method calls.

This class is a foundational utility, expected to be initialized early in the server lifecycle and then treated as a read-only source of truth for the remainder of the session.

### Lifecycle & Ownership
- **Creation:** The class is never instantiated; its private constructor enforces a pure-static access pattern. The static fields are loaded by the JVM, but their meaningful state is uninitialized (zero) until explicitly populated.
- **Scope:** The cached integer indices persist for the entire lifetime of the server process. This is global, static state.
- **Destruction:** There is no explicit destruction or cleanup mechanism. The state is only lost upon server shutdown. The `update` method is an initialization or re-initialization trigger, not a cleanup routine.

## Internal State & Concurrency
- **State:** The class holds mutable, static integer fields. This state is intended to be **write-once, read-many**. Before the `update` method is called, the state is uninitialized and invalid (all values are 0). After `update` completes, the state is considered stable and canonical.

- **Thread Safety:** This class is **not thread-safe**. The `update` method performs non-atomic writes to shared static variables. Accessing the getter methods while `update` is executing on another thread will result in undefined behavior, potentially yielding a partially updated or default (0) value.

    **WARNING:** All interaction with this class must be strictly controlled. The `update` method must be called from a single thread during a controlled startup sequence before any other game systems can access the getters.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getHealth() | int | O(1) | Returns the cached integer index for the Health stat. |
| getOxygen() | int | O(1) | Returns the cached integer index for the Oxygen stat. |
| getStamina() | int | O(1) | Returns the cached integer index for the Stamina stat. |
| getMana() | int | O(1) | Returns the cached integer index for the Mana stat. |
| getSignatureEnergy() | int | O(1) | Returns the cached integer index for the SignatureEnergy stat. |
| getAmmo() | int | O(1) | Returns the cached integer index for the Ammo stat. |
| update() | void | O(1) | Populates the static fields by querying the EntityStatType asset map. This is a write operation. |

## Integration Patterns

### Standard Usage
The `update` method must be called once during the server's initialization phase, after all game assets have been loaded and indexed. Subsequently, game logic should use the static getters to retrieve the stat indices.

```java
// During server startup, after assets are loaded...
DefaultEntityStatTypes.update();

// In a game system, e.g., a combat handler...
public void applyDamage(Entity target, float amount) {
    int healthStatId = DefaultEntityStatTypes.getHealth();
    StatComponent stats = target.getComponent(StatComponent.class);
    stats.modifyValue(healthStatId, -amount);
}
```

### Anti-Patterns (Do NOT do this)
- **Premature Access:** Calling any `get` method before `update` has been successfully executed. This will return 0, which is likely an invalid or incorrect index, leading to critical gameplay bugs that are difficult to trace.
- **Re-initialization During Gameplay:** Calling `update` after the server is running and the game loop is active. This introduces a race condition and is not supported. The values are assumed to be immutable post-startup.
- **Direct Instantiation:** Attempting to create an instance with `new DefaultEntityStatTypes()` is prohibited by the private constructor and violates the class's design contract.

## Data Pipeline
This class does not process a continuous stream of data. Instead, it participates in a one-time initialization flow at server startup.

> Flow:
> Asset System Loads & Indexes All Assets -> `EntityStatType.getAssetMap()` becomes populated -> **`DefaultEntityStatTypes.update()`** is called -> Static integer fields are populated from the asset map -> Game Systems read cached integers from **DefaultEntityStatTypes** getters.

