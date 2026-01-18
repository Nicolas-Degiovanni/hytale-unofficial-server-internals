---
description: Architectural reference for FlyCameraModule
---

# FlyCameraModule

**Package:** com.hypixel.hytale.server.core.modules.camera
**Type:** Plugin Module

## Definition
```java
// Signature
public class FlyCameraModule extends JavaPlugin {
```

## Architecture & Concepts
The FlyCameraModule is a server-side, event-driven enforcement mechanism responsible for synchronizing a player's ability to use the fly camera with their server-side permissions. It functions as a reactive security and state management component, ensuring that the fly camera state on the client is immediately revoked when the corresponding permission is removed on the server.

This module does not grant permissions or enable the fly camera. Its sole responsibility is to listen for events indicating a potential loss of the `hytale.camera.flycam` permission and dispatch a network packet to the client to disable the feature if necessary. It acts as a critical bridge between the PermissionsModule, the server's event bus, and the client-side camera controller.

Its design is entirely passive; it performs no actions until triggered by a relevant event from the HytaleServer EventBus.

### Lifecycle & Ownership
- **Creation:** Instantiated by the server's plugin loading system during the main server bootstrap sequence. The static MANIFEST field declares this class as a core plugin, ensuring it is loaded automatically.
- **Scope:** Session-scoped. The FlyCameraModule instance persists for the entire runtime of the Hytale server.
- **Destruction:** The instance is destroyed and garbage collected during the server shutdown process when all plugins are unloaded.

## Internal State & Concurrency
- **State:** The FlyCameraModule is **stateless**. It maintains no internal cache or mutable fields related to player permissions or state. All lookups are performed in real-time against authoritative sources like the PermissionsModule and the Universe.

- **Thread Safety:** This module is designed to be invoked by the server's main EventBus. It is assumed that all permission-related events are dispatched on a single, main game thread. Therefore, the module itself contains no explicit locking or concurrency controls.

    **Warning:** Direct invocation of its internal methods from multiple threads is not supported and would be unsafe, as it relies on the thread safety of the underlying PermissionsModule and Universe components.

## API Surface
The FlyCameraModule exposes no public API for direct developer interaction. Its contract is with the plugin system via its manifest and its lifecycle methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| MANIFEST | PluginManifest | O(1) | Static definition declaring the class as a core plugin and specifying its dependency on the PermissionsModule. |

## Integration Patterns

### Standard Usage
This module is not designed to be used directly. Its functionality is enabled simply by being present in the server's runtime. The server's plugin loader will automatically instantiate and register it.

Developers interact with this system implicitly by manipulating player permissions through the PermissionsModule API.

```java
// Example: Revoking a permission that triggers the FlyCameraModule
PermissionsModule perms = PermissionsModule.get();
UUID playerUuid = ...;

// This action will fire a PlayerPermissionChangeEvent,
// which the FlyCameraModule is listening for.
perms.removePermission(playerUuid, "hytale.camera.flycam");

// The module will automatically handle sending the packet to disable flycam.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new FlyCameraModule()`. The server's plugin framework is the sole owner of this class's lifecycle.
- **Manual Event Registration:** Do not attempt to register this module or a new instance of it with the EventBus. The internal `setup` method handles all necessary registrations.
- **Dependency Inversion:** Do not rely on this module for permission checks. Always query the PermissionsModule directly as the source of truth. This module is a listener, not a provider.

## Data Pipeline
The primary data flow through this component is triggered by a change in the permission system. The module translates a server-side event into a client-bound network packet.

> Flow:
> EventBus (e.g., GroupPermissionChangeEvent.Removed) -> **FlyCameraModule::handleGroupPermissionsRemoved** -> PermissionsModule::getGroupsForUser -> Universe::getPlayers -> **FlyCameraModule::checkAndEnforceFlyCameraPermission** -> PlayerRef::getPacketHandler -> `new SetFlyCameraMode(false)` -> Network Layer -> Client Camera Controller

