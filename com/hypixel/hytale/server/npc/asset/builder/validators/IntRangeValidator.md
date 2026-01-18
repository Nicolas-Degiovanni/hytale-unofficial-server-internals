---
description: Architectural reference for IntRangeValidator
---

# IntRangeValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Utility

## Definition
```java
// Signature
public class IntRangeValidator extends IntValidator {
```

## Architecture & Concepts
The IntRangeValidator is a specialized, immutable component within the server's NPC asset validation framework. Its primary function is to enforce numerical constraints on integer properties read from asset configuration files. It acts as a specific strategy for the more generic IntValidator, encapsulating the logic for checking if a value falls within a predefined range.

This class is designed for clarity and reusability. By combining two integer bounds with two RelationalOperator enums, it can represent any standard interval:
*   Fully inclusive: `[lower, upper]`
*   Fully exclusive: `(lower, upper)`
*   Half-open: `[lower, upper)` or `(lower, upper]`

This design centralizes boundary-checking logic, preventing the proliferation of manual `if (value >= X && value < Y)` checks throughout the asset building codebase. It promotes a declarative style of validation where rules are defined as objects and then applied to data.

## Lifecycle & Ownership
- **Creation:** IntRangeValidator instances are transient and created on-demand. They are typically instantiated within an asset builder or a configuration processor using one of the provided static factory methods, such as `between` or `fromInclToExcl`.

- **Scope:** The object's lifetime is extremely short. It is scoped to the validation phase of a single asset's construction. It does not persist and is not shared between different asset-building operations.

- **Destruction:** The object is eligible for garbage collection as soon as the `test` method returns and its result has been processed. No manual resource management is required.

## Internal State & Concurrency
- **State:** **Immutable**. All internal fields (`lower`, `upper`, `relationLower`, `relationUpper`) are `private final` and are set exclusively at construction time. An instance of IntRangeValidator cannot be modified after it is created.

- **Thread Safety:** **Inherently thread-safe**. Due to its immutable nature, a single IntRangeValidator instance can be safely shared and executed by multiple threads concurrently without any risk of race conditions or data corruption. No external locking is necessary.

## API Surface
The public contract is minimal and focused on validation and error reporting.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(int value) | boolean | O(1) | Executes the validation logic. Returns true if the value satisfies both the lower and upper bound conditions. |
| errorMessage(int value, String name) | String | O(1) | Generates a descriptive, human-readable error message for a failed validation, including the property name. |
| between(int lower, int upper) | static IntRangeValidator | O(1) | Factory method for creating a validator for an inclusive range `[lower, upper]`. |
| fromInclToExcl(int lower, int upper) | static IntRangeValidator | O(1) | Factory method for creating a validator for a half-open range `[lower, upper)`. |
| fromExclToIncl(int lower, int upper) | static IntRangeValidator | O(1) | Factory method for creating a validator for a half-open range `(lower, upper]`. |

## Integration Patterns

### Standard Usage
The validator is intended to be used as part of a larger data validation or asset construction process. It is created, used immediately for a check, and then discarded.

```java
// In a hypothetical NPC asset builder
int maxHealth = npcData.getMaxHealth(); // Read from config
IntRangeValidator healthValidator = IntRangeValidator.between(1, 10000);

if (!healthValidator.test(maxHealth)) {
    throw new AssetValidationException(
        healthValidator.errorMessage(maxHealth, "Max Health")
    );
}

// The validated value is now safe to use
npc.setMaxHealth(maxHealth);
```

### Anti-Patterns (Do NOT do this)
- **Manual Logic Duplication:** Avoid writing manual range checks where this validator can be used. The validator centralizes the rule and provides standardized error messages.
  ```java
  // BAD: Brittle, custom logic and error message
  if (value < 1 || value > 10) {
      throw new Exception("Value must be between 1 and 10");
  }

  // GOOD: Uses the centralized, reusable component
  IntRangeValidator.between(1, 10).test(value);
  ```
- **Using the Constructor Directly:** While the public constructor is available, the static factory methods like `between` are strongly preferred. They are more expressive and reduce the chance of errors from mixing up `RelationalOperator` arguments.

## Data Pipeline
The IntRangeValidator is a gate in the data transformation pipeline from raw configuration to a finalized in-memory game asset. It does not transform data but rather asserts its correctness.

> Flow:
> NPC Configuration File -> Deserializer -> Raw Data Object -> Asset Builder -> **IntRangeValidator.test()** -> Validation Result (Pass/Fail) -> Finalized NPC Asset<ctrl63>

