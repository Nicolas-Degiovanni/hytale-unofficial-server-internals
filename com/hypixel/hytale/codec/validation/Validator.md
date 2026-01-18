---
description: Architectural reference for Validator
---

# Validator<T>

**Package:** com.hypixel.hytale.codec.validation
**Type:** Functional Interface / Strategy

## Definition
```java
// Signature
public interface Validator<T> extends BiConsumer<T, ValidationResults> {
```

## Architecture & Concepts
The Validator interface is a fundamental component of Hytale's data serialization and schema-driven configuration system, known as the *codec* framework. It embodies the Strategy Pattern, representing a single, reusable rule for validating the integrity of a deserialized object of type T.

Its primary role is to decouple the logic of data validation from the data structures themselves. Instead of embedding validation checks within game objects, developers compose a set of Validator implementations and attach them to a Schema. The codec engine then automatically invokes these validators during the data deserialization pipeline, ensuring that all incoming data—whether from network packets, save files, or configuration—adheres to a strict contract.

This interface is not meant to be used in isolation. It operates within a larger system orchestrated by a SchemaContext, which manages the overall structure, and a Codec, which handles the actual data transformation. The presence of the `updateSchema` method indicates a highly dynamic and reflective system where validation rules can programmatically contribute metadata to the schema they are part of.

## Lifecycle & Ownership
- **Creation:** Implementations of Validator are typically defined declaratively, often as lambdas or method references, during the construction of a Schema. They are not managed by a dependency injection container or service locator. They are configuration artifacts.
- **Scope:** The lifecycle of a Validator instance is strictly tied to the Schema or Codec that defines it. It exists for as long as its owning configuration is held in memory.
- **Destruction:** Instances are subject to standard Java garbage collection. They are destroyed when the owning Schema is no longer reachable.

## Internal State & Concurrency
- **State:** The Validator contract is inherently stateless. Implementations **must** be designed to be stateless and fully re-entrant. Storing mutable state within a Validator implementation is a severe anti-pattern that will lead to unpredictable behavior in a concurrent environment.
- **Thread Safety:** The interface itself is thread-safe. However, the responsibility for thread safety lies with the implementer. Given that the codec system may be used by multiple network or worker threads simultaneously, all Validator implementations **must** be thread-safe. The recommended approach is to implement them as pure functions whose output depends solely on their inputs.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| accept(T, ValidationResults) | void | O(N) | Executes the validation logic against the target object. Failures are reported by adding errors to the ValidationResults collection. This method should not throw exceptions for validation failures. |
| updateSchema(SchemaContext, Schema) | void | O(1) | A callback invoked during the schema-building phase. Allows the validator to programmatically add metadata or constraints to the target schema. For example, a range validator could add *minimum* and *maximum* attributes. |
| late() | LateValidator<T> | O(1) | Returns a new LateValidator that wraps the current instance. This is a critical adapter for deferring validation to a second pass, which is necessary for validating inter-object references that may not be resolved during the initial deserialization. |

## Integration Patterns

### Standard Usage
A developer does not invoke a Validator directly. Instead, it is attached to a field or type definition within the schema-building DSL. The codec framework is responsible for its invocation.

```java
// Hypothetical schema definition
// The developer provides the validation logic (e.g., a method reference).
// The system invokes it during deserialization.
Schema<PlayerProfile> profileSchema = Schema.create(PlayerProfile.class, schema -> {
    schema.field("username", HytaleCodecs.STRING)
          .withValidator(Validators::isNonEmpty); // Attaching a validator

    schema.field("level", HytaleCodecs.INTEGER)
          .withValidator(Validators.inRange(1, 100));
});
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Do not create validators that rely on mutable instance variables. This will fail catastrophically under concurrent loads.
- **Throwing Exceptions:** Do not throw exceptions to report validation failures. The system is designed to aggregate multiple errors into the ValidationResults object. Throwing an exception will prematurely terminate the entire validation process.
- **Manual Invocation:** Do not call the `accept` method directly. The timing and context of validation are managed by the codec engine, especially concerning multi-pass validation via LateValidator.

## Data Pipeline
The Validator participates in two distinct data flows: one at configuration time and one at runtime.

**Configuration-Time Flow (Schema Building):**
> Flow:
> Schema Definition DSL -> **Validator.updateSchema()** -> Finalized Schema Metadata -> Codec Construction

**Runtime Flow (Deserialization):**
> Flow:
> Raw Data (e.g., Network Buffer) -> Codec Deserialization -> Object Graph Instantiation -> **Validator.accept()** -> Aggregated ValidationResults -> Application Logic<ctrl63>

