---
description: Architectural reference for CrouchSlidePlugin
---

# CrouchSlidePlugin

**Package:** com.hypixel.hytale.builtin.crouchslide
**Type:** Plugin Component

## Definition
```java
// Signature
public class CrouchSlidePlugin extends JavaPlugin {
```

## Architecture & Concepts
The CrouchSlidePlugin is a server-side component responsible for declaring and enabling the "Crouch Slide" gameplay feature. It serves as a declarative bridge between the server's core plugin system and the client-server feature negotiation protocol.

During the server's initialization sequence, the plugin loader discovers and instantiates this class. Its primary function is to register the CrouchSlide capability with the ClientFeatureRegistry. This registration informs the server that it can support clients requesting this specific movement mechanic. When a client connects, the server uses this registry during the protocol handshake to communicate which optional features, like CrouchSlide, are available for the session.

This class is a minimal, fire-and-forget implementation of the JavaPlugin contract, focused exclusively on feature advertisement. It contains no dynamic logic, state management, or event listeners.

## Lifecycle & Ownership
- **Creation:** Instantiated by the server's internal Plugin Loader during the server startup or plugin discovery phase. The constructor receives a JavaPluginInit context object, which provides access to core server registries.
- **Scope:** The instance persists for the entire lifecycle of the server process, or until the plugin is explicitly unloaded by an administrator.
- **Destruction:** The object is marked for garbage collection when the server shuts down or the plugin is unloaded. The base JavaPlugin class handles the de-registration and cleanup process.

## Internal State & Concurrency
- **State:** This class is stateless. It does not define or manage any mutable fields. Its purpose is fulfilled entirely within the setup lifecycle method.
- **Thread Safety:** The instance is not thread-safe. All lifecycle methods, such as setup, are invoked by the server's main thread during a controlled initialization phase. Concurrently invoking methods on this object is an unsupported and dangerous operation.

## API Surface
The public API is limited to the constructor, which is intended for framework use only. The core logic resides in the protected lifecycle method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setup() | void | O(1) | **Lifecycle Method.** Registers the CrouchSlide feature and a related client tag with the server's registries. This method is called exactly once by the plugin system after the object is constructed. |

## Integration Patterns

### Standard Usage
This class is not designed for direct developer interaction. It is automatically discovered and managed by the Hytale server's plugin system. Its presence in the server's classpath is sufficient for its functionality to be integrated.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance of this class manually using `new CrouchSlidePlugin()`. The plugin system is responsible for its creation and dependency injection. Manual instantiation will result in a non-functional object that is not registered with the server.
- **Manual Lifecycle Invocation:** Do not call the setup method directly. This is a lifecycle callback managed by the server. Calling it manually can lead to duplicate feature registration, unpredictable behavior, and potential server instability.

## Data Pipeline
This component's role is to inject a feature flag into the server's configuration, which is later used in the client connection pipeline.

> Flow:
> Server Plugin Loader -> **CrouchSlidePlugin.setup()** -> ClientFeatureRegistry -> Client Handshake Protocol -> Client Enables Feature

