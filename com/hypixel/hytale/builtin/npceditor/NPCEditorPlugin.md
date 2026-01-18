---
description: Architectural reference for NPCEditorPlugin
---

# NPCEditorPlugin

**Package:** com.hypixel.hytale.builtin.npceditor
**Type:** Singleton (Framework-Managed)

## Definition
```java
// Signature
public class NPCEditorPlugin extends JavaPlugin {
```

## Architecture & Concepts
The NPCEditorPlugin serves as a specialized integration layer, or "glue," between the generic Asset Editor framework and the NPC behavioral system. Its primary function is to enable a live 3D preview of an NPC model when a developer is editing an NPCRole configuration file within the in-game Asset Editor.

Architecturally, this plugin operates as a passive listener. It does not expose any direct API for other systems to call. Instead, it subscribes to a specific UI event, `AssetEditorSelectAssetEvent`. Upon receiving this event, it interrogates the event payload to determine if the selected asset is an NPCRole.

If the asset matches, the plugin orchestrates a complex, read-only sequence across multiple core systems:
1.  It communicates with the `NPCPlugin` to locate and validate the selected NPCRole.
2.  It leverages the NPC spawning and builder systems (`Builder`, `SpawningContext`) to resolve the role configuration into a concrete `Model` asset. This is the same logic path used when spawning an NPC in the game world, ensuring the preview is accurate.
3.  It serializes the resolved `Model` into a network packet.
4.  It dispatches this packet (`AssetEditorUpdateModelPreview`) back to the specific client that initiated the event, instructing its Asset Editor UI to render the model.

This component is critical for developer experience but is completely decoupled from the runtime NPC logic that occurs during normal gameplay.

## Lifecycle & Ownership
- **Creation:** A single instance of NPCEditorPlugin is instantiated by the server's core plugin loader during the server bootstrap sequence. The framework provides a `JavaPluginInit` context required for its construction.
- **Scope:** The instance is application-scoped. It persists for the entire lifetime of the server process.
- **Destruction:** The instance is dereferenced and eligible for garbage collection only upon server shutdown when the plugin system is dismantled. There is no public API for manual unloading or destruction.

## Internal State & Concurrency
- **State:** This class is effectively stateless. It contains a single `static final` field for default camera settings but holds no mutable instance or static state. All operations are transient, processing data from incoming events and external services.
- **Thread Safety:** The core logic resides in the static `onSelectAsset` event handler. Its thread safety is **contingent** on the server's event bus architecture. It is designed to be executed on a single main server thread. Direct, concurrent invocation of its logic from multiple threads is unsupported and would lead to severe race conditions, as it calls into non-thread-safe singletons like `NPCPlugin`.

## API Surface
The public API of this class is minimal and intended for framework use only. Its primary interaction mechanism is through the server's event bus.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setup() | protected void | O(1) | Framework-invoked initialization method. Registers the NPCRole asset type and subscribes to the asset selection event. |

## Integration Patterns

### Standard Usage
This plugin is not designed for direct developer interaction. It is loaded automatically by the server's plugin framework. Its functionality is invoked implicitly by the system when a user performs the following action in the game client:

1.  Opens the Asset Editor.
2.  Navigates to and selects an asset file corresponding to an `NPCRole`.

The plugin will then automatically handle the process of displaying the NPC's model in the editor's preview pane.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new NPCEditorPlugin()`. The plugin's lifecycle is exclusively managed by the server's plugin framework, which provides necessary initialization context.
- **Direct Event Handling:** Do not attempt to call the static `onSelectAsset` method directly. It is tightly coupled to the `AssetEditorSelectAssetEvent` and expects to be invoked only by the event registry. Bypassing the event bus will break its assumptions about state and context.

## Data Pipeline
The plugin's primary responsibility is to manage a data flow that begins with a user action on the client and ends with a targeted UI update on the same client.

> Flow:
> Client UI Action (Selects NPCRole) -> `AssetEditorSelectAssetEvent` -> Server Event Bus -> **NPCEditorPlugin.onSelectAsset** -> `NPCPlugin` (Role Validation & Model Resolution) -> `AssetEditorUpdateModelPreview` Packet -> Network Handler -> Client UI Update (Renders Model)

