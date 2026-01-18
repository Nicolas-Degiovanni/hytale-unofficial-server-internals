---
description: Architectural reference for DoubleArraySizeValidator
---

# DoubleArraySizeValidator

**Package:** com.hypixel.hytale.codec.validation.validator
**Type:** Transient

## Definition
```java
// Signature
public class DoubleArraySizeValidator implements Validator<double[]> {
```

## Architecture & Concepts
The DoubleArraySizeValidator is a specific, single-purpose component within the Hytale Codec and Schema Validation framework. It embodies a concrete implementation of the *Strategy Pattern*, where the Validator interface defines a generic contract for data validation, and this class provides the specific strategy for enforcing a fixed size on a double array.

Its role is twofold:
1.  **Runtime Validation:** During data deserialization or processing, it checks if a given double array instance conforms to a predefined size.
2.  **Schema Generation:** It participates in the construction of a formal data schema by annotating the schema definition with constraints (minItems, maxItems). This ensures that the generated schema accurately reflects the runtime validation rules, which is critical for network clients, tooling, and documentation.

This class acts as a low-level building block, intended to be composed by higher-level schema builders rather than used directly.

### Lifecycle & Ownership
- **Creation:** Instantiated by a schema configuration or validation builder when a fixed-size constraint is declared for a double array. A new instance is created for each unique size constraint.
- **Scope:** The lifecycle of a DoubleArraySizeValidator instance is tied to the lifecycle of the schema or validation rule set that contains it. It is a lightweight, short-lived object.
- **Destruction:** The object is eligible for garbage collection as soon as the parent schema or validator configuration is discarded. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** The internal state, consisting of the target array size, is **immutable**. It is set once via the constructor and never changes. The class is therefore stateless from an operational perspective.
- **Thread Safety:** This class is **inherently thread-safe**. Its immutable state and lack of side effects (other than modifying the passed-in ValidationResults object) allow the accept and updateSchema methods to be called concurrently from multiple threads without synchronization.

## API Surface
The public contract is minimal and focused on its role within the validation framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| accept(double[], ValidationResults) | void | O(1) | Executes the size validation. If the check fails, it registers a failure in the provided ValidationResults object. |
| updateSchema(SchemaContext, Schema) | void | O(1) | Modifies the target ArraySchema to enforce a fixed item count, aligning the schema with the validation logic. |

## Integration Patterns

### Standard Usage
Direct interaction with this class is uncommon. It is designed to be used declaratively by a higher-level schema definition system. The framework is responsible for instantiating and invoking it during the validation process.

```java
// Conceptual Example: How the framework would use this validator
// A developer would typically not write this code directly.

// 1. A schema builder creates the validator internally
Validator<double[]> sizeValidator = new DoubleArraySizeValidator(3);

// 2. The framework invokes it during data processing
double[] incomingData = new double[]{1.0, 2.0}; // Incorrect size
ValidationResults results = new ValidationResults();
sizeValidator.accept(incomingData, results);

if (results.hasFailed()) {
    // Handle validation failure...
}
```

### Anti-Patterns (Do NOT do this)
- **Misuse for Variable-Size Arrays:** This validator is exclusively for arrays with a single, exact, required size. Do not use it for arrays that have a minimum or maximum size range; other validators exist for that purpose.
- **Ignoring ValidationResults:** Calling the accept method but failing to inspect the ValidationResults object renders the validation pointless. The validator does not throw exceptions on failure; it reports failures via the results object.

## Data Pipeline
This validator operates within two primary data flows: runtime validation and schema generation.

**Runtime Validation Flow:**
> Flow:
> Raw Data (e.g., Network Packet) -> Deserializer -> `double[]` instance -> **DoubleArraySizeValidator.accept()** -> ValidationResults -> Application Logic

**Schema Generation Flow:**
> Flow:
> Schema Builder -> **DoubleArraySizeValidator.updateSchema()** -> In-Memory Schema Object -> Schema Serializer -> Formal Schema (e.g., JSON)

