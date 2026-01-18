---
description: Architectural reference for GameModeType
---

# GameModeType

**Package:** com.hypixel.hytale.server.core.asset.type.gamemode
**Type:** Data Asset

## Definition
```java
// Signature
public class GameModeType implements JsonAssetWithMap<String, DefaultAssetMap<String, GameModeType>> {
```

## Architecture & Concepts

The GameModeType class is a data-driven model that represents the server-side configuration for a specific game mode, such as Adventure or Creative. It is not an active service but rather a static data container loaded from JSON asset files at server startup.

Its primary architectural role is to decouple game mode configuration from the core server logic. Instead of hardcoding behaviors, the server can define game modes, their associated permissions, and entry-point interactions entirely within data files.

The static **CODEC** field is the central mechanism for this class. It is an `AssetBuilderCodec` that declaratively defines how a JSON object is deserialized into a GameModeType instance. This pattern allows developers to add new configuration properties to the JSON files and map them to new fields in this class with minimal code changes, promoting a data-oriented design.

This class acts as a bridge, enriching the simple `com.hypixel.hytale.protocol.GameMode` enum with rich, server-specific metadata required by systems like permissions and interactions.

## Lifecycle & Ownership

-   **Creation:** Instances are created exclusively by the Hytale Asset System during the server's initial asset loading phase. The static `CODEC` is used to parse corresponding JSON files (e.g., `adventure.json`) and construct GameModeType objects. **WARNING:** Direct instantiation by user code is unsupported and will result in a non-functional object.
-   **Scope:** The lifecycle of a GameModeType instance is tied to the server's runtime. All discovered and loaded game mode assets are stored in a static, singleton `AssetStore` and persist for the entire server session.
-   **Destruction:** All instances are destroyed and garbage collected when the server shuts down and its classloader is unloaded. There is no mechanism for hot-reloading or explicit destruction of individual GameModeType assets.

## Internal State & Concurrency

-   **State:** A GameModeType instance is **effectively immutable**. Its fields are populated only once during deserialization at server startup. The state represents static configuration from disk and is not designed to be mutated at runtime.
-   **Thread Safety:** The class is **thread-safe for read operations**. Since its internal state does not change after initialization, multiple threads can safely access its properties without synchronization. The static `getAssetStore` method contains a lazy initializer; however, it is expected to be first called during the single-threaded server bootstrap sequence. Subsequent concurrent calls are safe.

## API Surface

The public API is focused on static lookups and read-only access to configuration data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | static AssetStore | O(1) | Returns the singleton store for all GameModeType assets. Throws if called before the AssetRegistry is initialized. |
| fromGameMode(GameMode) | static GameModeType | O(1) | Resolves a protocol enum into its corresponding configured GameModeType. This is the primary entry point for using the class. |
| getInteractionsOnEnter() | String | O(1) | Returns the asset key for the RootInteraction to trigger when a player enters this mode. |
| getPermissionGroups() | String[] | O(1) | Returns the list of permission groups associated with this game mode. Guaranteed to be non-null. |
| getId() | String | O(1) | Returns the unique string identifier for this game mode (e.g., "ADVENTURE"). |

## Integration Patterns

### Standard Usage

The standard pattern is to use the static factory method `fromGameMode` to retrieve the configuration associated with a player's current game mode.

```java
// Retrieve the configuration for a player's current game mode
GameMode currentMode = player.getGameMode(); // e.g., GameMode.ADVENTURE
GameModeType modeConfig = GameModeType.fromGameMode(currentMode);

// Use the configuration to drive other systems
permissionManager.applyGroups(player, modeConfig.getPermissionGroups());

String interactionKey = modeConfig.getInteractionsOnEnter();
if (interactionKey != null) {
    interactionSystem.trigger(player, interactionKey);
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new GameModeType()`. The constructor is protected and intended only for use by the asset deserialization system. Manually created instances will be empty and bypass the central `AssetStore`, leading to inconsistent server state.
-   **Runtime State Modification:** Do not use reflection or other means to modify the fields of a GameModeType instance after it has been loaded. The server's systems assume this data is static and will not react to changes, causing desynchronization.
-   **Premature Access:** Do not call `getAssetStore()` or `fromGameMode()` from a system that initializes before the main `AssetRegistry` is populated. This will result in a `NullPointerException` or an empty asset map.

## Data Pipeline

GameModeType is a destination for data loaded from disk, not a processor in a pipeline. Its data flows from configuration files into the server's memory at startup.

> **Loading Flow:**
> `assets/hytale/gamemodes/adventure.json` -> Server Asset Loader -> **GameModeType.CODEC** -> **GameModeType Instance** -> Caching in `GameModeType.ASSET_STORE`

> **Usage Flow:**
> Player State Change -> `GameMode` Enum -> `GameModeType.fromGameMode()` -> **GameModeType Instance** -> Permission System / Interaction System<ctrl63>

