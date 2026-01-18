---
description: Architectural reference for WieldingCondition
---

# WieldingCondition

**Package:** com.hypixel.hytale.server.core.modules.entitystats.asset.condition
**Type:** Transient

## Definition
```java
// Signature
public class WieldingCondition extends Condition {
```

## Architecture & Concepts
The WieldingCondition is a specific, stateless implementation of the abstract Condition pattern. Its sole responsibility is to act as a predicate, evaluating to true if a target entity is currently wielding any item.

This class is a fundamental building block of the server's data-driven logic system, particularly for entity statistics and abilities. It is not a long-lived service but rather a transient data object, deserialized from asset files (e.g., JSON) that define game rules. For example, a "Berserker" status effect might grant a damage bonus only if a `WieldingCondition` is met.

Architecturally, it serves as a leaf node in a condition tree. The engine's ConditionEvaluator traverses these trees to determine if complex criteria are met before applying a game mechanic. The inherited `inverse` flag allows this component to be configured to check for the opposite state: "the entity is *not* wielding an item".

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale codec system during the server's asset loading phase. The static `CODEC` field is the factory responsible for deserializing a data definition (e.g., from a JSON object with a matching type identifier) into a concrete WieldingCondition object. **Warning:** Direct manual instantiation is an anti-pattern and circumvents the data-driven design of the engine.
- **Scope:** The object's lifetime is tied to the parent asset that contains it, such as a StatModifier or an Ability definition. It is effectively a piece of configuration data held in memory.
- **Destruction:** As a simple, transient object with no external resource handles, it is managed entirely by the Java garbage collector. It becomes eligible for collection once the parent asset is unloaded.

## Internal State & Concurrency
- **State:** **Immutable**. The only state held by a WieldingCondition instance is the `inverse` boolean, which is inherited from the base Condition class and set at creation time. The class itself does not cache data or maintain any mutable state between evaluations.
- **Thread Safety:** **Conditionally Thread-Safe**. The class itself is stateless and therefore inherently thread-safe. However, the `eval0` method operates on components within the Entity-Component-System (ECS) framework. The safety of an evaluation is therefore entirely dependent on the thread-safety guarantees of the `ComponentAccessor` and `EntityStore` context provided by the calling system. The WieldingCondition performs no internal locking and assumes the caller provides safe access to the entity's component data.

## API Surface
The primary contract is the inherited `eval` method, which internally calls `eval0`.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| eval0(accessor, ref, time) | boolean | O(1) | Core evaluation logic. Checks for the existence of a non-null item in the target entity's `DamageDataComponent`. Throws an `AssertionError` if the component is missing. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by gameplay programmers. It is configured declaratively within asset files. The system interacts with it through the generic Condition interface.

```java
// In a system that processes entity effects, a Condition is retrieved from an asset.
// This instance could be a WieldingCondition, loaded by the codec.
Condition condition = loadedEffect.getCondition();

// The system evaluates the condition against a specific entity.
boolean isConditionMet = condition.eval(ecsContext.getAccessor(), targetEntityRef, Instant.now());

if (isConditionMet) {
    // Apply the effect because the entity is wielding an item.
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new WieldingCondition()`. Game logic should be data-driven. If you need this check, define it in the appropriate JSON asset file. This ensures game designers can modify the logic without code changes.
- **Missing Component Dependency:** The evaluation logic asserts that a `DamageDataComponent` exists on the entity. Calling `eval` on an entity that lacks this component will cause a server crash. Any system that uses this condition must first ensure the target entity is of a type that includes this component.

## Data Pipeline
The flow begins with a designer's definition in an asset file and ends with a boolean result during gameplay.

> Flow:
> JSON Asset Definition -> Server Asset Loader -> **BuilderCodec** -> **WieldingCondition** Instance -> ConditionEvaluator -> Boolean Result

