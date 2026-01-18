---
description: Architectural reference for GameplayConfig
---

# GameplayConfig

**Package:** com.hypixel.hytale.server.core.asset.type.gameplay
**Type:** Data Asset

## Definition
```java
// Signature
public class GameplayConfig implements JsonAssetWithMap<String, DefaultAssetMap<String, GameplayConfig>> {
```

## Architecture & Concepts

The **GameplayConfig** class is a foundational data asset that defines the core rules and parameters for a server's gameplay session. It is not a service or manager, but rather a composite data object that aggregates dozens of domain-specific configurations into a single, addressable unit.

This class acts as the primary container for server-side game mechanics, including combat, world generation, player spawning, crafting, and more. Each of these domains is represented by a dedicated configuration object (e.g., **CombatConfig**, **WorldConfig**) which is a member of **GameplayConfig**.

The central architectural pattern is the use of the static **AssetBuilderCodec**. This codec declaratively defines how a JSON asset file is deserialized into a **GameplayConfig** object. Key features of this design include:

*   **Composition:** It builds a complex object graph by composing smaller, focused configuration objects.
*   **Inheritance:** The `appendInherited` method is critical. It allows gameplay configurations to form a hierarchy. For example, a "Hardcore" config can inherit from the "Default" config and only specify the values it wishes to override, reducing data duplication.
*   **Extensibility:** The special handling for **PluginConfig** demonstrates a pattern for allowing third-party plugins to inject their own configuration data directly into the main gameplay config asset, which is then accessible to plugin systems.

An instance of **GameplayConfig** represents a complete, coherent set of rules for a game world. The server loads one or more of these configurations at startup, which can then be applied to different worlds or game modes.

## Lifecycle & Ownership

-   **Creation:** **GameplayConfig** instances are **never** instantiated directly via the `new` keyword in game logic. They are exclusively created by the Hytale **AssetStore** framework during the server's initial asset loading phase. The static **CODEC** field dictates the entire construction and data-binding process from JSON source files.
-   **Scope:** An instance persists for the entire server session once loaded. It is a long-lived object, effectively a global constant for the duration of the server's runtime.
-   **Destruction:** Instances are garbage collected along with the entire **AssetStore** when the server process terminates. There is no explicit cleanup or destruction mechanism.

## Internal State & Concurrency

-   **State:** The state of a **GameplayConfig** object is mutable only during its creation by the **AssetBuilderCodec**. After the `afterDecode` hook (which calls **processConfig**) completes, the object must be treated as **effectively immutable**. The `processConfig` method performs post-load processing, such as resolving the `creativePlaySoundSet` string into a more performant integer index, `creativePlaySoundSetIndex`.

-   **Thread Safety:** This class is **not thread-safe** for mutation. However, as its state is fixed after the initial loading phase, it is safe for concurrent reads from any number of threads. Game systems running on different threads can safely access the same **GameplayConfig** instance to retrieve rule parameters.

    **Warning:** The lazy initialization pattern in **getAssetStore** relies on the underlying **AssetRegistry** to be thread-safe. Direct manipulation of the static **ASSET_STORE** field from multiple threads would be catastrophic.

## API Surface

The primary interaction points are the static methods for retrieving assets and the instance getters for accessing sub-configurations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | static AssetStore | O(1) | Returns the central store for all **GameplayConfig** assets. The primary entry point. |
| getAssetMap() | static DefaultAssetMap | O(1) | Convenience method to get the asset map directly from the store. |
| getId() | String | O(1) | Returns the unique identifier for this specific configuration (e.g., "Default"). |
| get*Config() | *Config | O(1) | A family of getters (e.g., **getCombatConfig**, **getWorldConfig**) for accessing sub-configuration objects. |

## Integration Patterns

### Standard Usage

Game logic should retrieve the required configuration from the global **AssetStore** during its own initialization phase and cache the reference if needed.

```java
// Retrieve the map of all available gameplay configurations
DefaultAssetMap<String, GameplayConfig> configs = GameplayConfig.getAssetMap();

// Get the default configuration by its well-known ID
GameplayConfig defaultConfig = configs.get(GameplayConfig.DEFAULT_ID);

// Access a specific sub-configuration to read a game rule
if (defaultConfig != null) {
    CombatConfig combatRules = defaultConfig.getCombatConfig();
    float knockbackStrength = combatRules.getKnockbackStrength();
    // ... use rule in game logic
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new GameplayConfig()`. This creates a default, empty object that is not managed by the asset system and does not contain any data loaded from JSON files. It will lead to default, and likely incorrect, game behavior.
-   **State Mutation:** Do not modify a **GameplayConfig** object or its sub-objects after it has been loaded. This violates the "effectively immutable" contract and will cause unpredictable behavior and race conditions across the server.
-   **Repeated Lookups:** Avoid calling **getAssetStore** or **getAssetMap** inside tight loops. While fast, it is best practice to retrieve the reference once and reuse it.

## Data Pipeline

The **GameplayConfig** is the result of a data pipeline that transforms a declarative file on disk into a usable in-memory object graph for the server engine.

> Flow:
> JSON Asset File (e.g., `default_gameplay.json`) -> Server Asset Loader -> **AssetBuilderCodec** (Deserialization & Inheritance) -> **GameplayConfig Instance** -> Populated into **AssetStore** -> Accessed by Game Systems (Combat, World, etc.)

