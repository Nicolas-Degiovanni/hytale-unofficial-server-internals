---
description: Architectural reference for ThrowingValidationResults
---

# ThrowingValidationResults

**Package:** com.hypixel.hytale.codec.validation
**Type:** Transient

## Definition
```java
// Signature
public class ThrowingValidationResults extends ValidationResults {
```

## Architecture & Concepts
ThrowingValidationResults is a concrete implementation of the ValidationResults strategy pattern. It enforces a strict, **fail-fast** validation policy within the Hytale Codec framework.

Its primary architectural role is to immediately terminate a data deserialization process the moment a critical validation error is detected. Unlike other ValidationResults implementations that might aggregate multiple errors for later review, this class halts execution by throwing a checked CodecValidationException. This design is critical for systems where data integrity is non-negotiable, such as loading core game assets or processing critical network packets. If the data is not perfectly valid, the system considers it corrupted and aborts the operation to prevent undefined behavior downstream.

For non-critical issues (warnings), it logs the information without throwing, allowing the process to continue but ensuring the issue is recorded.

### Lifecycle & Ownership
- **Creation:** Instantiated on-the-fly by higher-level systems like an AssetLoader or a network message decoder. The creator must provide an ExtraInfo object, which supplies the necessary context for generating a detailed and actionable error message.
- **Scope:** The object's lifetime is extremely short, scoped to a single data validation operation. It is created, used for one or more calls to the add method, and then immediately becomes eligible for garbage collection.
- **Destruction:** There is no explicit destruction. The object is discarded after its `add` method either throws an exception or returns.

## Internal State & Concurrency
- **State:** This class is effectively stateless. While it holds a reference to an ExtraInfo object, it does not accumulate results or modify its internal state across multiple calls. Each invocation of the `add` method is a self-contained, terminal operation.
- **Thread Safety:** **Not thread-safe.** This class is designed to be used exclusively within the context of a single-threaded data processing task. The internal use of StringBuilder and its immediate-throw nature make it unsuitable for shared use across multiple threads.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| add(ValidationResult result) | void | O(1) | Processes a single validation result. Throws a CodecValidationException if the result represents a failure. Logs a warning if the result is non-critical. |

## Integration Patterns

### Standard Usage
This class should be used within a try-catch block by any system performing data deserialization that requires strict validation. The caller is responsible for handling the CodecValidationException to gracefully manage the failed loading process.

```java
// A codec or asset loader using the fail-fast strategy
ValidationResults validator = new ThrowingValidationResults(currentContextInfo);
try {
    // ... logic that calls validator.add() internally ...
    assetDecoder.decode(inputStream, validator);
    log.info("Asset successfully validated and loaded.");
} catch (CodecValidationException e) {
    log.error("Failed to load asset due to validation error. Aborting.", e);
    // Rollback or unload partially loaded data
}
```

### Anti-Patterns (Do NOT do this)
- **Swallowing Exceptions:** Catching CodecValidationException and ignoring it defeats the entire purpose of this class. If you do not intend to halt execution on an error, use a different ValidationResults implementation.
- **Reusing Instances:** Do not reuse an instance of ThrowingValidationResults across different deserialization operations. A new instance should be created for each distinct asset or data structure being validated to ensure error context is accurate.

## Data Pipeline
ThrowingValidationResults acts as a terminal gate in a data processing pipeline. Upon failure, it short-circuits the entire flow.

> Flow:
> Serialized Data (File, Network) -> Deserializer -> Validation Logic -> **ThrowingValidationResults.add()** -> Throws CodecValidationException (Halts Pipeline) **OR** Logs Warning (Pipeline Continues)

