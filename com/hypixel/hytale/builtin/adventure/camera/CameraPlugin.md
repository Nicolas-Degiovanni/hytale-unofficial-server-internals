---
description: Architectural reference for CameraPlugin
---

# CameraPlugin

**Package:** com.hypixel.hytale.builtin.adventure.camera
**Type:** Plugin Module

## Definition
```java
// Signature
public class CameraPlugin extends JavaPlugin {
```

## Architecture & Concepts
The CameraPlugin class is a bootstrap and registration module, not a runtime service. Its sole responsibility is to integrate camera-related features into the core server engine during the plugin loading phase. It acts as a central manifest, declaring all assets, data codecs, commands, and systems required for camera effects like screen shake and view bobbing.

This class does not implement any game logic itself. Instead, it populates the server's various registries with the necessary components:

*   **Asset Stores:** It configures and registers asset loaders for `CameraShake` and `ViewBobbing` definitions, specifying how they are loaded from disk, indexed, and synchronized with clients.
*   **Codecs:** It registers serialization and deserialization logic for `CameraShakeEffect` and `CameraShakeInteraction`. This is critical for network transport and saving game state.
*   **Commands:** It registers the `CameraEffectCommand`, exposing camera functionality to server administrators and script engines.
*   **ECS System:** It registers the `CameraEffectSystem`, which is the runtime component responsible for processing camera effects on entities each game tick.

By centralizing these registrations, the CameraPlugin ensures that all camera-related functionality is initialized correctly and in the proper order within the server's lifecycle.

## Lifecycle & Ownership
- **Creation:** Instantiated by the server's internal PluginLoader during the server startup sequence. A `JavaPluginInit` context, which provides access to core server registries, is injected via the constructor.
- **Scope:** The CameraPlugin object itself is ephemeral. Its existence is primarily for the duration of the `setup` method call. The components it registers (systems, asset stores, commands) are transferred to and managed by the core server registries, persisting for the entire server session.
- **Destruction:** The CameraPlugin instance is eligible for garbage collection immediately after its `setup` method completes. The registered components are unloaded only when the server shuts down or if the plugin is explicitly reloaded or disabled by an administrator.

## Internal State & Concurrency
- **State:** This class is effectively stateless. It does not maintain any internal state across method calls. All configurations are passed directly into the server registries, which are responsible for managing that state.
- **Thread Safety:** The `setup` method is invoked by the PluginLoader on the server's main thread during a single-threaded initialization phase. As such, the class is not designed for concurrent access and requires no internal synchronization. The systems it registers, such as `CameraEffectSystem`, must manage their own thread safety within the context of the server's multithreaded Entity Component System.

## API Surface
The public API of this class is intended for framework invocation only, not for general developer use.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setup() | protected void | O(1) | **Framework Override.** Registers all camera components with the server. Invoked once by the plugin system during startup. |

## Integration Patterns

### Standard Usage
Developers do not interact with the CameraPlugin class directly. It is loaded automatically by the server. Instead, developers use the features that this plugin registers. For example, triggering a camera effect via the registered command.

```java
// This code would be executed by the server's command handler,
// not by directly interacting with CameraPlugin.

// Example: A server administrator runs a command in the console
// > camera effect @a hytale:camera_shake_light

// The CameraEffectCommand (registered by this plugin) handles this,
// which in turn might create a component that the CameraEffectSystem
// (also registered by this plugin) will process.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new CameraPlugin()`. The constructor requires a `JavaPluginInit` context that is only supplied by the server's PluginLoader. Attempting to create it manually will fail.
- **Manual Invocation:** Do not call the `setup` method directly. This would cause duplicate registrations in the server's core registries, leading to critical server errors, asset conflicts, and unpredictable behavior. The framework guarantees it is called exactly once at the correct time.

## Data Pipeline
The CameraPlugin establishes the necessary components for the following data flow. It does not participate in the flow itself but is responsible for building the pipeline.

> Flow:
> Game File (`assets/.../Camera/CameraShake/*.json`) -> **AssetRegistry** (Configured by CameraPlugin) -> `CameraEffectCommand` or `CameraShakeInteraction` -> **CameraEffectSystem** (Registered by CameraPlugin) -> Network Packet -> Client-side Camera Update

