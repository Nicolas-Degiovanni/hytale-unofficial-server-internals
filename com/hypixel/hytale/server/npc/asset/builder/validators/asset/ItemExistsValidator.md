---
description: Architectural reference for ItemExistsValidator
---

# ItemExistsValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators.asset
**Type:** Utility

## Definition
```java
// Signature
public class ItemExistsValidator extends AssetValidator {
```

## Architecture & Concepts

The ItemExistsValidator is a specialized rule engine component within the server's NPC asset building and validation pipeline. Its primary function is to enforce data integrity by ensuring that item identifiers referenced in configuration files correspond to actual, loaded game assets.

This class embodies the Strategy pattern, providing a concrete implementation of the abstract AssetValidator. It is used during the server's startup and asset loading phase to preemptively catch configuration errors. By validating that an NPC's specified items, droplists, or equipment exist in the central asset registries, it prevents runtime errors and ensures that the game world loads into a consistent and valid state.

The validator is highly configurable through its static factory methods, allowing it to enforce different rules, such as requiring an item to be a block or allowing a reference to be an ItemDropList instead of a single Item. This flexibility makes it a crucial tool for defining the constraints of various NPC attributes.

### Lifecycle & Ownership
- **Creation:** Instances are created exclusively through static factory methods like `required()`, `requireBlock()`, and `orDroplist()`. The calling system, typically the NPC asset builder, is responsible for selecting and instantiating the correct validator configuration for a given attribute. A shared, default instance is available via `required()` for common use cases.
- **Scope:** Instances are typically transient and short-lived. They are created for the duration of a specific validation task and then become eligible for garbage collection. The `DEFAULT_INSTANCE` is a static final field, meaning it persists for the lifetime of the server process.
- **Destruction:** Transient instances are destroyed by the garbage collector once the asset validation process that created them is complete.

## Internal State & Concurrency
- **State:** The internal state consists of the `requireBlock` and `allowDroplist` booleans. This state is set at construction time via the private constructors and is immutable for the lifetime of the object.
- **Thread Safety:** This class is thread-safe. Its internal state is immutable, and its validation logic relies on calls to globally accessible, static asset registries like `ItemDropList.getAssetMap()` and `InventoryHelper`. It is safe to assume these underlying core systems are designed for concurrent access, making the ItemExistsValidator suitable for use in parallelized asset loading and validation tasks.

## API Surface

The public contract is exposed through static factory methods for instantiation and the inherited `test` method for execution.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| required() | ItemExistsValidator | O(1) | Returns a shared, default validator instance. |
| requireBlock() | ItemExistsValidator | O(1) | Creates a new validator that requires the item to be a block. |
| orDroplist() | ItemExistsValidator | O(1) | Creates a new validator that also accepts droplist identifiers. |
| withConfig(config) | ItemExistsValidator | O(1) | Creates a new validator with a specified base configuration. |
| test(String item) | boolean | O(1) | Executes the validation logic. Returns true if the item string is valid according to the instance's configuration. Complexity assumes underlying asset maps are hash-based. |

## Integration Patterns

### Standard Usage

This validator is not intended to be used in isolation. It is designed to be integrated into a larger asset building framework where it is applied to specific fields of an asset definition.

```java
// Hypothetical usage within an asset builder
// A developer would configure the builder, not call test() directly.

// AssetBuilder builder = new AssetBuilder<NpcDefinition>();
// builder.forField("heldItem")
//        .addValidator(ItemExistsValidator.requireBlock());
// builder.forField("lootTable")
//        .addValidator(ItemExistsValidator.orDroplist());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructors are private. Do not attempt to create instances using reflection. Always use the provided static factory methods to ensure correct initialization.
- **Premature Validation:** Do not use this validator before the server's core `Item` and `ItemDropList` asset registries have been fully populated. Doing so will always result in a validation failure. This component is only effective late in the server startup lifecycle.
- **Incorrect Configuration:** Using the wrong validator for a given context can lead to subtle bugs. For example, using `ItemExistsValidator.required()` for an attribute that must logically be a placeable block will allow non-block items, potentially causing runtime errors later.

## Data Pipeline

The validator acts as a gate in the data flow from raw configuration to a loaded in-memory asset. It transforms a string identifier into a binary pass or fail signal.

> Flow:
> NPC JSON/HOCON File -> Configuration Parser -> String value ("hytale:stone") -> **ItemExistsValidator.test()** -> Global Asset Registries -> **boolean (true/false)** -> Asset Build Success or Failure

