---
description: Architectural reference for ClientFeatureRegistry
---

# ClientFeatureRegistry

**Package:** com.hypixel.hytale.server.core.registry
**Type:** Transient

## Definition
```java
// Signature
public class ClientFeatureRegistry extends Registry<ClientFeatureRegistration> {
```

## Architecture & Concepts
The ClientFeatureRegistry is a specialized, plugin-scoped registry that manages server-side declarations of features and asset tags required by the client. It serves as a high-level abstraction layer, decoupling plugins from the low-level mechanics of capability negotiation and asset synchronization.

Its primary architectural role is to act as the authoritative source for a plugin's client-side requirements. When a plugin registers a feature or tag, this class orchestrates two critical side effects:
1.  It updates the server's global state via static handlers like AssetRegistry and ClientFeatureHandler.
2.  It triggers network broadcasts to inform all connected clients of new asset requirements, ensuring clients can download necessary resources.

This registry is a key component of the server's dynamic content system, allowing plugins to extend client functionality and asset sets at runtime.

### Lifecycle & Ownership
-   **Creation:** An instance of ClientFeatureRegistry is created by the server's plugin loader for each individual plugin. The constructor signature, which requires a PluginBase context and preconditions, makes it clear that it is framework-managed and not intended for direct user instantiation.
-   **Scope:** The lifecycle of a ClientFeatureRegistry instance is tightly coupled to the lifecycle of the plugin that owns it. It exists for the duration that the plugin is enabled.
-   **Destruction:** The instance is eligible for garbage collection when the corresponding plugin is disabled or unloaded. The parent Registry class handles the teardown of internal collections.

## Internal State & Concurrency
-   **State:** This class is stateful. It inherits a collection of ClientFeatureRegistration objects from its parent. More importantly, its methods are designed to mutate shared, global server state within the AssetRegistry and ClientFeatureHandler. It does not cache data but acts as a proxy to global caches.
-   **Thread Safety:** **This class is not thread-safe.** Its methods perform write operations on static, non-synchronized collections and trigger network events through the global Universe object. All invocations on a ClientFeatureRegistry instance must be performed on the main server thread to prevent severe race conditions and state corruption.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| register(ClientFeature feature) | ClientFeatureRegistration | O(1) | Registers a client feature with the global ClientFeatureHandler. This informs the server of a capability the plugin requires the client to have. |
| registerClientTag(String tag) | void | O(N) | Registers a client-side asset tag. This operation is expensive as it may trigger a network broadcast to N connected clients, instructing them to acknowledge the new tag. |

## Integration Patterns

### Standard Usage
The intended use is for a plugin to retrieve its dedicated registry during its startup lifecycle and declare all necessary client features and tags. This should be considered a setup-time operation.

```java
// Example from within a PluginBase's onEnable() method
ClientFeatureRegistry featureRegistry = this.getClientFeatureRegistry();

// Declare that this plugin requires clients to support custom UI rendering
featureRegistry.register(ClientFeature.CUSTOM_UI_RENDERING);

// Register a new asset tag so clients can resolve related assets
featureRegistry.registerClientTag("myplugin:custom_entities_v1");
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new ClientFeatureRegistry()`. The server framework provides a correctly configured instance tied to your plugin's context. Manual creation will result in a disconnected registry that does not affect global server state.
-   **Late Registration:** Do not register features or tags after the server has started and players have joined. Doing so can create a state discrepancy for already-connected clients, potentially causing crashes or visual artifacts. All registrations must occur within the plugin's `onEnable` method.
-   **Asynchronous Invocation:** Calling `registerClientTag` from an asynchronous task or a different thread is extremely dangerous. It will bypass server thread safety mechanisms and likely cause a ConcurrentModificationException or network buffer corruption during the packet broadcast.

## Data Pipeline
The ClientFeatureRegistry acts as an entry point into two distinct server systems: the global state machine for client capabilities and the network pipeline for asset synchronization.

> **Tag Registration Flow:**
> Plugin `onEnable` -> **ClientFeatureRegistry.registerClientTag()** -> AssetRegistry (Global State Update) -> ServerTags Packet Creation -> Universe.broadcastPacket() -> Network Layer -> All Connected Clients

> **Feature Registration Flow:**
> Plugin `onEnable` -> **ClientFeatureRegistry.register()** -> ClientFeatureHandler (Global State Update)

