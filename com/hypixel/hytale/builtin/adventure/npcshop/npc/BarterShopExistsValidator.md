---
description: Architectural reference for BarterShopExistsValidator
---

# BarterShopExistsValidator

**Package:** com.hypixel.hytale.builtin.adventure.npcshop.npc
**Type:** Utility / Flyweight

## Definition
```java
// Signature
public class BarterShopExistsValidator extends AssetValidator {
```

## Architecture & Concepts
The BarterShopExistsValidator is a concrete implementation of the **Strategy Pattern**, designed to enforce data integrity during the server's asset loading and processing phase. Its sole responsibility is to verify that a string identifier, intended to reference a barter shop, corresponds to a `BarterShopAsset` that has been successfully loaded into the engine's central asset registry.

This class acts as a critical guard within the NPC asset pipeline. When the server parses NPC definition files, this validator is invoked to check attributes that link an NPC to a specific shop. By confirming the existence of the target `BarterShopAsset` at load-time, it prevents entire classes of runtime errors, such as NullPointerExceptions or illegal state, that would otherwise occur if an NPC attempted to interact with a misconfigured or non-existent shop.

Architecturally, its reliance on the static `BarterShopAsset.getAssetMap()` method signifies a tight coupling with the global asset management system. This design assumes a centralized, statically accessible registry for all loaded assets of a given type, which is a common pattern in game engines for performance and simplicity.

### Lifecycle & Ownership
-   **Creation:** Instances are created exclusively via the static factory methods `required()` or `withConfig()`. The `required()` method implements a **Flyweight pattern**, returning a shared, pre-instantiated singleton instance for default validation cases. The `withConfig()` method allocates a new object for specialized validation rules. Direct instantiation is prohibited by a private constructor.
-   **Scope:** The default singleton instance returned by `required()` is static and persists for the entire lifetime of the server process. Instances created via `withConfig()` are typically ephemeral, existing only for the duration of a specific asset-building configuration and becoming eligible for garbage collection thereafter.
-   **Destruction:** The default static instance is never destroyed. All other instances are managed by the Java garbage collector.

## Internal State & Concurrency
-   **State:** The BarterShopExistsValidator is effectively **immutable**. Its only state is a configuration object inherited from the parent `AssetValidator`, which is set at construction time and never modified. The core validation logic is stateless.
-   **Thread Safety:** This class is **conditionally thread-safe**. The validator object itself has no mutable state and can be safely shared across threads. However, its correctness is entirely dependent on the thread safety of the global `BarterShopAsset.getAssetMap()`.

    **Warning:** The underlying asset system is assumed to be populated in a single-threaded context during server initialization and subsequently treated as read-only. Invoking this validator from multiple threads while the `BarterShopAsset` map is being concurrently modified will result in race conditions and non-deterministic validation failures.

## API Surface
The public API is designed for configuration within an asset-building framework, not for direct invocation in game logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| required() | static BarterShopExistsValidator | O(1) | Returns the shared, default instance of the validator. |
| withConfig(config) | static BarterShopExistsValidator | O(1) | Creates a new validator instance with specific configuration flags. |
| test(marker) | boolean | O(1) | Executes the validation logic. Returns true if an asset exists for the given marker. |
| errorMessage(marker, attributeName) | String | O(1) | Generates a detailed error message for logging if validation fails. |

## Integration Patterns

### Standard Usage
This validator is not intended for direct use by gameplay programmers. It is a component to be registered within a higher-level asset definition or builder framework. The framework invokes it automatically during the asset loading process.

```java
// This is a hypothetical example of how the asset framework would use the validator.
// Developers do not write this code directly.
npcAssetBuilder
    .withAttribute("shopMarker", String.class)
    .addValidator(BarterShopExistsValidator.required());
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not attempt to create an instance with `new BarterShopExistsValidator()`. The constructor is private. Use the static `required()` or `withConfig()` factory methods.
-   **Runtime Logic:** Do not use this validator for runtime checks within game systems. It is designed for load-time validation only. For runtime checks, query the `AssetManager` or a relevant service directly to see if an asset is available.
-   **Misplaced Responsibility:** Do not call `validator.test("my_shop")` manually. The asset processing system is responsible for invoking the `test` method at the correct point in its pipeline.

## Data Pipeline
The BarterShopExistsValidator functions as a gate in the data flow from raw asset files to in-memory game objects. It does not transform data; it either permits it to pass or halts the pipeline with an error.

> Flow:
> NPC Asset File (e.g., JSON) -> Asset Deserializer -> **BarterShopExistsValidator** -> (Success) -> In-Memory NPC Asset | (Failure) -> Server Startup Error Log<ctrl63>

