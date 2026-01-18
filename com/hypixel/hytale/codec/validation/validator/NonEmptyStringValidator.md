---
description: Architectural reference for NonEmptyStringValidator
---

# NonEmptyStringValidator

**Package:** com.hypixel.hytale.codec.validation.validator
**Type:** Singleton / Utility

## Definition
```java
// Signature
public class NonEmptyStringValidator implements Validator<String> {
```

## Architecture & Concepts
The NonEmptyStringValidator is a foundational component within the Hytale Codec framework, responsible for enforcing a single, critical business rule: a string must contain at least one non-whitespace character. It embodies the Strategy pattern, providing a concrete, reusable implementation of the Validator interface.

Architecturally, this class serves two distinct but related purposes:

1.  **Runtime Data Validation:** Its primary role is to inspect a string value at runtime via the *accept* method and report failures.
2.  **Schema Definition Enrichment:** Through the *updateSchema* method, it programmatically enhances a schema definition to reflect its validation constraints. This allows the framework to generate more precise schemas (e.g., for API documentation, client-side form validation, or database constraints) that are guaranteed to be in sync with the server-side validation logic.

The use of the Singleton pattern, exposed via the public static final INSTANCE field, is intentional. As the validator is completely stateless, a single shared instance prevents unnecessary object allocation, improving performance in high-throughput validation scenarios.

### Lifecycle & Ownership
-   **Creation:** The singleton INSTANCE is instantiated by the JVM class loader when the NonEmptyStringValidator class is first referenced. There is no manual creation step performed by the application's bootstrap sequence.
-   **Scope:** As a static final field, the INSTANCE persists for the entire lifetime of the application.
-   **Destruction:** The object is eligible for garbage collection only when the application is shutting down and its class loader is unloaded by the JVM.

## Internal State & Concurrency
-   **State:** This class is **immutable and stateless**. It holds no instance-level data. The NON_WHITESPACE_PATTERN is a static final constant, shared across all interactions. All operations are performed exclusively on the arguments passed into its methods.
-   **Thread Safety:** This class is **inherently thread-safe**. Due to its stateless design, the public INSTANCE can be safely shared and invoked from any number of concurrent threads without requiring external synchronization or locks.

## API Surface
The public contract is minimal and focused, adhering to the Validator interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| accept(string, results) | void | O(N) | Validates that the input string is not blank. Adds a failure to the results if the check fails. N is the length of the string. |
| updateSchema(context, target) | void | O(1) | Modifies the target Schema to enforce a minimum length of 1 and a non-whitespace pattern. **Warning:** Throws ClassCastException if the target is not a StringSchema. |

## Integration Patterns

### Standard Usage
This validator is not intended to be invoked directly. Instead, it should be registered with a schema builder or a validation service, which will then manage its lifecycle and invocation.

```java
// Correct usage: Registering the validator with a schema definition
StringSchema usernameSchema = new StringSchema();

// The framework will later use this validator to check data
// and can use its updateSchema method to configure itself.
usernameSchema.addValidator(NonEmptyStringValidator.INSTANCE);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** The constructor is protected and should not be invoked. Always use the provided `NonEmptyStringValidator.INSTANCE`. Creating new instances defeats the purpose of the singleton optimization.
-   **Manual Invocation:** Avoid calling the *accept* method directly in application code. Let the parent validation framework manage the validation pipeline. Manual invocation can lead to inconsistent error handling and bypasses other potential validation steps.
-   **Incorrect Schema Type:** Passing a non-StringSchema object to the *updateSchema* method will result in a runtime ClassCastException. The framework is expected to enforce this contract, but developers extending the system must be aware of this constraint.

## Data Pipeline
NonEmptyStringValidator acts as a single processing stage within a larger data validation and deserialization pipeline. It does not produce data but rather acts as a gatekeeper, signaling failure to an aggregator object.

> Flow:
> Raw Data (JSON, Binary) -> Deserializer -> String Object -> Validation Service -> **NonEmptyStringValidator.accept()** -> ValidationResults -> Application Logic

