---
description: Architectural reference for FlockAssetExistsValidator
---

# FlockAssetExistsValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators.asset
**Type:** Utility

## Definition
```java
// Signature
public class FlockAssetExistsValidator extends AssetValidator {
```

## Architecture & Concepts
The FlockAssetExistsValidator is a specialized rule component within the server's asset validation framework. Its primary function is to enforce data integrity during the loading and building of NPC assets. It acts as a gatekeeper, ensuring that any string reference to a flocking behavior configuration, known as a FlockAsset, corresponds to an actual, loaded asset in the central registry.

This class decouples the validation logic from the asset parsing and object construction logic. By centralizing this check, the system avoids runtime NullPointerExceptions or invalid state errors that would occur if an NPC attempted to use a non-existent flocking behavior. It is a critical component for pre-flight checks that run when the server boots or when game assets are reloaded, providing early and clear feedback on configuration errors.

Its operation is entirely dependent on the state of the static FlockAsset registry, querying it to confirm the existence of an asset by its unique string identifier.

### Lifecycle & Ownership
-   **Creation:** The validator is instantiated exclusively through its static factory methods.
    -   The `required()` method returns a shared, static, default-configured singleton instance. This instance is created once when the FlockAssetExistsValidator class is loaded by the JVM.
    -   The `withConfig(EnumSet<Config>)` method constructs a new, transient instance with specific validation behaviors (e.g., null handling). These instances are typically created by higher-level asset builders or configuration processors.
-   **Scope:** The default singleton instance (`DEFAULT_INSTANCE`) is application-scoped and persists for the entire server session. Custom instances created via `withConfig` are ephemeral and should be considered scoped to the specific validation operation that created them.
-   **Destruction:** There is no explicit destruction mechanism. The singleton instance is cleaned up when its classloader is unloaded. Transient instances are managed by the Java garbage collector once they are no longer referenced.

## Internal State & Concurrency
-   **State:** This class is stateless. It holds no mutable instance fields and does not cache results. Its `test` method's outcome is determined solely by its input and the external state of the global `FlockAsset.getAssetMap()`.
-   **Thread Safety:** The class is immutable and inherently thread-safe. Multiple threads can safely call methods on a shared instance.

    **Warning:** Thread safety is contingent on the assumption that the underlying `FlockAsset.getAssetMap()` is safe for concurrent read operations. This is a standard design expectation for engine-level asset registries after the initial loading phase is complete.

## API Surface
The public contract is minimal, focusing on instance creation and the core validation predicate.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(String flockAsset) | boolean | O(1) | The core validation logic. Returns true if an asset with the given name exists in the global registry. |
| errorMessage(String, String) | String | O(1) | Generates a formatted, human-readable error message for failed validations. |
| required() | FlockAssetExistsValidator | O(1) | Static factory for retrieving the shared, default singleton instance. |
| withConfig(EnumSet) | FlockAssetExistsValidator | O(1) | Static factory for creating a new, configured instance. |

## Integration Patterns

### Standard Usage
This validator is not intended for direct use in general game logic. It is designed to be composed into a larger validation chain, typically during asset deserialization or within a builder pattern.

```java
// Within an asset loading or builder class
AssetValidator validator = FlockAssetExistsValidator.required();
String potentialFlockName = "avian_flock_default"; // Read from a config file

if (!validator.test(potentialFlockName)) {
    String error = validator.errorMessage(potentialFlockName, "npc.behavior.flocking");
    throw new AssetValidationException(error);
}

// Proceed with asset creation...
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** The constructors are private. Do not attempt to create instances using reflection. Always use the static `required()` or `withConfig()` factory methods.
-   **Premature Validation:** Do not invoke this validator before the server's asset loading sequence has fully populated the `FlockAsset` registry. Doing so will produce false negatives, causing valid configurations to fail validation. This validator is only reliable after the main asset bootstrap phase is complete.

## Data Pipeline
The validator functions as a single, synchronous step in a larger data validation pipeline.

> Flow:
> NPC Definition File (JSON) -> Deserializer -> String `flockAsset` -> **FlockAssetExistsValidator.test()** -> Validation Result (boolean) -> Asset Builder / Error Reporter

