---
description: Architectural reference for SafetyRollPlugin
---

# SafetyRollPlugin

**Package:** com.hypixel.hytale.builtin.safetyroll
**Type:** Plugin Component

## Definition
```java
// Signature
public class SafetyRollPlugin extends JavaPlugin {
```

## Architecture & Concepts
The SafetyRollPlugin is a declarative, built-in server plugin whose sole responsibility is to register the client-side **Safety Roll** feature with the server's core systems. It acts as a manifest entry, enabling the server to recognize and negotiate this specific movement capability with connecting clients.

This component is not an active service that processes data or manages game state. Instead, it contributes configuration to a central registry during the server's initial bootstrap phase. By registering the `ClientFeature.SafetyRoll` enum and an associated `Allows=Movement` tag, it allows other systems, particularly the network protocol and anti-cheat layers, to query for the existence and properties of this feature.

Its presence is critical for enabling the corresponding client-side prediction and animation for the safety roll maneuver. Without this plugin, the server would be unaware of the feature, likely rejecting any client packets related to it.

### Lifecycle & Ownership
- **Creation:** Instantiated by the server's `PluginLoader` during the server startup sequence. The framework injects a `JavaPluginInit` context object, which provides access to core registries.
- **Scope:** The plugin instance is a singleton within the context of the plugin system. It persists for the entire lifetime of the server process.
- **Destruction:** The object is marked for garbage collection when the server shuts down and the `PluginLoader` releases its references. There is no explicit destruction logic within this class.

## Internal State & Concurrency
- **State:** This class is entirely stateless. It does not contain any mutable fields and its behavior does not change after its initial setup phase. The state it modifies is external, residing within the `ClientFeatureRegistry`.
- **Thread Safety:** The `setup` method is invoked by the plugin framework on the server's main thread during a single-threaded initialization phase. Therefore, this class is inherently thread-safe as it has no concurrent access patterns. Any concerns about thread safety apply to the `ClientFeatureRegistry` it interacts with, not the plugin itself.

## API Surface
The public API is inherited from `JavaPlugin`. The only implemented method of significance is the `setup` lifecycle callback.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setup() | protected void | O(1) | Lifecycle callback invoked by the plugin system. Registers the SafetyRoll feature and its metadata. This method is not intended for external invocation. |

## Integration Patterns

### Standard Usage
This plugin is not designed to be used directly. It is automatically discovered and loaded by the server. Other systems can then query the registry to check for the feature this plugin enables.

```java
// Example: A different system checking for the feature
ClientFeatureRegistry registry = serverContext.getClientFeatureRegistry();

if (registry.isRegistered(ClientFeature.SafetyRoll)) {
    // Enable server-side logic that depends on the Safety Roll feature
    log.info("SafetyRoll feature is enabled and available for negotiation.");
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new SafetyRollPlugin()`. The plugin requires a context object (`JavaPluginInit`) that is only supplied by the server's plugin loading framework. Manual instantiation will fail.
- **Manual Invocation:** Never call the `setup` method directly. Doing so bypasses the server's managed lifecycle and will likely cause duplicate registration errors or system instability.

## Data Pipeline
This component is not part of a real-time data processing pipeline. Rather, it participates in the server's configuration and initialization flow.

> Flow:
> Server Bootstrap → Plugin Loader Discovers Jar → **SafetyRollPlugin.setup()** → ClientFeatureRegistry Updated → Connection Handshake Logic → Client Feature Negotiation

---

