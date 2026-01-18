---
description: Architectural reference for ValidateIfEnumIsValidator
---

# ValidateIfEnumIsValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Transient Strategy Object

## Definition
```java
// Signature
public class ValidateIfEnumIsValidator<E extends Enum<E> & Supplier<String>> extends Validator {
```

## Architecture & Concepts

The ValidateIfEnumIsValidator is a higher-order, conditional validator within the server's NPC asset validation framework. It does not perform validation itself; instead, it acts as a logical gate, delegating to a nested Validator only if a specific condition is met. This implements a form of the Decorator or Strategy pattern, allowing for the construction of complex, context-aware validation rule sets.

Its primary role is to enforce internal consistency within an asset definition. For example, it can express a rule such as: "IF the parameter *animationType* is set to the enum value *LOOPING*, THEN apply a *NumberIsPositiveValidator* to the *loopCount* parameter."

The generic constraint, `E extends Enum<E> & Supplier<String>`, is critical. It mandates that any enum used with this validator must also implement the Supplier interface for a String. This pattern ensures that the enum can provide a canonical, serializable string representation of itself, which is used for comparison against the asset data.

This component is fundamental to creating robust and declarative asset specifications, preventing logical errors in NPC configurations before they are loaded by the game server.

### Lifecycle & Ownership
-   **Creation:** Instantiated exclusively through the static factory method `withAttributes`. This is typically invoked by a higher-level asset builder or a validation ruleset configurator when defining the validation pipeline for a specific NPC asset type.
-   **Scope:** Short-lived and stateless. An instance exists only for the duration of a single asset validation pass. It is created, its validation logic is invoked, and it is then eligible for garbage collection.
-   **Destruction:** Managed automatically by the Java garbage collector. No manual cleanup or resource release is necessary.

## Internal State & Concurrency
-   **State:** This class is **immutable**. All its fields are final and are assigned only during construction. It holds references to the conditional parameters and the nested validator but does not modify this state after creation.
-   **Thread Safety:** Inherently **thread-safe**. Due to its immutability, a single instance can be safely used across multiple threads to validate different asset files concurrently, provided the nested Validator instance is also thread-safe.

## API Surface

The public contract is focused entirely on its construction. The core validation logic is invoked through the `validate` method inherited from the parent Validator class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| withAttributes(p1, validator, p2, value) | ValidateIfEnumIsValidator | O(1) | Static factory method. Constructs a conditional validator instance. |

## Integration Patterns

### Standard Usage

This validator is designed to be composed with other validators to build a chain of rules. It is never used in isolation.

```java
// Example: Define a rule that validates 'loopCount' only if 'animationType' is 'LOOPING'.
// Assume AnimationType is an enum that implements Supplier<String>.

Validator conditionalRule = ValidateIfEnumIsValidator.withAttributes(
    "animationType",                      // The parameter to check
    new NumberIsPositiveValidator(),      // The validator to run if the condition is met
    "loopCount",                          // The parameter to validate
    AnimationType.LOOPING                 // The required enum value
);

// This 'conditionalRule' would then be added to a list of validators for the asset.
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** The constructor is private. Do not attempt to create an instance using reflection; always use the `withAttributes` static factory method.
-   **Null Arguments:** Providing null for any parameter to `withAttributes` is unsupported and will lead to a NullPointerException during the validation process. All parameters are considered mandatory.
-   **Stateful Nested Validators:** Nesting a validator that is not thread-safe or that maintains state across invocations will break the immutability and concurrency guarantees of this class.

## Data Pipeline

This component functions as a single step within a larger asset deserialization and validation pipeline. It does not perform I/O or manage external resources.

> Flow:
> NPC Asset File (JSON) -> Jackson Deserializer -> Raw Data Object -> Validation Engine -> **ValidateIfEnumIsValidator** -> Validation Result (Success/Failure)

