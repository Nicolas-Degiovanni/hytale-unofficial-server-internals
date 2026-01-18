---
description: Architectural reference for NonEmptyDoubleArrayValidator
---

# NonEmptyDoubleArrayValidator

**Package:** com.hypixel.hytale.codec.validation.validator
**Type:** Singleton

## Definition
```java
// Signature
public class NonEmptyDoubleArrayValidator implements Validator<double[]> {
```

## Architecture & Concepts
The NonEmptyDoubleArrayValidator is a specialized, stateless component within the Hytale data codec and validation framework. Its primary function is to enforce a single, specific constraint: that an array of doubles must not be null and must contain at least one element.

As an implementation of the Validator interface, it is designed to be a pluggable rule in a larger validation pipeline. This allows it to be composed with other validators to build complex validation logic for configuration files, network packets, or any other structured data.

A key architectural feature is its dual responsibility. Beyond runtime data validation via the *accept* method, it also participates in schema definition through the *updateSchema* method. This allows the validation logic to be self-describing. When a schema is being generated or configured, this validator can programmatically update the schema definition (specifically, an ArraySchema) to reflect its constraint (minItems = 1). This bridges the gap between runtime enforcement and static data contracts, enabling tools to generate documentation, client-side validation, or UI forms that are aware of this "non-empty" rule without it being manually specified elsewhere.

## Lifecycle & Ownership
- **Creation:** The single instance is created statically by the JVM during class loading via the public static final INSTANCE field. Its lifecycle is not managed by any application container or dependency injection framework.
- **Scope:** As a stateless singleton, it has a global, application-wide scope. It persists for the entire runtime of the application.
- **Destruction:** The instance is eligible for garbage collection only when the application's class loader is unloaded, typically during JVM shutdown. No manual cleanup is required.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It contains no instance fields and its behavior depends solely on the arguments passed to its methods.
- **Thread Safety:** The class is inherently **thread-safe**. The absence of mutable state guarantees that the singleton INSTANCE can be safely shared and invoked by multiple threads concurrently without any need for external synchronization or locks.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| accept(double[], ValidationResults) | void | O(1) | Executes the validation logic. If the input array is null or its length is zero, it registers a failure in the provided ValidationResults object. |
| updateSchema(SchemaContext, Schema) | void | O(1) | Modifies a schema definition to reflect the validator's constraint. Casts the target to an ArraySchema and sets its minimum item count to 1. Throws ClassCastException if the target is not an ArraySchema. |

## Integration Patterns

### Standard Usage
This validator is not intended to be invoked directly. Instead, it should be registered with a schema builder or a validation service, which will then apply it during its processing lifecycle.

```java
// Example: Applying the validator during schema definition
SchemaBuilder builder = new SchemaBuilder();

// The validator is registered with a field definition.
// The framework will later call accept() during validation
// and updateSchema() during schema generation.
builder.field("positions", double[].class)
       .withValidator(NonEmptyDoubleArrayValidator.INSTANCE);

Schema finalSchema = builder.build();
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is private. Do not attempt to create an instance using reflection. Always use the provided `NonEmptyDoubleArrayValidator.INSTANCE`.
- **Incorrect Schema Type:** Passing a Schema object to `updateSchema` that is not an instance of ArraySchema will cause a runtime ClassCastException. The framework integrating this validator is responsible for ensuring type compatibility.

## Data Pipeline
The validator operates in two distinct pipelines: the runtime data validation pipeline and the design-time schema generation pipeline.

**Runtime Validation Pipeline:**
> Flow:
> Raw Data (e.g., from file) -> Deserializer -> `double[]` object -> **NonEmptyDoubleArrayValidator.accept()** -> ValidationResults -> Application Logic

**Schema Generation Pipeline:**
> Flow:
> Schema Definition Code -> Schema Builder -> **NonEmptyDoubleArrayValidator.updateSchema()** -> Finalized Schema Object -> Schema Exporter (e.g., to JSON)

