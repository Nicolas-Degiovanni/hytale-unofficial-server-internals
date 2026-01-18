---
description: Architectural reference for MissingAssetException
---

# MissingAssetException

**Package:** com.hypixel.hytale.assetstore
**Type:** Transient Data Object

## Definition
```java
// Signature
public class MissingAssetException extends RuntimeException {
```

## Architecture & Concepts
The MissingAssetException is a specialized unchecked exception that signals a failure to resolve an asset reference during the data deserialization process. It is a critical component of the engine's asset integrity and validation system.

Its primary architectural significance lies not in its role as a simple exception, but in its static **handle** methods. These methods act as a strategic fork in control flow, allowing the system to differentiate between two critical operational modes:

1.  **Validation Mode:** During asset validation (e.g., in an editor or a build pipeline), a missing asset is a reportable error, but not a fatal one. The **handle** method detects a validation context (via AssetValidationResults) and delegates the error reporting to it. This allows the validation process to continue and accumulate a comprehensive list of all missing asset references in a single pass.

2.  **Runtime Mode:** During live gameplay, a missing asset reference is a catastrophic and unrecoverable error. In this context, the **handle** method's fallback behavior is to throw a new instance of MissingAssetException, which typically halts the loading process and results in a controlled crash or error screen.

This dual-mode behavior makes the asset loading pipeline robust and adaptable to different environments without cluttering the deserialization logic with conditional checks.

### Lifecycle & Ownership
-   **Creation:** Instantiated dynamically within the static **handle** method when an asset reference cannot be resolved and the system is not in a validation context. It is created on the stack at the point of failure.
-   **Scope:** Extremely short-lived. The object exists only from the moment it is thrown to the moment it is caught by a higher-level exception handler, typically at the boundary of a major engine subsystem (e.g., the world loader or UI screen manager).
-   **Destruction:** The object is eligible for garbage collection immediately after its corresponding **catch** block completes execution. It holds no external references and is not managed by any container.

## Internal State & Concurrency
-   **State:** Immutable. The exception's state, which includes the field name, asset type, and asset ID of the missing reference, is set at construction time and cannot be modified. It serves purely as a data carrier for error information.
-   **Thread Safety:** This class is inherently thread-safe. Its immutability guarantees that no data races can occur on its instances. The static **handle** methods are also thread-safe as they operate only on their arguments and do not modify any shared static state.

## API Surface
The primary interaction point is the static **handle** method, not the constructors.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| handle(extraInfo, field, assetType, assetId) | static void | O(1) | Primary entry point. Conditionally reports a missing asset to a validator or throws this exception. |
| handle(extraInfo, field, assetType, assetId, extra) | static void | O(1) | Overload that includes an extra string for more detailed error messages. |
| getField() | String | O(1) | Returns the name of the field in the parent asset that contained the broken reference. |
| getAssetType() | Class | O(1) | Returns the expected class type of the missing asset. |
| getAssetId() | Object | O(1) | Returns the unique identifier of the asset that could not be found. |

## Integration Patterns

### Standard Usage
The correct pattern is to always defer error handling to the static **handle** method during asset deserialization. This ensures that the system correctly adapts to validation or runtime environments.

```java
// Inside a hypothetical asset deserialization method
public void decode(ExtraInfo extraInfo, JsonObject data) {
    String requiredSoundId = data.getString("walkSound");
    if (!assetStore.exists(Sound.class, requiredSoundId)) {
        // Correct: Let the system decide how to handle the error.
        // This will either throw or log to a validation report.
        MissingAssetException.handle(extraInfo, "walkSound", Sound.class, requiredSoundId);
        return; // Stop processing this asset
    }
    this.walkSound = assetStore.get(Sound.class, requiredSoundId);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not call `throw new MissingAssetException(...)` directly. This bypasses the crucial validation-aware logic within the **handle** method. Bypassing the handler forces a runtime failure even during a safe validation pass, defeating the purpose of the architecture.
-   **Ignoring the Exception:** Do not wrap calls that might produce this error in an empty `try/catch` block. A MissingAssetException at runtime indicates a critical content bug that must be fixed. Swallowing it will lead to unpredictable NullPointerExceptions and corrupted game state later in execution.

## Data Pipeline
The MissingAssetException acts as a control flow switch rather than a traditional data processor. Its path determines the outcome of the asset loading process.

> **Flow:**
> Asset Deserialization → Reference Resolution Fails → **MissingAssetException.handle()** is called
> 
> **Path A (Validation Context):**
> **MissingAssetException.handle()** → AssetValidationResults.handleMissingAsset() → Error is appended to Validation Report → Deserialization continues
> 
> **Path B (Runtime Context):**
> **MissingAssetException.handle()** → `throw new MissingAssetException()` → Call Stack Unwinds → Engine Exception Handler → Game Crash / Error Screen

