---
description: Architectural reference for ValidateAssetIfEnumIsValidator
---

# ValidateAssetIfEnumIsValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Transient / Value Object

## Definition
```java
// Signature
public class ValidateAssetIfEnumIsValidator<E extends Enum<E> & Supplier<String>> extends Validator {
```

## Architecture & Concepts
The ValidateAssetIfEnumIsValidator is a specialized, conditional rule within the server's asset validation framework. It does not perform validation itself; instead, it acts as a logical gate, deciding whether to execute another, composed AssetValidator based on the state of the asset data.

Architecturally, this class embodies the **Strategy** and **Composite** patterns. It is a concrete implementation of the abstract Validator, but its primary responsibility is to conditionally delegate to another Validator instance. This allows for the construction of complex, dynamic validation chains from simple, reusable components.

The core function is to check if a specific parameter within an asset (identified by *parameter2*) has a string value that matches the string representation of a specific enum constant (*enumValue*). If and only if this condition is met, it applies the composed *validator* to a different parameter (identified by *parameter1*).

The generic constraint, `<E extends Enum<E> & Supplier<String>>`, is a critical design choice. It mandates that any enum used with this validator must also provide its own string representation via the Supplier interface. This decouples the validation logic from the specific string values of enum constants, which is essential for handling asset data formats like JSON where enums are represented as strings.

## Lifecycle & Ownership
- **Creation:** Instantiated exclusively through the static factory method `withAttributes`. This class is typically created by a higher-level asset builder or configuration loader that parses validation rules defined for a specific NPC or entity type. It is not intended for manual creation during the game loop.
- **Scope:** Extremely short-lived. An instance exists only for the duration of a single asset's validation process. It is a configuration object, not a persistent service.
- **Destruction:** The object is eligible for garbage collection as soon as the validation chain it belongs to has finished executing. No manual cleanup is required.

## Internal State & Concurrency
- **State:** **Immutable**. All internal fields are final and are set once in the private constructor. The object's configuration cannot be altered after creation. The `transient` keyword on the `validator` field indicates it is excluded from Java's default serialization process, which is consistent with its role as a runtime-only logic component.
- **Thread Safety:** **Fully thread-safe**. Its immutability guarantees that it can be safely used across multiple threads without any external locking or synchronization. An entire validation chain composed of such objects can be executed concurrently on different assets.

## API Surface
The public contract is limited to its static factory method, as the primary interaction is through the `validate` method inherited from the parent Validator class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| withAttributes(p1, validator, p2, value) | ValidateAssetIfEnumIsValidator | O(1) | Constructs a new validator instance with the specified conditional logic. |

## Integration Patterns

### Standard Usage
This validator is designed to be part of a larger collection of validation rules. It is constructed and added to a builder or registry during the asset definition phase.

```java
// Define an enum that meets the generic constraints
public enum AssetType implements Supplier<String> {
    WEAPON("weapon"),
    ARMOR("armor");

    private final String key;
    AssetType(String key) { this.key = key; }
    public String get() { return this.key; }
}

// Assume 'someOtherValidator' is a pre-existing AssetValidator
// that checks if an asset path is valid.
AssetValidator pathValidator = new ValidateAssetPathExists();

// Create the conditional validator
ValidateAssetIfEnumIsValidator validator = ValidateAssetIfEnumIsValidator.withAttributes(
    "model",          // Parameter to validate if condition is met
    pathValidator,    // The validator to run
    "type",           // Parameter to check the value of
    AssetType.WEAPON  // The required enum value
);

// This 'validator' would now be added to an asset definition's
// validation chain.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to instantiate this class using reflection. The private constructor is an intentional design choice to enforce valid object creation through the `withAttributes` factory method.
- **State Modification:** Do not attempt to modify the internal state via reflection. The immutability of this object is critical for ensuring predictable behavior and thread safety in the validation pipeline.
- **Incorrect Enum Type:** Passing an enum that does not implement `Supplier<String>` will result in a compile-time error. The system relies on this contract to compare asset data against the enum's string key.

## Data Pipeline
This component acts as a conditional branch within a larger data validation pipeline. It receives the asset data structure, inspects it, and either terminates its branch of the validation or delegates further down the chain.

> Flow:
> Asset Data (JSON/Binary) -> Asset Deserializer -> Validator Chain -> **ValidateAssetIfEnumIsValidator**
> 1.  Reads string value of `parameter2` from Asset Data.
> 2.  Compares value with `enumValue.get()`.
> 3.  **If Match:** Invokes `validator.validate()` on `parameter1`.
> 4.  **If No Match:** The check succeeds trivially; no further action is taken.
> Validation Result -> Asset Instantiation

