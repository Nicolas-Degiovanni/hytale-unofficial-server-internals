---
description: Architectural reference for NonNullValidator
---

# NonNullValidator

**Package:** com.hypixel.hytale.codec.validation.validator
**Type:** Singleton

## Definition
```java
// Signature
public class NonNullValidator<T> implements Validator<T> {
```

## Architecture & Concepts
The NonNullValidator is a foundational component within the Hytale data codec framework. It represents the most atomic and common validation rule: ensuring a value is not null. As an implementation of the generic Validator interface, its purpose is to be a stateless, reusable predicate that can be composed into more complex validation schemas.

Architecturally, this class embodies the Singleton pattern. Because its validation logic is pure and carries no state between invocations, a single shared instance is sufficient for the entire application. This design minimizes memory overhead and object allocation, which is critical in performance-sensitive systems like a game engine's data serialization layer. It is a terminal link in a validation chain, designed to be invoked by a higher-level validation orchestrator.

## Lifecycle & Ownership
- **Creation:** The single instance is created eagerly during class loading via the static final field `INSTANCE`. It is not managed by a dependency injection container and exists before any application logic runs.
- **Scope:** Application-wide. The singleton instance persists for the entire lifetime of the JVM process.
- **Destruction:** The instance is eligible for garbage collection only when its class loader is unloaded, which typically occurs at application shutdown. No explicit cleanup is required.

## Internal State & Concurrency
- **State:** This class is completely stateless and immutable. It contains no instance fields and its behavior is solely dependent on the arguments passed to its methods.
- **Thread Safety:** Inherently thread-safe. Due to its stateless nature, the `accept` method can be invoked concurrently by any number of threads without risk of data corruption or race conditions. No synchronization primitives are necessary.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| accept(T, ValidationResults) | void | O(1) | Executes the null check. If the input object is null, a failure is recorded in the provided ValidationResults object. |
| updateSchema(SchemaContext, Schema) | void | O(1) | No-op. This implementation does not contribute any information to the schema definition. |

## Integration Patterns

### Standard Usage
This validator is not intended to be invoked directly in business logic. Instead, it should be retrieved via its static `INSTANCE` field and registered with a schema builder or a validation service, which will manage its execution.

```java
// CORRECT: Registering the validator within a schema definition (hypothetical)
SchemaBuilder.forType(Player.class)
    .field("name")
    .withValidator(NonNullValidator.INSTANCE) // The validator is supplied to the framework
    .build();

// The framework later invokes the validator during a process like deserialization.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to create a new instance. The constructor is protected to prevent this, but reflection or package-local access would violate the design contract. Always use `NonNullValidator.INSTANCE`.
- **Redundant Checks:** Do not precede a call to a validation framework using this validator with a manual null check. This is redundant and defeats the purpose of the validation system.

## Data Pipeline
The NonNullValidator acts as a simple processing node within a larger data validation pipeline. It does not orchestrate data flow but rather acts upon the data it receives.

> Flow:
> Data Source (e.g., Network Packet, File) -> Deserializer -> Validation Orchestrator -> **NonNullValidator.accept()** -> ValidationResults -> Application Logic<ctrl63>

