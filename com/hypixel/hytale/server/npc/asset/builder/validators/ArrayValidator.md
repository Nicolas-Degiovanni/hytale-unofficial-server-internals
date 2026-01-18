---
description: Architectural reference for ArrayValidator
---

# ArrayValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Base Class / Strategy

## Definition
```java
// Signature
public abstract class ArrayValidator extends Validator {
```

## Architecture & Concepts
The ArrayValidator is an abstract base class that establishes a formal contract for validation logic applied to array-like structures within the server-side NPC asset definition pipeline. It serves as a core component of the data integrity and schema enforcement system for game assets, ensuring that configured NPC data conforms to engine and game design constraints before being loaded into the server.

This class embodies the **Strategy Pattern**. It defines an abstract validation algorithm, deferring the specific implementation details—the actual test logic and error message generation—to concrete subclasses. This design allows the asset building framework to remain decoupled from the specifics of any single validation rule, enabling developers to introduce new, complex validation rules without modifying the core asset loading machinery.

It operates exclusively on the BuilderObjectArrayHelper, an abstraction that provides a unified interface over array-like data encountered during the asset build process.

## Lifecycle & Ownership
- **Creation:** Concrete implementations of ArrayValidator are instantiated by the NPC asset building framework during the validation phase of an asset's lifecycle. They are typically created on-demand by a higher-level builder or a validator registry.
- **Scope:** Transient and short-lived. An instance's lifetime is confined to a single validation operation on a specific array within an asset file. It is created, its test method is invoked, and it is then eligible for garbage collection.
- **Destruction:** Instances are managed by the Java Garbage Collector and are reclaimed shortly after the validation check is complete. They hold no persistent state or external resources that would require manual cleanup.

## Internal State & Concurrency
- **State:** Stateless. As an abstract class, it holds no state. Concrete implementations are expected to be stateless as well, with their behavior depending solely on the input parameters provided to their methods. This ensures that validation logic is pure, predictable, and free from side effects.
- **Thread Safety:** Inherently thread-safe. Due to its stateless design, a single instance of a concrete ArrayValidator can be safely used by multiple threads simultaneously to validate different data structures without risk of interference or data corruption.

## API Surface
The public contract consists entirely of abstract methods that must be implemented by subclasses.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(BuilderObjectArrayHelper) | boolean | Varies | **Core validation method.** Executes the specific validation logic against the provided array helper. Returns true if valid, false otherwise. |
| errorMessage(String, BuilderObjectArrayHelper) | String | Varies | Generates a detailed, context-aware error message, typically including the field name. |
| errorMessage(BuilderObjectArrayHelper) | String | Varies | Generates a generic error message for the validation failure. |

## Integration Patterns

### Standard Usage
A developer's primary interaction is through the creation of a concrete subclass to implement a specific business rule. The framework then applies this validator.

```java
// 1. Define a concrete validation strategy
public class MinimumItemCountValidator extends ArrayValidator {
    private final int minCount;

    public MinimumItemCountValidator(int minCount) {
        this.minCount = minCount;
    }

    @Override
    public boolean test(BuilderObjectArrayHelper<?, ?> arrayHelper) {
        return arrayHelper.size() >= this.minCount;
    }

    @Override
    public String errorMessage(String fieldName, BuilderObjectArrayHelper<?, ?> arrayHelper) {
        return String.format("Field '%s' must have at least %d items, but found %d.",
            fieldName, this.minCount, arrayHelper.size());
    }
    
    // ... other errorMessage implementation
}

// 2. The asset builder framework applies the validator
BuilderObjectArrayHelper<?, ?> itemsToValidate = asset.getArray("equipment");
Validator minItems = new MinimumItemCountValidator(1);

if (!minItems.test(itemsToValidate)) {
    throw new AssetBuildException(minItems.errorMessage("equipment", itemsToValidate));
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to use `new ArrayValidator()`. The class is abstract and cannot be instantiated. You must extend it.
- **Stateful Implementations:** Avoid adding mutable member variables to concrete subclasses. A validator's result should never depend on previous invocations, as this breaks thread safety and predictability.
- **Operating on Raw Collections:** Do not design subclasses to operate on raw Java collections like List or arrays. The entire validation pipeline is built around the BuilderObjectArrayHelper abstraction to provide necessary context and metadata from the asset source.

## Data Pipeline
The ArrayValidator is a critical processing stage within the server's asset ingestion pipeline. It acts as a gatekeeper, ensuring data quality before an asset is fully constructed and loaded into the game world.

> Flow:
> NPC Asset File (e.g., JSON) -> Parser -> Raw Data Structure -> BuilderObjectArrayHelper -> **Concrete ArrayValidator** -> Validation Result (Pass/Fail) -> Asset Instantiation OR Build Error Report

