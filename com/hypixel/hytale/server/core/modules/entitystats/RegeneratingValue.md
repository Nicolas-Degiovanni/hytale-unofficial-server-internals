---
description: Architectural reference for RegeneratingValue
---

# RegeneratingValue

**Package:** com.hypixel.hytale.server.core.modules.entitystats
**Type:** Transient State Object

## Definition
```java
// Signature
public class RegeneratingValue {
```

## Architecture & Concepts
The RegeneratingValue class is a stateful, transient object responsible for managing the runtime logic of a single regenerating statistic for a specific entity. It is not a global system or service, but rather a granular component that encapsulates the state and behavior of one instance of regeneration, such as a player's health or a creature's mana.

Its primary architectural role is to act as a bridge between static asset data and dynamic game state. It consumes a configuration object, EntityStatType.Regenerating (defined in assets), and uses it to drive a per-tick simulation. This class holds the mutable state, specifically the countdown timer for the next regeneration tick, decoupling the per-entity state from the shared, immutable regeneration rules.

This object is a fundamental building block of the server-side Entity Statistics module and is designed to be processed by a higher-level system during the main server game loop.

### Lifecycle & Ownership
- **Creation:** A RegeneratingValue is instantiated when an entity gains a statistic with regeneration properties. This is typically managed by a parent component, such as an EntityStatComponent, which reads the entity's definition and creates a new RegeneratingValue instance for each configured regenerating stat.
- **Scope:** The lifecycle of a RegeneratingValue instance is strictly bound to its parent component on a single entity. It persists as long as the entity possesses the associated regenerating statistic.
- **Destruction:** The object has no explicit destruction or cleanup method. It becomes eligible for garbage collection when its owning entity is removed from the world or its parent component is destroyed.

## Internal State & Concurrency
- **State:** This class is mutable. Its core state is the `remainingUntilRegen` field, a float that serves as a countdown timer. The `regenerating` field holds an immutable reference to the asset-defined configuration for the stat, which does not change during the object's lifetime.
- **Thread Safety:** **This class is not thread-safe.** It is designed with the expectation of being accessed and modified by a single thread, specifically the main server game loop thread. The `shouldRegenerate` method performs a non-atomic read-modify-write operation on the `remainingUntilRegen` field. Any concurrent access from multiple threads will result in race conditions, incorrect timer calculations, and unpredictable regeneration behavior. All processing must be serialized.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| regenerate(...) | float | O(N+M) | The primary entry point. Calculates and returns the amount to regenerate for the current tick. Returns 0 if no regeneration should occur. Complexity depends on N conditions and M modifiers. |
| shouldRegenerate(...) | boolean | O(N) | Determines if a regeneration tick should occur. **Warning:** This method has the side effect of decrementing the internal timer. Complexity depends on N conditions to check. |

## Integration Patterns

### Standard Usage
A managing system, such as an EntityStatSystem or a parent component, should call the `regenerate` method once per game tick. The returned value is then applied to the entity's actual stat value.

```java
// Hypothetical usage within an EntityStatComponent's update method
void onTick(float deltaTime, Instant currentTime, ComponentAccessor<EntityStore> store, Ref<EntityStore> ref) {
    // this.regeneratingValue is the instance of RegeneratingValue
    // this.statValue is the instance of EntityStatValue
    float amountToRegen = this.regeneratingValue.regenerate(
        store,
        ref,
        currentTime,
        deltaTime,
        this.statValue,
        this.statValue.getCurrent()
    );

    if (amountToRegen > 0) {
        this.statValue.add(amountToRegen);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Instance Sharing:** Never share a single RegeneratingValue instance across multiple entities or even multiple stats on the same entity. Each regenerating stat requires its own unique instance to track its timer state independently.
- **Multi-threaded Access:** Do not call any methods on this class from worker threads or asynchronous tasks. All interactions must occur on the main server thread to prevent data corruption.
- **Calling shouldRegenerate Separately:** Avoid calling `shouldRegenerate` and then `regenerate` in the same tick. The `regenerate` method calls `shouldRegenerate` internally. Calling it separately will cause the timer to be decremented twice in one tick, leading to faster-than-intended regeneration.

## Data Pipeline
The flow of data is driven by the server's main game loop and is processed sequentially for each relevant entity.

> Flow:
> Server Game Tick -> Entity Update System -> **RegeneratingValue.regenerate()** -> Checks Conditions -> Applies Modifiers -> Returns clamped float -> Entity Stat Component is updated

