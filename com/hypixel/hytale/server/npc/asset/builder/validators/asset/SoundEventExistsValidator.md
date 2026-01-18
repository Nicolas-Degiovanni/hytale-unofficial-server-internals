---
description: Architectural reference for SoundEventExistsValidator
---

# SoundEventExistsValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators.asset
**Type:** Utility

## Definition
```java
// Signature
public class SoundEventExistsValidator extends AssetValidator {
```

## Architecture & Concepts
The SoundEventExistsValidator is a specialized, stateless predicate component within the server's asset validation framework. Its sole responsibility is to confirm that a given string identifier corresponds to a fully loaded and registered SoundEvent asset.

This validator acts as a gatekeeper during the NPC asset building process. When an NPC definition file specifies a sound event by name (e.g., for a footstep or an alert), this validator is invoked to ensure the reference is valid before the NPC asset is finalized. It decouples the NPC configuration parsing logic from the underlying asset management system by providing a simple, testable contract.

Its core operational dependency is the static asset map maintained by the SoundEvent class. The validator performs a direct lookup against this central registry, making its correctness contingent on the state of the global asset system.

### Lifecycle & Ownership
-   **Creation:** The class employs a hybrid factory and singleton pattern.
    -   A default, shared instance is pre-initialized as a static final field (DEFAULT_INSTANCE) and exposed via the static **required** method. This is the preferred mechanism for standard validation.
    -   Customized instances can be created on-demand via the static **withConfig** factory method, which allows for non-default validation behaviors. These instances are transient.

-   **Scope:**
    -   The default singleton instance is application-scoped and persists for the entire server session.
    -   Instances created via **withConfig** are ephemeral and should be considered method-scoped. They are typically created, used for a validation pass, and then become eligible for garbage collection.

-   **Destruction:** The default instance is reclaimed by the JVM during application shutdown. Transient instances are managed by the garbage collector.

## Internal State & Concurrency
-   **State:** The SoundEventExistsValidator is effectively immutable and stateless. It holds no mutable instance fields. Its behavior is determined entirely by its inputs and the state of the external, static SoundEvent asset map.

-   **Thread Safety:** This class is inherently thread-safe. No internal locks are required as it does not manage any state. However, callers must ensure that any invocations of the **test** method occur *after* the SoundEvent asset map has been fully and safely published during server initialization. Concurrent modification of the underlying asset map while this validator is in use will result in undefined behavior.

## API Surface
The public contract consists of the inherited validation interface and static factory methods for instantiation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(String soundEvent) | boolean | O(1) | Executes the validation check against the global SoundEvent asset map. Returns true if the asset exists. |
| errorMessage(String, String) | String | O(1) | Generates a formatted, human-readable error message for a failed validation. |
| required() | SoundEventExistsValidator | O(1) | Static factory method. Returns the shared, default singleton instance. |
| withConfig(EnumSet) | SoundEventExistsValidator | O(1) | Static factory method. Returns a new, transient instance with custom configuration. |

## Integration Patterns

### Standard Usage
This validator is intended to be used by higher-level asset builders or configuration parsers. It should be retrieved via its static factory methods and used as a predicate.

```java
// Within an NPC asset building service
String soundIdFromConfig = "hytale.monster.creeper.fuse";
AssetValidator validator = SoundEventExistsValidator.required();

if (!validator.test(soundIdFromConfig)) {
    throw new AssetBuildException(
        validator.errorMessage(soundIdFromConfig, "onFuseSound")
    );
}
// Proceed with asset construction...
```

### Anti-Patterns (Do NOT do this)
-   **Invocation Before Asset Loading:** Do not invoke the **test** method before the server's main asset loading phase is complete. Doing so will produce false negatives, as the SoundEvent asset map it depends on will be empty. This is the most common source of errors when using this component.
-   **Redundant Instantiation:** Avoid calling **withConfig** with an empty configuration set. Use the **required** method to retrieve the shared singleton instance for default validation to prevent unnecessary object allocation.

## Data Pipeline
The validator functions as a simple step in a larger data validation pipeline, transforming a string input into a boolean result based on the state of a central registry.

> Flow:
> NPC Definition File (JSON) -> Deserializer -> String (soundEvent name) -> **SoundEventExistsValidator.test()** -> Boolean (exists or not) -> Asset Builder Logic

