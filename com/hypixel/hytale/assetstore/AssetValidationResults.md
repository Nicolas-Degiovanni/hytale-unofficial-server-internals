---
description: Architectural reference for AssetValidationResults
---

# AssetValidationResults

**Package:** com.hypixel.hytale.assetstore
**Type:** Transient

## Definition
```java
// Signature
public class AssetValidationResults extends ValidationResults {
```

## Architecture & Concepts
The AssetValidationResults class is a specialized state container responsible for aggregating, interpreting, and reporting outcomes during the validation of a single game asset. It extends the more generic ValidationResults, adding context-specific logic for handling asset-related issues, most notably missing asset references.

Its primary architectural role is to decouple the process of *detecting* a validation failure from the policy of *handling* that failure. Instead of a validator immediately throwing a fatal exception, it reports the issue to an AssetValidationResults instance. This instance then makes a final determination based on its internal configuration and the execution environment.

A key feature is its ability to selectively suppress exceptions for missing assets of a certain type via the disableMissingAssetFor method. This allows the asset pipeline to treat certain broken references as non-fatal, which is critical for development workflows and content iteration.

Furthermore, the class contains environment-aware logging. It detects if it is running within a GitHub Actions environment and, if so, formats its output using special annotations that integrate directly with the GitHub UI, pointing developers to the exact file and line number of an issue.

## Lifecycle & Ownership
- **Creation:** An instance is created by the asset loading framework immediately before the deserialization and validation of a single asset file begins. It is initialized with an ExtraInfo object, which provides metadata about the asset being processed.
- **Scope:** The object's lifetime is strictly limited to the validation scope of one asset. It is a short-lived, single-use object.
- **Destruction:** It is eligible for garbage collection as soon as the validation process for the asset is complete and the results have been logged or thrown. It does not persist beyond the loading operation.

## Internal State & Concurrency
- **State:** This class is highly mutable. Its primary state consists of a collection of validation errors inherited from its parent and an internal Set, disabledMissingAssetClasses, which tracks which asset types should not trigger a MissingAssetException. The state is progressively built up as an asset is parsed.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be used exclusively by a single thread processing a single asset. The internal collections, such as HashSet, are not synchronized. Concurrent access will lead to unpredictable behavior and is strictly unsupported.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| handleMissingAsset(field, assetType, assetId) | void | O(1) | Checks if exceptions for the given assetType are disabled. Throws MissingAssetException if not. |
| disableMissingAssetFor(assetType) | void | O(1) | Configures the instance to ignore missing asset errors for the specified class. |
| logOrThrowValidatorExceptions(logger, msg, path, lineOffset) | void | O(N) | Processes all collected validation results. Throws an exception if any fatal errors exist. Formats and logs warnings and errors, with special handling for GitHub environments. N is the number of validation results. |

## Integration Patterns

### Standard Usage
This class is intended to be used as part of a larger asset loading and validation pipeline. The orchestrator of the pipeline is responsible for its lifecycle.

```java
// A system responsible for loading an asset would use it like this.
AssetExtraInfo<?> metadata = createAssetMetadata(assetPath);
AssetValidationResults results = new AssetValidationResults(metadata);

// Allow a specific type of missing asset to not be a fatal error.
results.disableMissingAssetFor(Decoration.class);

// The deserializer uses the results object to report issues.
JsonAsset asset = deserializer.load(assetStream, results);

// After processing, check the results and report them.
// This will throw if fatal errors were found.
results.logOrThrowValidatorExceptions(logger, "Validation failed for " + assetPath, assetPath, 0);
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not reuse an AssetValidationResults instance for validating more than one asset file. Its state is specific to a single operation and will cause incorrect error reporting if carried over.
- **Concurrent Modification:** Do not share an instance across multiple threads. All calls to report or configure results must be made from the same thread.
- **Premature Logging:** Do not call logOrThrowValidatorExceptions before the asset has been fully parsed. This method is a terminal operation that should only be called once all validation is complete.

## Data Pipeline
AssetValidationResults acts as a terminal sink in the asset validation data flow. It does not transform data but rather collects metadata about failures that occur during transformation and validation.

> Flow:
> Asset File (JSON) -> Deserializer -> Validation Logic -> **AssetValidationResults** (collects errors) -> Exception or Logger Backend

