---
description: Architectural reference for RootInteractionValidator
---

# RootInteractionValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators.asset
**Type:** Transient

## Definition
```java
// Signature
public class RootInteractionValidator extends AssetValidator {
```

## Architecture & Concepts
The RootInteractionValidator is a concrete implementation of the **Strategy** pattern, designed for a singular, critical purpose: to verify that a string identifier corresponds to a valid, registered RootInteraction asset.

This component operates within the server's asset processing pipeline. During server initialization or hot-reloading, asset definition files (e.g., JSON for an NPC) are parsed. These definitions often contain string references to other game assets, such as interactions. The RootInteractionValidator is invoked by the asset builder to ensure these references are not broken. It acts as a gatekeeper, preventing the instantiation of game objects with invalid configuration, thereby ensuring data integrity and preventing runtime errors.

Its design decouples the specific logic of validating an interaction from the generic asset parsing and building machinery. This allows for a flexible and maintainable validation system where new validators can be added without modifying the core asset loaders.

### Lifecycle & Ownership
- **Creation:** Instances are created exclusively through the static factory methods *required()* or *withConfig(EnumSet)*. This is typically done by a higher-level asset builder or a configuration processor that is constructing a set of validation rules for a given asset type.
- **Scope:** Ephemeral and task-scoped. An instance of RootInteractionValidator exists only for the duration of the validation process for a specific asset or set of assets. It is not a long-lived service.
- **Destruction:** The object is lightweight and holds no external resources. It becomes eligible for garbage collection as soon as the reference from the asset builder is dropped, which is typically after the validation check is complete.

## Internal State & Concurrency
- **State:** Immutable. The validator's behavior, determined by the optional Config enum set, is fixed at the time of construction. The class itself contains no mutable fields.
- **Thread Safety:** Conditionally thread-safe. The *test* method performs a read-only operation against the static *RootInteraction.getAssetMap()*. The safety of this validator in a multi-threaded context is therefore entirely dependent on the thread safety of that central asset map.

    **WARNING:** This validator assumes that the RootInteraction asset map is populated during a single-threaded server bootstrap phase and is treated as an immutable, read-only collection thereafter. Concurrent modification of the asset map while this validator is in use will lead to non-deterministic behavior and potential race conditions.

## API Surface
The public contract is minimal, focusing entirely on the validation operation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(String value) | boolean | O(1) | The core validation method. Returns true if the provided string value is a key in the global RootInteraction asset map. |
| required() | RootInteractionValidator | O(1) | Static factory for creating a default validator instance. |
| withConfig(EnumSet) | RootInteractionValidator | O(1) | Static factory for creating a validator with specific configuration flags. |
| errorMessage(String, String) | String | O(1) | Generates a formatted, human-readable error message for a failed validation. |

## Integration Patterns

### Standard Usage
This validator is not intended for direct use in general game logic. It should be supplied to an asset building or configuration system that understands how to process AssetValidator implementations.

```java
// Within an asset definition or builder
AssetBuilder builder = new AssetBuilder();
builder.addAttribute("onInteract", "hytale:player_greet");

// The builder uses the validator to check the attribute
AssetValidator validator = RootInteractionValidator.required();
boolean isValid = validator.test(builder.getAttribute("onInteract"));

if (!isValid) {
    throw new AssetValidationException(
        validator.errorMessage(builder.getAttribute("onInteract"), "onInteract")
    );
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructors are private. Attempting to instantiate with *new RootInteractionValidator()* is a compile-time error and violates the intended factory-based creation pattern.
- **Premature Validation:** Do not invoke the *test* method before the server's asset loading sequence has fully completed and populated the *RootInteraction.getAssetMap()*. Doing so will cause validation to fail for all interactions, even valid ones, leading to catastrophic asset loading failures.

## Data Pipeline
The validator is a single step in a larger data validation pipeline. It consumes a string and produces a boolean, influencing the control flow of the asset loader.

> Flow:
> Asset Definition (e.g., NPC JSON file) -> Server Asset Deserializer -> String value for an interaction -> **RootInteractionValidator.test()** -> Validation Result (boolean) -> Asset Builder Control Flow

