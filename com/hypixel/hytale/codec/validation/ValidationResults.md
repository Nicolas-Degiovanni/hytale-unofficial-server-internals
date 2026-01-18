---
description: Architectural reference for ValidationResults
---

# ValidationResults

**Package:** com.hypixel.hytale.codec.validation
**Type:** Transient

## Definition
```java
// Signature
public class ValidationResults {
```

## Architecture & Concepts
The ValidationResults class is a stateful container designed to accumulate and process the outcomes of data validation operations, primarily within the asset codec and serialization systems. It functions as a "report card" for a single, overarching validation task, such as parsing an entire asset file.

Its core architectural purpose is to decouple the *detection* of validation issues from the *handling* of those issues. Instead of throwing an exception at the first sign of an error, validators can register failures or warnings with a ValidationResults instance. This allows the system to gather a comprehensive list of all problems within a data structure before making a terminal decision—either throwing a single, detailed exception or logging a consolidated warning.

This pattern is critical for providing high-quality feedback to content creators, as it presents all errors at once rather than forcing a frustrating fix-compile-fail loop for each individual issue. The class leverages an `ExtraInfo` object to maintain context, such as file paths and line numbers, ensuring that final error messages are precise and actionable.

## Lifecycle & Ownership
- **Creation:** A ValidationResults instance is created at the beginning of a validation-sensitive process. Typically, a higher-level manager like an AssetLoader or a top-level Codec instantiates it, providing the necessary `ExtraInfo` context for the asset being processed.
- **Scope:** The object's lifetime is scoped to a single, complete validation operation. It is passed down through the call stack to various sub-validators and processors, accumulating results along the way. It is not intended to be a long-lived object.
- **Destruction:** The object holds no native resources and does not require explicit cleanup. It becomes eligible for garbage collection once the final `logOrThrowValidatorExceptions` method has been called and all references to the instance are out of scope.

## Internal State & Concurrency
- **State:** This class is highly mutable. Its primary function is to aggregate state within its internal lists, `results` and `validatorExceptions`. These lists are lazily instantiated to minimize memory allocation for valid assets that produce no warnings or errors. The state transitions from collecting individual results to grouping them into `ValidatorResultsHolder` records.
- **Thread Safety:** **This class is not thread-safe.** All internal collections are standard, unsynchronized lists. Concurrent calls to methods like `add`, `fail`, or `_processValidationResults` from multiple threads will result in race conditions, potential `ConcurrentModificationException`s, and an unpredictable final report. It is designed to be confined to the single thread responsible for validating a specific asset.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| fail(String reason) | void | O(1) amortized | Adds a FAIL result to the internal list. Lazily initializes the list if necessary. |
| warn(String reason) | void | O(1) amortized | Adds a WARNING result to the internal list. Lazily initializes the list if necessary. |
| _processValidationResults() | void | O(N) | Consolidates currently collected results into a `ValidatorResultsHolder`. This is an internal state transition method. |
| logOrThrowValidatorExceptions(HytaleLogger logger, String msg) | void | O(M) | Terminal operation. Builds a detailed report string from all processed results. Throws `CodecValidationException` if any failures were recorded; otherwise, logs warnings. |
| hasFailed() | boolean | O(N) | Scans the current, unprocessed results to check for any FAIL outcomes. |

## Integration Patterns

### Standard Usage
The typical lifecycle involves creating an instance, passing it through one or more validation stages, and concluding with a terminal call to check the results.

```java
// 1. A service creates the results object with context
ExtraInfo context = new ExtraInfo("assets/models/item.json");
ValidationResults results = new ValidationResults(context);

// 2. It is passed to various validators
ItemValidator itemValidator = new ItemValidator();
itemValidator.validate(data, results); // Validator calls results.fail() or results.warn() internally

// 3. The service processes and checks the results
results._processValidationResults(); // Consolidate findings
results.logOrThrowValidatorExceptions(assetLogger, "Validation failed for item.json");
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not re-use a ValidationResults instance for a new, unrelated validation task. The internal state and the initial `ExtraInfo` context are tied to the first operation and will produce misleading error reports. Always create a new instance.
- **Ignoring Terminal Operation:** Failing to call `logOrThrowValidatorExceptions` will silently discard all collected validation issues. The system will incorrectly assume the asset was valid.
- **Concurrent Access:** Do not share a single ValidationResults instance across multiple threads processing different parts of an asset simultaneously. This will corrupt its internal state.

## Data Pipeline
The flow of data through this component is designed to transform discrete validation events into a single, consolidated report or exception.

> Flow:
> Validator Logic → `fail()` / `warn()` → **Internal `results` List** → `_processValidationResults()` → **Internal `validatorExceptions` List** → `logOrThrowValidatorExceptions()` → `CodecValidationException` OR Log Entry

