---
description: Architectural reference for LateValidator
---

# LateValidator<T>

**Package:** com.hypixel.hytale.codec.validation
**Type:** Contract / Interface

## Definition
```java
// Signature
public interface LateValidator<T> extends Validator<T> {
```

## Architecture & Concepts
The LateValidator interface defines a contract for the second phase of a two-pass data validation pipeline within the codec system. It extends the base Validator interface, inheriting its fundamental validation contract, but introduces a distinct entry point, *acceptLate*, for deferred validation checks.

This two-phase system is critical for validating complex object graphs where the validity of one object depends on the state of another. The first pass, handled by standard Validator implementations, performs intrinsic, self-contained checks on individual objects as they are deserialized. The second pass, executed via LateValidator implementations, performs extrinsic, cross-object, or context-dependent validations after the entire object graph has been initially constructed.

This architecture decouples simple validation from complex, relational validation, preventing circular dependency issues and ensuring that all necessary data is available before performing checks that span multiple objects.

### Lifecycle & Ownership
As an interface, LateValidator itself is a stateless contract and has no lifecycle. The lifecycle and ownership pertain to the concrete classes that implement this interface.

- **Creation:** Implementations are typically stateless singletons. They are discovered and instantiated at application startup by a service locator or dependency injection framework, often through classpath scanning for specific annotations.
- **Scope:** An implementation's instance is scoped to the application's lifetime. It is registered with a central Codec or Validation registry and persists until shutdown.
- **Destruction:** The instances are destroyed during application shutdown when the central registry is cleared.

## Internal State & Concurrency
- **State:** The LateValidator contract implicitly requires all implementations to be **stateless**. Storing any mutable state within a validator instance is a severe anti-pattern that will lead to unpredictable behavior and data corruption, as the same instance is used across multiple threads to validate different data streams concurrently.
- **Thread Safety:** Implementations **must be unconditionally thread-safe**. The validation engine may invoke `acceptLate` on the same validator instance from multiple worker threads simultaneously. All logic within the method must be re-entrant and free of side effects.

## API Surface
The primary contract is the `acceptLate` method, which is invoked by the validation engine during the second validation pass.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| acceptLate(T, ValidationResults, ExtraInfo) | void | O(N) | Executes deferred validation logic on the target object. Populates the ValidationResults container with any discovered errors or warnings. |

## Integration Patterns

### Standard Usage
Developers do not typically invoke a LateValidator directly. Instead, they provide a concrete implementation for a specific data type, which is then automatically discovered and executed by the codec's validation engine.

The primary integration pattern is implementation.

```java
// How a developer should implement this contract
public class MyComplexObjectLateValidator implements LateValidator<MyComplexObject> {

    @Override
    public void acceptLate(MyComplexObject object, ValidationResults results, ExtraInfo extraInfo) {
        // This logic runs after all objects in the graph have passed initial validation.
        // Example: Validate that a reference to another object is not null and is in a valid state.
        if (object.getRelatedEntityId() != null) {
            RelatedEntity entity = extraInfo.getGraphContext().findById(object.getRelatedEntityId());
            if (entity == null) {
                results.addError("Dangling reference: RelatedEntity not found.");
            }
        }
    }

    // Implementation of the base Validator.accept() method is also required.
    @Override
    public void accept(MyComplexObject object, ValidationResults results, ExtraInfo extraInfo) {
        // This runs first. Perform only intrinsic checks here.
        if (object.getName() == null || object.getName().isEmpty()) {
            results.addError("Name cannot be empty.");
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Storing session-specific or request-specific data in member variables of a validator implementation. This will cause catastrophic race conditions.
- **Incorrect Phase Logic:** Placing intrinsic, self-contained validation logic inside `acceptLate`. This logic belongs in the `accept` method of the base Validator interface. The `acceptLate` method is reserved exclusively for checks that require a broader context which is only available after the initial deserialization and validation pass is complete.
- **Manual Invocation:** Manually creating and calling a LateValidator. The validation pipeline is managed by the codec system, which guarantees the correct ordering and context. Bypassing the engine will lead to incomplete or incorrect validation.

## Data Pipeline
The LateValidator is the final programmatic step in the data validation pipeline before an object is considered fully hydrated and trusted by the game engine.

> Flow:
> Serialized Data Stream -> Codec Deserialization -> Phase 1 Validation (Validator.accept) -> **Phase 2 Validation (LateValidator.acceptLate)** -> Fully Validated Object Graph

