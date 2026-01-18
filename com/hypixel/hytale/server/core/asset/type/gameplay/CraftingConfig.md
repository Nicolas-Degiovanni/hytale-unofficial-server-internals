---
description: Architectural reference for CraftingConfig
---

# CraftingConfig

**Package:** com.hypixel.hytale.server.core.asset.type.gameplay
**Type:** Data Model / Configuration Object

## Definition
```java
// Signature
public class CraftingConfig {
```

## Architecture & Concepts

The CraftingConfig class is a server-side data model that encapsulates configuration parameters for the crafting system. It is not a service or manager; rather, it is a passive, in-memory representation of settings loaded from an external asset file, such as a JSON or HOCON document.

The core architectural component of this class is the public static final field named **CODEC**. This field is an instance of BuilderCodec and serves as a complete, self-contained schema for serializing and deserializing CraftingConfig objects. The engine's asset loading system uses this CODEC to instantiate and populate CraftingConfig objects from configuration files on disk.

This design pattern decouples the configuration data from the game logic that consumes it. The CODEC defines:
*   The expected keys in the configuration file (e.g., BenchMaterialChestHorizontalSearchRadius).
*   The data type for each key (e.g., INTEGER).
*   Validation rules applied during deserialization (e.g., `Validators.range(0, 7)`).
*   Default values for all fields.
*   Logic for handling hierarchical configurations, where values can be inherited from a parent configuration.

This class is a terminal node in the configuration asset pipeline, providing a type-safe, validated, and easy-to-access container for crafting-related constants.

## Lifecycle & Ownership

*   **Creation:** CraftingConfig instances are **exclusively** created by the Hytale Codec framework during the server's asset loading phase. A central configuration manager or asset loader will locate the relevant configuration file and use the static CODEC field to deserialize its contents into a new CraftingConfig object.
*   **Scope:** An instance of CraftingConfig typically persists for the lifetime of the context it configures. For global server settings, this means it lives for the entire server session. It is effectively a read-only snapshot of the configuration at the time the server started.
*   **Destruction:** The object is marked for garbage collection when its owning context (e.g., the server world or game session) is shut down and all references to it are released. There is no manual destruction or cleanup method.

## Internal State & Concurrency

*   **State:** The class holds mutable integer fields that represent specific crafting mechanics. However, it is designed to be **effectively immutable** after its initial creation by the Codec system. The state is a direct reflection of the parsed configuration file, with defaults applied for any missing keys.
*   **Thread Safety:** This class is **not thread-safe for mutation**. However, as its intended use is as a read-only data container post-initialization, it is **safe for concurrent reads** from multiple game logic threads. Any system can safely call its getter methods without synchronization, as the underlying state is not expected to change during runtime.

## API Surface

The primary public contract is the static CODEC field, which is consumed by the engine. The instance methods are simple data accessors.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getBenchMaterialHorizontalChestSearchRadius() | int | O(1) | Returns the configured horizontal search radius for crafting bench material chests. |
| getBenchMaterialVerticalChestSearchRadius() | int | O(1) | Returns the configured vertical search radius for crafting bench material chests. |
| getBenchMaterialChestLimit() | int | O(1) | Returns the maximum number of chests a crafting bench will pull materials from. |

## Integration Patterns

### Standard Usage

Game systems should retrieve the loaded CraftingConfig instance from a central service registry or context provider. They should never create their own instance.

```java
// Example: A crafting bench block entity retrieving configuration
// The 'context' object is a hypothetical service locator.
CraftingConfig config = context.getGameplayConfig().getCraftingConfig();

int horizontalRadius = config.getBenchMaterialHorizontalChestSearchRadius();
int verticalRadius = config.getBenchMaterialVerticalChestSearchRadius();

// Use the retrieved values to perform a search for nearby material chests.
findChestsInRadius(horizontalRadius, verticalRadius);
```

### Anti-Patterns (Do NOT do this)

*   **Direct Instantiation:** Under no circumstances should you use `new CraftingConfig()`. Doing so bypasses the entire configuration loading pipeline and results in an object with only hardcoded default values, ignoring server-specific customizations. This will lead to behavior that is inconsistent with the server's configuration files.
*   **Runtime Mutation:** Do not attempt to modify the fields of a loaded CraftingConfig instance via reflection. The configuration system is designed to be a single source of truth loaded at startup. Modifying it at runtime will cause unpredictable behavior and desynchronization between different game systems that have already read the values.

## Data Pipeline

The CraftingConfig object is the final product of a data transformation pipeline that begins with a raw asset file on the server's file system.

> Flow:
> `gameplay.json` (File System) -> Server Asset Loader -> Hytale Codec Deserializer (using `CraftingConfig.CODEC`) -> **CraftingConfig Instance** -> Gameplay Service Registry -> Crafting Bench Logic

