---
description: Architectural reference for ValidationResult
---

# ValidationResult

**Package:** com.hypixel.hytale.protocol.io
**Type:** Utility

## Definition
```java
// Signature
public record ValidationResult(boolean isValid, @Nullable String error) {
```

## Architecture & Concepts
The ValidationResult is an immutable value object designed to encapsulate the outcome of a data validation operation. It serves as a standardized data carrier within the protocol and I/O layers, providing a clear and explicit representation of success or failure without immediately resorting to exceptions for control flow.

This class promotes a functional approach to error handling. Instead of a method throwing an exception on invalid input, it returns a ValidationResult object. The caller can then inspect this result and decide how to proceed. The `throwIfInvalid` method provides a convenient bridge to an exception-based model, allowing a consumer to immediately halt execution upon receiving an invalid result.

Its primary role is to decouple the validation logic from the error-handling policy.

### Lifecycle & Ownership
- **Creation:** Instances are created on-demand by validation routines. The canonical way to create instances is via the static `OK` constant for success or the `error(String)` factory method for failure.
- **Scope:** ValidationResult objects are transient and short-lived. They typically exist only on the stack, returned from a validation method to its immediate caller, and are eligible for garbage collection once inspected.
- **Destruction:** As a simple record, it is managed entirely by the Java Garbage Collector. There are no external resources to release.

## Internal State & Concurrency
- **State:** This class is a Java record, making it fundamentally immutable. Its state, consisting of the `isValid` boolean and the `error` string, is finalized upon construction and cannot be modified.
- **Thread Safety:** Due to its immutability, ValidationResult is unconditionally thread-safe. Instances can be freely shared and passed between threads without any need for external synchronization or locking mechanisms.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| OK | static ValidationResult | O(1) | A shared, static instance representing a successful validation. |
| error(String) | static ValidationResult | O(1) | Factory method to create a new instance representing a failed validation with a specific error message. |
| throwIfInvalid() | void | O(1) | Checks the internal state. Throws a ProtocolException if the result is not valid. This is the primary mechanism for converting a failed validation into a control-flow exception. |

## Integration Patterns

### Standard Usage
The intended pattern is for a validation method to return a ValidationResult, which the caller then inspects or uses to conditionally throw an exception.

```java
// A component performs validation and returns a result
public ValidationResult validatePacket(PacketData data) {
    if (data.getLength() > MAX_LENGTH) {
        return ValidationResult.error("Packet exceeds maximum length.");
    }
    return ValidationResult.OK;
}

// The consumer of the validation logic checks the result
ValidationResult result = validator.validatePacket(someData);
result.throwIfInvalid(); // Halts execution if validation failed

// If execution continues, the data is considered valid
processValidPacket(someData);
```

### Anti-Patterns (Do NOT do this)
- **Ignoring the Return Value:** A critical error is to call a validation method and fail to check the returned ValidationResult. This completely bypasses the validation check.
- **Direct Instantiation:** While `new ValidationResult(true, null)` is legal, developers must use the provided static `OK` constant and `error` factory method. This ensures consistency and leverages the shared `OK` instance.
- **Null Error Messages:** Do not pass a null message to the `error` factory. The method is annotated with Nonnull to prevent this.

## Data Pipeline
ValidationResult is not a pipeline component itself, but rather the terminal output of a validation stage within a larger data processing pipeline.

> Flow:
> Network Byte Stream -> Packet Deserializer -> Packet Validator -> **ValidationResult** -> Application Logic

