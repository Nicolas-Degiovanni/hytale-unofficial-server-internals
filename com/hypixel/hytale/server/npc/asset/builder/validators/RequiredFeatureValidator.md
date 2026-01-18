---
description: Architectural reference for RequiredFeatureValidator
---

# RequiredFeatureValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Base Class / Template

## Definition
```java
// Signature
public abstract class RequiredFeatureValidator extends Validator {
```

## Architecture & Concepts
The RequiredFeatureValidator is an abstract base class that establishes a fundamental contract within the server-side NPC asset validation system. It serves as a template for creating specific validation rules that enforce the presence of mandatory features on an NPC asset definition.

This class is a key component of a Strategy Pattern implementation for asset validation. The core asset building service maintains a collection of concrete RequiredFeatureValidator implementations (e.g., a validator for a required model component, another for a base animation set). During the asset build or update process, the builder iterates through these validators, executing each one against the asset data to ensure its integrity.

Its primary architectural role is to decouple the generic validation loop from the specific, hard-coded rules. This allows developers to add new mandatory feature checks without modifying the core asset builder logic, promoting extensibility and maintainability. The FeatureEvaluatorHelper object, passed as an argument, acts as a context and data accessor, providing a safe and consistent interface for validators to inspect the asset under review.

### Lifecycle & Ownership
- **Creation:** Concrete implementations are instantiated and configured by a higher-level manager, such as the NpcAssetBuilder or a dedicated ValidatorRegistry, during the server's bootstrap phase or when the asset pipeline is initialized. They are not intended for direct instantiation by general game logic.
- **Scope:** Instances are typically long-lived, often existing as singletons within the context of the validation system. They are created once and reused for every NPC asset validation request throughout the server's uptime.
- **Destruction:** These objects are destroyed when the server shuts down or the asset system is reloaded, at which point they are garbage collected. They manage no external resources that would require explicit cleanup.

## Internal State & Concurrency
- **State:** This base class is entirely stateless. Concrete implementations are strongly expected to be stateless as well. All data required for a validation check is provided via the FeatureEvaluatorHelper argument in the `validate` method. This design ensures that validators are fully re-entrant and produce deterministic results.
- **Thread Safety:** The class is inherently thread-safe due to its stateless design. A single instance of a concrete validator can be safely invoked by multiple threads concurrently, provided each thread is processing a distinct asset and FeatureEvaluatorHelper context.

## API Surface
The public contract is defined by two abstract methods that all concrete subclasses must implement.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| validate(FeatureEvaluatorHelper) | boolean | O(N) | Executes the validation logic. Returns true if the required feature is present, false otherwise. Complexity is dependent on the asset's feature count. |
| getErrorMessage(String) | String | O(1) | Generates a human-readable error message to be logged or displayed if validation fails. The String argument is typically the name of the asset. |

## Integration Patterns

### Standard Usage
A validation runner or asset builder will hold a list of validators and execute them sequentially. The process is halted on the first failure to provide a clear, singular error.

```java
// Simplified example within an asset builder
FeatureEvaluatorHelper helper = new FeatureEvaluatorHelper(npcAsset);
List<RequiredFeatureValidator> validators = getRequiredValidators();

for (RequiredFeatureValidator validator : validators) {
    if (!validator.validate(helper)) {
        String assetName = npcAsset.getName();
        String error = validator.getErrorMessage(assetName);
        throw new AssetValidationException(error);
    }
}
// Asset is considered valid if all checks pass
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Subclasses must not contain mutable instance variables. A validator should not "remember" results from previous calls, as this breaks re-entrancy and can cause severe, difficult-to-diagnose bugs in a multithreaded environment.
- **Ignoring Failure:** The boolean return from `validate` is critical. Ignoring a `false` result defeats the purpose of the validator and can lead to corrupted or unstable NPC assets being loaded into the game world.
- **Complex Error Messages:** The `getErrorMessage` method should not perform complex logic or I/O. It is intended for simple, fast string formatting.

## Data Pipeline
This class acts as a gatekeeper within the NPC asset ingestion pipeline. It does not transform data but rather asserts conditions on it.

> Flow:
> NPC Asset Definition (Data File) -> Deserializer -> NpcAssetBuilder -> **RequiredFeatureValidator** -> Validation Result (Pass/Fail) -> Finalized In-Memory Asset OR Error Log

