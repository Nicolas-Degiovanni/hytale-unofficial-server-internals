---
description: Architectural reference for IntRangeBoundValidator
---

# IntRangeBoundValidator

**Package:** com.hypixel.hytale.math.range
**Type:** Utility

## Definition
```java
// Signature
public class IntRangeBoundValidator implements Validator<IntRange> {
```

## Architecture & Concepts
The IntRangeBoundValidator is a specialized, immutable component within the Hytale Codec framework. Its primary function is to enforce numerical constraints on a *single bound* (either the minimum or maximum value) of an IntRange object.

This class embodies the **Strategy Pattern**, providing a concrete implementation of the Validator interface. It serves two critical roles in the engine:

1.  **Runtime Data Validation:** During deserialization or data processing, it checks if an IntRange's value adheres to predefined limits.
2.  **Static Schema Definition:** It dynamically updates a data schema (e.g., for a JSON or binary format) to include these same constraints. This allows other systems, such as content editor UIs or external tools, to be aware of the validation rules without needing to execute the validation logic itself.

This dual-purpose design ensures that validation rules are defined in a single location but are enforced consistently across both runtime data flow and static data structure definitions.

### Lifecycle & Ownership
- **Creation:** Instances are created exclusively through the static factory methods `lowerBound` or `upperBound`. The constructor is private to prevent misconfiguration and enforce a clear, declarative instantiation pattern.
- **Scope:** An IntRangeBoundValidator is a configuration object. Its lifetime is bound to the schema or validation chain that references it. It does not persist beyond the scope of its containing configuration.
- **Destruction:** The object is managed by the Java garbage collector. It is eligible for collection as soon as the schema or validation process that created it is no longer in scope.

## Internal State & Concurrency
- **State:** The object is **fully immutable**. All internal fields (`min`, `max`, `inclusive`, `lowerBound`) are final and are set only once during construction via the static factory methods. It holds no reference to the data it validates and caches no results.

- **Thread Safety:** IntRangeBoundValidator is inherently **thread-safe**. Its immutable nature guarantees that it can be safely shared across multiple threads without any external locking or synchronization. Validation operations are pure functions with no side effects on the validator's state.

## API Surface
The public contract is focused on creation and execution of validation logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| lowerBound(min, max, inclusive) | IntRangeBoundValidator | O(1) | Factory method. Creates a validator for the minimum bound of an IntRange. |
| upperBound(min, max, inclusive) | IntRangeBoundValidator | O(1) | Factory method. Creates a validator for the maximum bound of an IntRange. |
| accept(intRange, results) | void | O(1) | Executes the validation logic against the provided IntRange, populating the results object. |
| updateSchema(context, target) | void | O(1) | Modifies the target Schema to reflect the validator's constraints. **Warning:** Throws IllegalArgumentException if the target schema is not a compatible type. |

## Integration Patterns

### Standard Usage
This validator is intended to be used within the Hytale Codec and Schema definition system. It is typically attached to a field definition to enforce constraints.

```java
// Example: Defining a schema for an item's damage range
// where the minimum damage must be between 1 and 100.

Schema itemSchema = new ObjectSchema()
    .property("damage", new ArraySchema(
        new IntegerSchema(),
        new IntegerSchema()
    ).withValidator(
        // The validator is created and attached here
        IntRangeBoundValidator.lowerBound(1, 100, true)
    ));

// Later, when data is validated...
ValidationResults results = new ValidationResults();
IntRange damageRange = new IntRange(0, 50); // Min bound is 0, which is invalid
itemSchema.getValidator("damage").accept(damageRange, results);

if (results.hasFailed()) {
    // Handle validation failure: "Min bound must be greater than or equal to 1"
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is private. Do not attempt to use reflection to create an instance. Always use the `lowerBound` or `upperBound` static factory methods.
- **Schema Mismatch:** Calling `updateSchema` on a Schema that is not an ArraySchema containing at least two IntegerSchema elements will result in a runtime `IllegalArgumentException`. The validator is tightly coupled to the expected structure of an IntRange representation.
- **Null Inputs:** While the `accept` method is null-safe for the IntRange parameter (it simply does nothing), passing a null `ValidationResults` object will cause a `NullPointerException`.

## Data Pipeline
The IntRangeBoundValidator participates in two distinct data flows: runtime validation and schema generation.

**Runtime Validation Flow**
> Flow:
> Raw Data (e.g., JSON) -> Deserializer -> IntRange object -> **IntRangeBoundValidator.accept()** -> ValidationResults

**Schema Generation Flow**
> Flow:
> Code-based Schema Definition -> **IntRangeBoundValidator.updateSchema()** -> Modified Schema Object -> Schema Serializer -> JSON Schema File

