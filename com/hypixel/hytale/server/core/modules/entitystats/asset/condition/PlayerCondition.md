---
description: Architectural reference for PlayerCondition
---

# PlayerCondition

**Package:** com.hypixel.hytale.server.core.modules.entitystats.asset.condition
**Type:** Transient

## Definition
```java
// Signature
public class PlayerCondition extends Condition {
```

## Architecture & Concepts
The PlayerCondition class is a specific implementation of the **Condition** strategy pattern, designed to operate within the server-side Entity Stats module. Its sole responsibility is to encapsulate a single, verifiable rule: whether a target entity, assumed to be a player, is currently in a specific GameMode.

This component is not intended for direct, procedural use. Instead, it is a data-driven building block. Instances are deserialized from asset files (e.g., JSON or HOCON) using the provided static **CODEC**. This allows game designers to construct complex conditional logic within data, such as "apply this stat bonus only if the player is in creative mode," without modifying server source code.

During evaluation, the `eval0` method serves as the execution point. It interfaces with the Entity Component System (ECS) via a ComponentAccessor to query the state of the target entity, making it a bridge between declarative asset definitions and the live, stateful game world.

## Lifecycle & Ownership
- **Creation:** PlayerCondition instances are created exclusively by the Hytale **Codec** system during the server's asset loading phase. A higher-level system, such as a stat or effect manager, encounters a PlayerCondition definition in an asset and uses the static `PlayerCondition.CODEC` to deserialize it into a Java object.
- **Scope:** The object's lifetime is bound to the containing asset or configuration that defined it. It is effectively an immutable data object once created and persists as long as its parent configuration is held in memory.
- **Destruction:** The object is managed by the Java garbage collector. It is reclaimed when the root asset that defined it is unloaded from memory. There are no manual destruction or cleanup methods.

## Internal State & Concurrency
- **State:** This class is **effectively immutable**. The internal `gameModeToCheck` field is populated once during deserialization by the CODEC and is not modified thereafter. The `eval0` method is a pure function with respect to the class's own state, relying entirely on the external state of the entity passed to it.
- **Thread Safety:** The class itself is inherently thread-safe due to its immutable nature. Multiple threads can safely call `eval0` on the same instance without causing data corruption.

    **WARNING:** While the object is thread-safe, the provided ComponentAccessor and the underlying EntityStore may not be. Callers are responsible for ensuring that any evaluation occurs within a thread-safe context, typically the main server game loop, to prevent race conditions when accessing entity component data.

## API Surface
The primary public contract is the inherited `eval0` method, used to execute the condition check.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| eval0(accessor, ref, time) | boolean | O(1) | Evaluates the condition against the entity specified by the Ref. Returns true if the entity has a Player component with a GameMode matching the configured value. If `gameModeToCheck` was not defined in the asset, this method always returns true. |

## Integration Patterns

### Standard Usage
This class is not meant to be instantiated or used directly in procedural code. Its primary interaction model is declarative, defined within game asset files. The system evaluates it as part of a larger conditional check.

```java
// Conceptual example of how a manager would use a loaded condition
// A developer would NOT write this code.

// 1. Condition is loaded from an asset file via PlayerCondition.CODEC
Condition playerIsCreative = loadedEffect.getCondition(); // Assume this is a PlayerCondition

// 2. The system evaluates it against a target player entity
boolean canApplyEffect = playerIsCreative.eval(accessor, playerEntityRef, now);

if (canApplyEffect) {
    // ... apply stat modifier or effect
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new PlayerCondition()`. The constructor is protected to prevent this. All instances must be created via the framework's CODEC system to ensure they are correctly configured. An unconfigured instance where `gameModeToCheck` is null will always evaluate to true, likely causing unintended behavior.
- **External State Modification:** Do not attempt to modify the `gameModeToCheck` field after instantiation using reflection. This violates the immutability contract and can lead to unpredictable behavior across the system.

## Data Pipeline
The PlayerCondition functions as a processing node in a data flow that begins with a static asset file and ends with a boolean result that influences live gameplay.

> Flow:
> Game Asset (e.g., JSON) -> Hytale Codec Deserializer -> **PlayerCondition Instance** -> Stat/Effect System -> `eval0` call during game tick -> ECS Query for Player Component -> Boolean Result

