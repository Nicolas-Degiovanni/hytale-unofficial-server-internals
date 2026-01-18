---
description: Architectural reference for EnumArrayValidator
---

# EnumArrayValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Abstract Base Class

## Definition
```java
// Signature
public abstract class EnumArrayValidator extends Validator {
```

## Architecture & Concepts
The EnumArrayValidator is an abstract base class that establishes a formal contract for validating arrays of enum constants within the server's NPC asset pipeline. It is a specialized component of the broader validation framework, inheriting from the base Validator class.

Its primary role is not to perform validation itself, but to define the required interface that all concrete enum array validation strategies must implement. This design enforces a consistent, type-safe approach for checking enum collections during asset deserialization and construction. By using generics `<T extends Enum<T>>`, the framework ensures that a single concrete implementation, such as a validator for unique enum values, can be reused across any enum type in the system (e.g., NpcState, NpcBehavior, NpcFaction).

This class is a key element in a Strategy design pattern, where an asset builder can be configured with a collection of different Validator implementations, each responsible for a specific rule.

### Lifecycle & Ownership
- **Creation:** This abstract class is never instantiated directly. Concrete subclasses are instantiated by the NpcAssetBuilder or a related factory during the asset loading process.
- **Scope:** Instances of concrete subclasses are transient and have a very short lifecycle. They typically exist only for the duration of a single validation operation on a specific field of an NPC asset.
- **Destruction:** As stateless, short-lived objects, instances are eligible for garbage collection immediately after their `test` method is invoked and the result is consumed. They do not hold references or manage resources.

## Internal State & Concurrency
- **State:** The EnumArrayValidator is designed to be completely stateless. It contains no member variables and all data required for its operations is passed as arguments to its methods. Concrete implementations are expected to adhere to this stateless principle.
- **Thread Safety:** This class, and any correctly implemented subclass, is inherently thread-safe. Because it holds no state, a single instance of a concrete validator can be safely shared and executed by multiple threads concurrently without locks or synchronization primitives.

## API Surface
The public contract consists entirely of abstract methods that must be implemented by subclasses.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(T[] values, Class<T> enumType) | boolean | Implementation-Defined | Executes the specific validation logic. Returns true if the array is valid, false otherwise. |
| errorMessage(String fieldName, T[] values) | String | O(N) | Constructs a detailed, human-readable error message when validation fails. |

## Integration Patterns

### Standard Usage
Developers do not interact with this class directly. Instead, they create a concrete implementation defining a specific validation rule and register it with the asset building system.

```java
// 1. Define a concrete validator for a specific rule
public class UniqueEnumValuesValidator extends EnumArrayValidator {
    @Override
    public <T extends Enum<T>> boolean test(T[] values, Class<T> enumType) {
        Set<T> uniqueValues = new HashSet<>(Arrays.asList(values));
        return uniqueValues.size() == values.length;
    }

    @Override
    public <T extends Enum<T>> String errorMessage(String fieldName, T[] values) {
        return "Field '" + fieldName + "' contains duplicate enum values.";
    }
}

// 2. The asset builder uses the concrete validator
// (This logic is internal to the builder)
Validator validator = new UniqueEnumValuesValidator();
boolean isValid = validator.test(npcData.getBehaviorStates(), NpcBehaviorState.class);
if (!isValid) {
    throw new AssetValidationException(validator.errorMessage("behaviorStates", npcData.getBehaviorStates()));
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Subclasses must not introduce mutable member variables. A validator that caches results or changes its behavior based on previous calls violates the core design and is not thread-safe.
- **Ignoring Generics:** Avoid casting or raw type usage in implementations. The generic contract `<T extends Enum<T>>` is provided to ensure compile-time type safety.

## Data Pipeline
EnumArrayValidator acts as a conditional gate within the NPC asset loading pipeline. It does not transform data but rather asserts its integrity, allowing the pipeline to proceed or halting it with an exception.

> Flow:
> NPC Asset Definition (JSON) -> Deserializer -> NpcAssetBuilder -> **Concrete EnumArrayValidator** -> (Success) Validated Asset | (Failure) ValidationException

