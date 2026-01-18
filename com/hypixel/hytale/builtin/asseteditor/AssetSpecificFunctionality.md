---
description: Architectural reference for AssetSpecificFunctionality
---

# AssetSpecificFunctionality

**Package:** com.hypixel.hytale.builtin.asseteditor
**Type:** Utility

## Definition
```java
// Signature
@Deprecated
public class AssetSpecificFunctionality {
```

## Architecture & Concepts
The AssetSpecificFunctionality class is a static utility responsible for implementing concrete, asset-type-specific behaviors within the Asset Editor. It functions as a centralized event handler that bridges the gap between high-level Asset Editor UI events and low-level game world state modifications.

**Warning:** This class is marked as **Deprecated**. New functionality should not be added here. It represents a legacy pattern of centralizing disparate logic. Future implementations should favor more modular, asset-type-specific handler classes.

Architecturally, this class acts as a controller in an event-driven system. It subscribes to a wide range of events published by the Asset Editor's event bus, such as asset selection, button activations, and asset reloads. Upon receiving an event, it executes logic tailored to the specific asset type involved (e.g., Item, Model, Weather). This logic often involves retrieving a player reference associated with an editor client and dispatching commands or packets to either the game world or back to the editor client's UI.

Key responsibilities include:
-   Applying a selected model to a player character in-game.
-   Equipping a selected item to a player's inventory.
-   Activating and managing live weather previews in the game world when a weather asset is being edited.
-   Providing data sets for UI autocomplete features, such as localization keys and block groups.
-   Sending updated model preview data to the editor client when an underlying asset (like a ModelAsset) is reloaded.

## Lifecycle & Ownership
-   **Creation:** As a static utility class, it is not instantiated. Its lifecycle is tied to the JVM's ClassLoader.
-   **Scope:** The class and its static state exist for the entire duration of the server's runtime. The primary initialization point is the static `setup` method, which must be called once during the `AssetEditorPlugin` bootstrap sequence to register all necessary event listeners.
-   **Destruction:** State is cleared on a per-client basis via the `onClientDisconnected` event handler. The central `activeWeatherPreviewMapping` cache is only fully cleared upon server shutdown.

## Internal State & Concurrency
-   **State:** The class maintains a mutable, shared state through the static `activeWeatherPreviewMapping` field. This `ConcurrentHashMap` tracks which players have active weather previews, linking a player's UUID to their preview settings. This state is critical for managing live previews and ensuring they are correctly torn down when a player disconnects or selects a different asset. All other static fields are final constants, making them immutable.
-   **Thread Safety:** This class is designed to be thread-safe. Event handlers can be invoked concurrently by different network threads servicing various editor clients. The use of `ConcurrentHashMap` for `activeWeatherPreviewMapping` ensures that modifications to the shared preview state are handled safely without explicit locking. Operations involving a specific player are typically delegated to the player's world thread via `world.execute`, ensuring that game state modifications are properly serialized.

## API Surface
The primary public contract is the `setup` method. All other methods are either private or package-private, intended for internal use within the asset editor plugin.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setup() | static void | O(N) | **Critical Initializer.** Registers all event listeners with the plugin's EventRegistry. Must be called once at startup. |
| resetTimeSettings(client, player) | static void | O(1) | Package-private. Resets the time override for a player, synchronizing the editor with the current world time. |
| handleWeatherOrEnvironmentSelected(...) | static void | O(1) | Package-private. Activates a weather or environment preview for a player in-game. |
| handleWeatherOrEnvironmentUnselected(...) | static void | O(1) | Package-private. Deactivates a weather or environment preview for a player. |

## Integration Patterns

### Standard Usage
This class is not meant to be called directly. Its functionality is invoked exclusively through the event system after being initialized by the main plugin. The only direct interaction is the initial setup call.

```java
// Correct initialization within the AssetEditorPlugin
public class AssetEditorPlugin extends JavaPlugin {
    public void onEnable() {
        // ... other initializations
        AssetSpecificFunctionality.setup();
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Invocation:** Do not call event handlers like `onEquipItem` or `onSelectAsset` directly. This bypasses the event system and can lead to unpredictable behavior. Always fire the corresponding event instead.
-   **Skipping Setup:** Failure to call `AssetSpecificFunctionality.setup()` during plugin initialization will result in a completely non-functional asset editor, as no specific behaviors will be registered.
-   **External State Modification:** Do not directly modify the static `activeWeatherPreviewMapping`. All interactions with this state must be managed through the class's own event handlers to ensure proper lifecycle control.

## Data Pipeline
This class sits at the intersection of several data flows, translating events into network packets for both the editor and game clients.

**Flow 1: Live Asset Reload**
> Flow:
> File System Change -> AssetStore Reload -> `LoadedAssetsEvent` -> **AssetSpecificFunctionality.onModelAssetLoaded** -> `AssetEditorUpdateModelPreview` Packet -> Editor Client UI

**Flow 2: In-Game Action from Editor**
> Flow:
> Editor Client Button Click -> `AssetEditorActivateButtonEvent` -> **AssetSpecificFunctionality.onEquipItem** -> PlayerRef Inventory Modification -> Server Game State Update -> (Implicit) Inventory Update Packet -> Game Client

**Flow 3: Live World Preview**
> Flow:
> Editor Client Asset Selection -> `AssetEditorSelectAssetEvent` -> **AssetSpecificFunctionality.handleWeatherOrEnvironmentSelected** -> `UpdateEditorWeatherOverride` Packet -> Game Client World Render

