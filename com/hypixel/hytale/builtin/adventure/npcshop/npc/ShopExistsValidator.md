---
description: Architectural reference for ShopExistsValidator
---

# ShopExistsValidator

**Package:** com.hypixel.hytale.builtin.adventure.npcshop.npc
**Type:** Utility / Factory

## Definition
```java
// Signature
public class ShopExistsValidator extends AssetValidator {
```

## Architecture & Concepts
The ShopExistsValidator is a specialized component within the server's asset validation framework. Its sole responsibility is to enforce data integrity for NPC configurations by ensuring that any reference to a shop asset points to a valid, loaded shop.

This class acts as a runtime guard during the server's asset loading phase. When the server parses NPC definition files (e.g., JSON or HOCON), it uses a system of validators to check the correctness of the data. The ShopExistsValidator is specifically invoked for attributes that are expected to contain the unique name of a ShopAsset. It queries the central ShopAsset registry to confirm the existence of the referenced shop, preventing the server from loading NPCs with broken or misconfigured shop links. This preemptive validation is critical for preventing null pointer exceptions and other state corruption issues during gameplay.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively through the static factory methods `required()` and `withConfig()`. The `required()` method returns a shared, stateless singleton instance, which is the most common use case. The `withConfig()` method is used to create a new, transient instance when non-default validation behavior is needed. This class is typically instantiated by the asset loading system itself, not by general game logic code.
- **Scope:** The default singleton instance, returned by `required()`, is static and persists for the entire lifetime of the server process. Instances created via `withConfig()` are short-lived and scoped to the specific asset validation operation they are part of.
- **Destruction:** The singleton instance is reclaimed by the JVM when the server shuts down. Transient instances are eligible for garbage collection as soon as the asset validation process completes.

## Internal State & Concurrency
- **State:** This class is effectively stateless. The default instance holds no state. Instances created with a configuration hold only an immutable `EnumSet` of configuration flags. The core validation logic in the `test` method is a read-only operation against the global `ShopAsset` map.
- **Thread Safety:** The class is thread-safe. Its methods do not modify any internal state. The safety of the `test` method is contingent upon the thread safety of the global `ShopAsset.getAssetMap()`. Asset maps are typically populated during a single-threaded startup phase and are safe for concurrent reads thereafter, making this validator safe for use in a multi-threaded asset loading environment.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(String marker) | boolean | O(1) | Checks if a ShopAsset exists for the given marker. Returns true if found, false otherwise. |
| errorMessage(String, String) | String | O(1) | Generates a standardized, human-readable error message for a failed validation. |
| required() | ShopExistsValidator | O(1) | Static factory method to retrieve the shared, default instance of the validator. |
| withConfig(EnumSet) | ShopExistsValidator | O(1) | Static factory method to create a new validator with specific configuration flags. |

## Integration Patterns

### Standard Usage
This validator is not intended for direct invocation in typical game logic. It is designed to be integrated into the asset definition and validation pipeline, often declaratively.

```java
// Pseudo-code for an NPC asset builder
// The framework would use the validator internally.
NPCTemplateBuilder builder = new NPCTemplateBuilder();

builder.withName("traveling_merchant");
builder.withModel("hytale:merchant");

// The 'shop' field is decorated with the validator.
// The framework automatically calls ShopExistsValidator.required().test("hytale:general_store")
// during the build/load process.
builder.withShop("hytale:general_store", ShopExistsValidator.required());

NPCTemplate merchant = builder.build(); // Throws ValidationException if shop does not exist
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to call `new ShopExistsValidator()`. The constructor is private to enforce the use of the static factory methods, which manage the singleton instance correctly.
- **Stateful Logic:** Do not extend this class with the intent of adding state. The validation framework relies on validators being stateless and reusable.
- **Incorrect Context:** Do not use this validator to check for the existence of any asset type other than a ShopAsset. The implementation is hardcoded to the `ShopAsset.getAssetMap()` and will always fail for other asset types.

## Data Pipeline
The validator sits in the middle of the asset loading pipeline, acting as a gate for data correctness.

> Flow:
> NPC Definition File (e.g., npc.json) -> Server Asset Deserializer -> Asset Validation Framework -> **ShopExistsValidator** -> Validated NPC Object in Memory OR ValidationException

